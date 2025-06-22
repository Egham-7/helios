**architecture.md**

# ServerlessLLM MVP Architecture

A minimal‐viable, OpenAI‐compatible serverless LLM inference system.

## 1. API

All clients speak the OpenAI API spec.  
Example endpoint:

• POST /v1/chat/completions  
 – Request/response match OpenAI’s JSON schema  
 – Supports streaming via `text/event-stream`

## 2. Components

1. **Client**  
   • Calls `/v1/chat/completions` on the Orchestrator

2. **Orchestrator / API Gateway**  
   • Exposes OpenAI endpoints (`/v1/models/{model}/completions`, etc.)  
   • Metadata store (Redis/etcd):  
    &nbsp;&nbsp;`model_id → [ { agent_id, cache_level, kv_hit_rate } ]`  
   • Scheduler: picks the “best” agent by minimizing

   ```
   score(agent) = α·cache_level
                  – β·kv_hit_rate
                  + γ·queue_wait
                  + δ·(model_size/bandwidth)
   ```

   – cache_level: 0=DRAM, 1=NVMe, 2=remote  
    – kv_hit_rate: recent per‐token KV‐cache hit ratio  
   • Fan-out downloader: on cache‐miss, parallel HTTP-Range downloads into NVMe on N agents  
   • Proxies OpenAI spec requests/responses to/from the selected Agent

3. **Agent (per GPU-host)**  
   • **Heartbeat**: every 1s reports to Orchestrator:  
    – `cache_level` for each loaded model  
    – recent `kv_hit_rate` (e.g., fraction of tokens served from GPU cache)  
    – current load / queue length  
   • **Download queue**: accepts chunk tasks via Redis/etcd  
   • **Multi-Tier Model Loader** (Zig or prototype):

   1. Try DRAM cache
   2. Else read NVMe SSD via `io_uring`/direct‐I/O
   3. Else HTTP-Range from S3/Blob  
       → pinned host buffer → CUDA DMA into GPU  
      • **Inference µService** (vLLM/Triton):  
       – Exposes OpenAI endpoints  
       – Streams tokens back, updates `kv_hit_rate`

4. **Checkpoint Store**  
   • Object storage (S3/Blob/HuggingFace Hub)  
   • Holds optimized checkpoints:  
    – `.idx` (tensor index: name, GPU_id, offset, size)  
    – `.bin` (GPU-partitioned, chunked binaries)

## 3. Request Flow

```text
Client
  │ POST /v1/chat/completions
  ▼
Orchestrator
  1. Lookup `model_id` in Redis/etcd → [agents…]
     each agent record: {cache_level, kv_hit_rate, queue_len}
  2. If no agent has model:
       • fan-out downloads → NVMe on N agents
       • update Redis/etcd
  3. For each agent, compute score(agent)
  4. Select agent A with lowest score
  5. If agent A needs load:
       • POST /load model to Agent A
  6. Proxy POST /v1/chat/completions → Agent A
  ▼
Agent A
  • Multi-Tier Loader → GPU memory
  • vLLM/Triton serves chat completion
     – updates `kv_hit_rate`
  • Streams tokens back to Orchestrator
  ▲
Orchestrator proxies stream
  ▲
Client receives streamed tokens
```

## 4. Key Properties

- **OpenAI-Spec Compliance**: drop-in replacement for `/v1/chat/completions`.
- **Low Cold-Start**: optimized checkpoint + multi-tier loader.
- **Locality- & Cache-Aware**: schedule by cache level, KV-cache hit rates, queue depth.
- **Scalable**: agents auto-register via heartbeat; orchestrator is stateless.
- **Extensible**: roadmap features can be added modularly.
