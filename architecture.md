# ServerlessLLM MVP Architecture

An OpenAI-compatible, low-latency serverless LLM inference system built for extreme performance.

---

## 1. Simplified Component Diagram

```mermaid
flowchart TB
  subgraph Client
    C[Client<br/>(OpenAI API)]
  end

  subgraph Orchestrator
    API[API Gateway]
    SCH[Scheduler]
    ING[Model Ingestion]
    DL[Fan-out Downloader]
  end

  subgraph KV["Metadata Store<br/>(etcd/Redis)"]
    KVData[(model→agent metadata)]
  end

  subgraph Store["Checkpoint Store<br/>(S3/Blob/HF)"]
    IDX[(.idx files)]
    BIN[(.bin files)]
  end

  subgraph Agent["Agent Daemon<br/>(Rust + Zig)"]
    HB[Heartbeat]
    DQ[Download Queue]
    LT[Loader (Zig)]
    IS[Inference μService]
  end

  C -->|REST/gRPC| API
  API --> SCH
  SCH --> KVData
  SCH --> Store
  SCH --> DL
  ING --> Store
  DL --> DQ
  API --> IS
  IS --> API
  DQ --> LT
  LT --> IS
  HB --> KVData
  IS --> KVData
  LT --> Store
```

---

## 2. Component Responsibilities & Tech Stack

| Component               | Responsibility                                                           | Tech Stack               |
| ----------------------- | ------------------------------------------------------------------------ | ------------------------ |
| **Client**              | Sends OpenAI-spec inference requests                                     | Any (Python/JS/etc.)     |
| **API Gateway**         | Receives `/v1/...` calls, handles authentication                         | Rust (Tokio + warp)      |
| **Scheduler**           | Picks best Agent by cache_level, kv_hit_rate, queue_len, startup_cost    | Rust                     |
| **Model Ingestion**     | On-demand HF raw checkpoint → Convert → Upload optimized (`.idx`+`.bin`) | Python + Rust lib        |
| **Fan-out Downloader**  | Parallel HTTP-Range download into Agents’ NVMe                           | Rust                     |
| **Metadata Store (KV)** | Stores `model_id → [{agent_id, cache_level, kv_hit_rate, queue_len}]`    | etcd (Go) or Redis       |
| **Checkpoint Store**    | Holds optimized artifacts (`.idx`, `.bin`)                               | S3 / Azure Blob / HF Hub |
| **Agent Daemon**        |

• Heartbeat to KV  
 • Download Queue worker  
 • Multi-Tier Loader (Zig)  
 • Inference μService (Rust) | Rust + Zig |
| **Multi-Tier Loader** | DRAM / NVMe / S3 → pinned buffer → CUDA DMA → GPU memory | Zig (direct syscalls) |
| **Inference μService** | Exposes OpenAI endpoints, calls backend adapters (vLLM/Triton/others) | Rust + FFI (C++/Py) |
| **Monitoring** | Prometheus metrics (load times, kv hits, queue lengths) | Prometheus / Grafana |
| **CI/CD Pipeline** | Trains/fine-tunes → Checkpoint conversion → Upload optimized artifacts | GitHub Actions / Jenkins |

---

## 3. Request Flow

1. **Client** → API Gateway:  
   `POST /v1/chat/completions` (OpenAI spec)

2. **API Gateway** → Scheduler:  
   • Check Metadata Store for agents with `model_id`  
   • If no optimized files in Checkpoint Store, invoke Model Ingestion  
   • Compute `score(agent)` = α·cache_level – β·kv_hit + γ·queue_len + δ·(size/bw)  
   • Select Agent A

3. **Scheduler** → Fan-out Downloader:  
   • If no agent has model in DRAM/NVMe, download `.bin` chunks in parallel into N agents

4. **Scheduler** → Agent A:  
   `POST /load {model_id}`

5. **Agent A**  
   a. **Loader** (Zig): multi-tier read → GPU  
   b. **Inference μService** (Rust): serves chat completion, updates kv_hit_rate

6. **Agent A** → API Gateway:  
   Streams tokens back (SSE/gRPC)

7. **API Gateway** → Client:  
   Proxies token stream as OpenAI spec

---

## 4. Next Steps & Extension Points

- **Multi-Backend Adapters**: easily add new inference engines
- **Live Migration**: zero-downtime in-flight inference moves
- **Layered Shard Streaming**: overlap load + infer
- **P2P / RDMA**: peer fetch of chunks
- **Quantization**: compressed weights + on-GPU decompress
- **Autoscaling**: popularity predictor + agent pool scaling
- **QoS**: tenant isolation, rate limiting
- **Observability**: eBPF telemetry, Grafana dashboards
