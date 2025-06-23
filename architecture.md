# 🧠 Serverless LLM Inference Architecture 

## 1. Overview

This architecture enables serverless, OpenAI-compatible LLM inference with fast cold-starts and horizontal scalability.

### Core Technologies

| Component            | Language      | Purpose                                        |
| -------------------- | ------------- | ---------------------------------------------- |
| Orchestrator         | Go            | API gateway, routing, scheduling               |
| Agent Daemon         | Zig (Zap)     | Tiered model loading, GPU staging, vLLM runner |
| Checkpoint Converter | Python (HTTP) | HF to optimized `.bin`/`.idx` model conversion |
| Inference Backend    | Python (vLLM) | Autoregressive token generation                |
| Communication        | HTTP/1.1 or 2 | REST & streaming                               |

---

## 2. Component Breakdown

### 🧭 Orchestrator (Go)

#### Responsibilities:

* Exposes OpenAI-compatible endpoints:

  * `/v1/chat/completions`
  * `/v1/models`
* Maintains in-memory metadata:

  * `{agent_id, model_id, cache_level, kv_hit_rate, queue_len}`
* On request:

  1. Check if optimized model exists in checkpoint store.

  2. If not: call Python HTTP API to convert HF checkpoint → `.bin`/`.idx`.

  3. Assign chunk download tasks to idle agents (`POST /download_chunk`).

  4. Score agents using:

     ```
     score = α·cache_level
           – β·kv_hit_rate
           + γ·queue_len
           + δ·(model_size / bandwidth_tier)
     ```

  5. If needed: `POST /load_model` to selected agent.

  6. Forward request to `/infer` on the selected agent and stream response.

#### Metrics:

* Exposes `/metrics` (Prometheus)

  * `inference_latency_seconds`, `cache_hit_ratio`, `agent_loads`, `queue_lengths`

---

### ⚡ Agent Daemon (Zig + Zap)

#### Responsibilities:

* Serves HTTP endpoints:

  * `POST /heartbeat`: Report cache state and metrics
  * `POST /download_chunk`: Accept and write chunk to NVMe
  * `POST /load_model`: Load model from NVMe/DRAM to GPU
  * `POST /infer`: Forward request to vLLM and stream tokens
* Tiered Model Loader:

  1. DRAM cache → NVMe → orchestrator download
  2. Read with `mmap` or async I/O (`std.io`, `zap.fs`)
  3. Allocate pinned memory → `cudaMemcpyAsync` to GPU
* Launches `vLLM` subprocess:

  * Forwards input via local HTTP or UNIX domain socket
  * Streams output to orchestrator

#### Technology Stack:

* `zap` HTTP server
* `std.json`, `std.fs`, `std.net` for I/O and parsing
* `zig-cuda`, `libc` bindings for GPU memory ops
* `zap.fs`, `mmap` for high-throughput file reads

---

### 🐍 Checkpoint Converter (Python + HTTP API)

* Converts raw HuggingFace model to `.bin`/`.idx` format
* Exposes REST API:

  * `POST /convert`
  * Accepts model name, output path
* Orchestrator invokes this service over HTTP
* Stores outputs in object storage (S3, Azure Blob, or local)

#### Example HTTP Call

```http
POST /convert
Content-Type: application/json

{
  "model_id": "mistralai/Mistral-7B-Instruct",
  "output_dir": "/mnt/store/mistral/"
}
```
