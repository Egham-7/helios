# 🧠 Serverless LLM Inference Architecture

## 1. Overview

This architecture provides a robust, scalable, and high-performance solution for serverless Large Language Model (LLM) inference. It focuses on achieving fast cold-starts and dynamic model loading by leveraging Zig for low-level control. The system maintains an OpenAI-compatible API surface and supports deployment across multiple cloud providers.

### Core Technologies

| Component           | Language | Purpose                                                                    |
| :------------------ | :------- | :------------------------------------------------------------------------- |
| Orchestrator        | Go       | API gateway, intelligent routing, scheduling, agent management             |
| Agent Daemon        | Zig      | All-in-one component: model download, staging, GPU loading, vLLM runner    |
| Inference Backend   | Python   | Autoregressive token generation (via vLLM subprocess)                      |
| Model Cache Storage | Azure Blob | Centralized, high-throughput storage for HuggingFace model checkpoints    |
| Communication       | HTTP/1.1 / HTTP/2 | REST & streaming for inference, HTTP for control plane communication      |

---

## 2. Component Breakdown

### 🧭 Orchestrator (Go)

The Orchestrator functions as the intelligent front-end and central control plane of the entire system. It exposes the public API, manages the real-time state of all available inference agents, and makes intelligent routing and model loading decisions.

#### Responsibilities:

*   **API Gateway:**
    *   Exposes OpenAI-compatible REST API endpoints:
        *   `/v1/chat/completions`: Handles LLM inference requests, streaming responses.
        *   `/v1/models`: Provides information about available models.
        *   `/metrics`: Exposes Prometheus-compatible metrics for monitoring.
*   **Agent Registry & State Management:**
    *   Maintains an in-memory (or optionally, a persistent, replicated) registry of all active Agent Daemons.
    *   Stores real-time metadata for each agent, including:
        *   `agent_id`: Unique identifier for the agent.
        *   `state`: Operational status (e.g., `IDLE`, `LOADING_MODEL`, `INFERRING`, `DEGRADED`).
        *   `metrics`: Reported performance metrics (GPU utilization, memory free, vLLM queue length, KV cache hit rate).
        *   `loaded_models`: List of model IDs currently loaded on the agent, with their specific `cache_level` (e.g., `GPU_RESIDENT`, `DRAM_CACHED`, `NVME_CACHED`) and the `inference_endpoint_url` for that specific model on the agent.
*   **Intelligent Routing & Scheduling:**
    *   **On Request (`/v1/chat/completions`):**
        1.  **Model Availability Check:** Queries its in-memory metadata to determine if the requested `model_id` is currently `READY` on any Agent Daemon.
        2.  **Agent Selection & Loading (Cold Start Path):**
            *   If the model is not `READY` on any agent, or all agents with the model are overloaded:
                *   Selects the most suitable *idle* agent based on current load, available resources, and potentially proximity to cached model data.
                *   Initiates the model loading process by calling the selected Agent Daemon's **HTTP `POST /load_model`** method.
                *   Monitors the Agent's state via heartbeats (HTTP `POST /heartbeat`), waiting until the model transitions to `READY` status. The `inference_endpoint_url` will be obtained from the `POST /load_model` response or a subsequent `POST /heartbeat` update.
        3.  **Inference Forwarding (Warm Path):**
            *   Once a suitable agent with the `READY` model is identified (either already loaded or newly loaded):
                *   Scores available agents using a dynamic scoring function (details of scoring omitted for brevity).
                *   Retrieves the specific `inference_endpoint_url` for that model on the chosen Agent from its registry.
                *   Forwards the incoming inference request (HTTP POST) directly to the retrieved `inference_endpoint_url` on the selected Agent Daemon.
                *   Streams the token responses from the Agent back to the client.
    *   **Model Unloading Decisions:**
        *   Implements a model eviction policy (e.g., Least Recently Used - LRU, or time-to-live - TTL).
        *   Periodically, or when memory pressure is detected (via agent heartbeats or failed load attempts), identifies models to offload.
        *   Calls the selected Agent Daemon's **HTTP `POST /unload_model`** method to free up GPU resources.

