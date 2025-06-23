# ServerlessLLM MVP Architecture

A low-latency, OpenAI-compatible serverless LLM inference system using only Go and Zig for core components, and vLLM for model execution.

---

## 1. Components & Tech Stack

1. **Orchestrator**  
   • Language: Go  
   • Responsibilities:

   - Expose OpenAI-spec endpoints (`/v1/chat/completions`, `/v1/models/*`) via HTTP/gRPC
   - Maintain in-memory metadata: which agents cache which models, their cache level, KV-cache hit rates, queue lengths
   - On request:
     1. Ensure optimized checkpoint exists (download raw HuggingFace checkpoint and convert it in-process)
     2. Score agents by
        ```
        score = α·cache_level
              – β·kv_hit_rate
              + γ·queue_len
              + δ·(model_size / bandwidth_tier)
        ```
     3. If no agent has model in DRAM/NVMe, fan-out HTTP-Range downloads of `.bin` chunks to multiple agents
     4. Send `LoadModel(model_id)` RPC to chosen agent if needed
     5. Forward the client’s `/v1/chat/completions` call to the agent; stream tokens back
   - Metrics: expose Prometheus metrics for request latencies, cache hits, scheduling decisions

2. **Agent Daemon**  
   • Languages: Zig
   • Responsibilities:

   - Heartbeat over gRPC to orchestrator with `{agent_id, model_id, cache_level, kv_hit_rate, queue_len}`
   - Download-queue worker: accept chunk-download tasks via gRPC from orchestrator; write to NVMe
   - Multi-Tier Loader (Zig):
     1. Read from DRAM cache if present
     2. Else read from NVMe SSD via `io_uring`/O_DIRECT
     3. Else HTTP-Range from orchestrator/Checkpoint Store
     4. Copy into pinned host buffer → `cudaMemcpyAsync` → GPU memory
   - Inference launcher: spawn vLLM process (Python) with IPC or gRPC, point it at the loaded model
   - Expose a minimal gRPC endpoint (`Infer(request) → token stream`) consumed by orchestrator

3. **Checkpoint Store**  
   • External object storage (S3, Azure Blob, or HuggingFace Hub)  
   • Holds optimized artifacts per model:

   - `model_id.idx` (tensor index)
   - `model_id.bin` (GPU-partitioned chunks)

4. **Inference Backend**  
   • vLLM (Python/C++), running as a child process or sidecar of the Agent  
   • Responsible for autoregressive token generation using GPU memory populated by the Zig loader

5. **Client**  
   • Any HTTP/gRPC client speaking the OpenAI API spec (e.g., Python, JavaScript, Go)

---

## 2. Data & Control Flow

1. **Client** → **Orchestrator**: `POST /v1/chat/completions` with `{model_id, prompt…}`
2. **Orchestrator**:
   - Check local metadata for agents with `model_id` cached in DRAM/NVMe
   - If optimized files not in Checkpoint Store, download raw HF checkpoint and convert to `.idx`/`.bin` in-process
   - If no agent has a hot cache, issue parallel HTTP-Range download tasks to agents’ download queues
   - Compute `score(agent)` and select the best agent
   - If agent has not loaded the model, send `LoadModel(model_id)` RPC
   - Call `Infer(request)` RPC on agent; stream tokens back to client
3. **Agent**:
   - Heartbeat metadata to orchestrator
   - Process download tasks: write chunks to NVMe
   - On `LoadModel`: Zig loader stages chunks → GPU
   - On `Infer(request)`: forward to vLLM, collect tokens, update kv_hit_rate, stream back over gRPC
4. **vLLM**: uses GPU memory to generate tokens

---

## 3. Why Go & Zig?

- **Go** provides a single binary for both orchestrator and agent control logic, easy concurrency, and fast development.
- **Zig** enables zero-overhead, no-GC data-plane for multi-tier checkpoint loading with direct syscalls and CUDA FFI.
- **vLLM** remains the high-performance inference engine without reimplementation.

---

## 4. Scalability & Extensibility

- **Horizontal scaling**: run multiple orchestrators behind a load-balancer; agents auto-register via heartbeat.
- **Pluggable inference**: while MVP uses vLLM, new backends can be integrated by extending the Agent’s gRPC adapter.
- **Future roadmap**: live migration, layered loading, P2P chunk sharing, quantization—each can be added without splitting services.