#### Metrics:

*   Exposes Prometheus metrics on `/metrics` for system observability.

---

### ⚡ Agent Daemon (Zig)

The Agent Daemon is the core, high-performance execution unit running on each GPU-equipped host. It is implemented entirely in Zig to provide unparalleled control over resources, memory management, and I/O. It orchestrates all local operations, including model downloading, GPU loading, managing the vLLM subprocess, and handling all inter-component communication.

#### Technology Stack:

*   **Zig Language:** For core application logic, HTTP server/client, subprocess management, and low-level I/O.
*   [`zap`](https://github.com/zigzap/zap): Used for robust and performant HTTP server/client implementations within Zig.
*   [`std.json`](https://ziglang.org/documentation/master/std/#A;std:json), [`std.fs`](https://ziglang.org/documentation/master/std/#A;std:fs), [`std.net`](https://ziglang.org/documentation/master/std/#A;std:net): Zig's standard library for JSON parsing, file system operations, and network communication.
*   [`zig-cuda`](https://github.com/Nyanlis/zig-cuda) (or similar bindings): For direct interaction with CUDA APIs, enabling optimal GPU memory management and data transfer.
*   `mmap`: Linux system call for memory-mapping files, used for high-throughput reads.

#### Responsibilities:

*   **HTTP Control Plane Endpoints:**
    *   `POST /load_model`:
        *   Receives `model_id` (e.g., in JSON body) from the Orchestrator.
        *   **Updates internal state to `DOWNLOADING`**.
        *   **High-Throughput Model Download:** Performs highly parallelized HTTP GET requests using byte range headers to download HuggingFace checkpoints from Azure Blob Storage directly to a designated local NVMe cache directory.
        *   **Updates internal state to `LOADING_TO_GPU`**: Once the download is complete.
        *   **Orchestrates vLLM Subprocess:**
            *   Manages the lifecycle of the `vLLM` Python subprocess.
            *   Passes the local NVMe model path to the `vLLM` subprocess, instructing it to load the model onto the GPU.
            *   Monitors vLLM's loading status (e.g., by polling a local vLLM endpoint).
            *   **Obtains the `vLLM` inference endpoint URL** (e.g., `http://localhost:8000/v1/chat/completions`) from the `vLLM` subprocess upon successful startup.
        *   **Updates internal state to `READY`**: Once vLLM reports successful model loading.
        *   Returns an HTTP success response to the Orchestrator, **including the `inference_endpoint_url` for the newly loaded model.**
            ```json
            {
              "status": "success",
              "model_id": "your_model_id",
              "inference_endpoint_url": "http://localhost:8000/v1/chat/completions",
              "cache_state": "READY"
            }
            ```
    *   `POST /unload_model`:
        *   Receives `model_id` from the Orchestrator.
        *   Instructs the `vLLM` subprocess to unload the specified model (if vLLM supports dynamic unloading) or restarts/reconfigures the vLLM process.
        *   Clears the model from the local NVMe cache (optional, based on disk space and desired model persistence).
        *   Updates its internal state to reflect the model as `UNLOADED`.
        *   Returns an HTTP success response.
    *   `POST /heartbeat`:
        *   Receives periodic heartbeat requests from the Orchestrator (Orchestrator initiates this as a client).
        *   Returns its current operational status and metrics in the response body (e.g., JSON): `agent_id`, `gpu_utilization`, `gpu_memory_free` (obtained via `zig-cuda` / `libc`), `vllm_queue_length` (obtained from vLLM's local API), and a detailed list of currently known `model_id`s along with their `cache_state`, `kv_cache_hit_rate`s, and `inference_endpoint_url`s.
*   **HTTP Inference Endpoint (`POST /infer` - Optional/Internal Proxy):**
    *   The Agent could *optionally* expose its own `/infer` endpoint that proxies to the `vLLM`'s specific endpoint. This allows for internal processing or metrics collection before forwarding to `vLLM`.
    *   Receives requests forwarded from the Orchestrator to the specific `inference_endpoint_url` (which might be `vLLM`'s direct endpoint).
    *   Forwards the request to the local `vLLM` subprocess (e.g., via a local HTTP or UNIX domain socket).
    *   Streams the generated tokens back from `vLLM` to the Orchestrator.
*   **Local Model Cache Management:**
    *   Manages a dedicated directory on local NVMe SSDs for storing downloaded HuggingFace model checkpoints.
    *   Implements basic cache eviction (e.g., LRU) if disk space becomes a constraint, coordinating with vLLM's unloading.
*   **Subprocess Management:**
    *   Handles the robust launching, monitoring, and graceful termination of the `vLLM` Python subprocess.

---

### 🐍 Inference Backend (Python - vLLM)

The `vLLM` library is launched as a subprocess by the Zig Agent Daemon and is dedicated to the core task of high-throughput LLM token generation.

#### Technology Stack:

*   [`vLLM`](https://vllm.ai/) library.
*   CUDA Toolkit & NVIDIA GPU Drivers.

#### Responsibilities:

*   **Model Loading:** Loads HuggingFace format checkpoints from the local NVMe cache onto the GPU as instructed by the Zig Agent.
*   **Autoregressive Token Generation:** Performs highly optimized inference, including:
    *   PagedAttention for efficient KV cache management.
    *   Continuous batching of incoming requests.
    *   Speculative decoding (if enabled).
    *   Optimized CUDA kernels for LLM operations.
*   **Local HTTP/UNIX Socket Interface:** Configured to expose its own HTTP server or UNIX domain socket for the Zig Agent Daemon to forward inference requests. This is the `inference_endpoint_url` reported to the Orchestrator.
*   **Metrics & State:** Exposes internal metrics (e.g., queue length, GPU memory usage, token generation rate) that the Zig Agent Daemon can query to include in its heartbeats.

---

### ☁️ Model Cache Storage (Azure Blob Storage)

This serves as the central, highly available, and durable repository for all LLM checkpoints.

#### Technology Stack:

*   Azure Blob Storage (or equivalent cloud object storage like AWS S3, GCP Cloud Storage).

#### Responsibilities:

*   **Durable Storage:** Persistently stores all LLM model checkpoints in standard HuggingFace format.
*   **High Availability & Scalability:** Provides robust access with high throughput, essential for parallel model downloads by Agents.
*   **Global Distribution (Optional):** Can be configured for geo-replication or regional distribution to reduce latency for Agents in different geographic regions.
*   **Cost-Effectiveness:** Tiered storage options (hot, cool, archive) can be used to manage costs for less frequently accessed models.

---

## 3. Communication Flows

1.  **Client Request:** Client sends OpenAI-compatible REST request to Orchestrator's API Gateway.
2.  **Orchestrator Decision:** Orchestrator checks Agent registry.
3.  **Model Loading (Cold Path):**
    *   If model needs loading on an Agent, Orchestrator sends **HTTP `POST /load_model`** call to Agent Daemon (Zig).
    *   Zig Agent Daemon performs parallel model download from Azure Blob Storage to local NVMe.
    *   Zig Agent Daemon then launches/reconfigures the vLLM Python subprocess with the local model path, obtaining `inference_endpoint_url` upon vLLM readiness.
    *   Zig Agent Daemon returns success response **including `inference_endpoint_url`** to Orchestrator.
    *   Zig Agent Daemon sends `READY` status in subsequent **HTTP `POST /heartbeat`** to Orchestrator, confirming readiness and endpoint.
4.  **Inference (Warm Path):**
    *   Orchestrator retrieves the `inference_endpoint_url` for the chosen Agent/Model from its registry.
    *   Orchestrator forwards client's inference request (HTTP POST) directly to the retrieved `inference_endpoint_url` on the Agent Daemon.
    *   vLLM performs inference and streams tokens back via the Zig Agent.
    *   Zig Agent streams tokens back to Orchestrator, which streams to client.
5.  **Agent Monitoring:** Orchestrator periodically sends **HTTP `POST /heartbeat`** requests to Agent Daemon (Zig), which returns state and metrics in the response.
6.  **Model Unloading:** Orchestrator sends **HTTP `POST /unload_model`** call to Agent Daemon (Zig) to free GPU resources.

