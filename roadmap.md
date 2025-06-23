# ServerlessLLM Development Roadmap

A phased plan to take ServerlessLLM from MVP to a full-featured, high-performance serverless LLM platform.

---

## Phase 0 (Weeks 1–4): MVP Completion

Deliver a working end-to-end prototype with OpenAI-spec API.

* Checkpoint Conversion Service
  ✓ HF/PyTorch → `.idx` + `.bin`
  ✓ Python HTTP service

* Model Ingestion Service
  ✓ On-demand raw HF download + conversion
  ✓ Python container exposed over HTTP

* Metadata KV Store
  ✓ etcd/Redis schema + agent heartbeat

* Orchestrator (Go)
  ✓ OpenAI API endpoints
  ✓ Scheduling by cache\_level + kv\_hit + queue\_len + size/bw
  ✓ Fan-out downloader trigger
  ✓ Proxy HTTP chunked streams

* Agent Daemon
  ✓ Control-plane (Zig + Zap): heartbeat, queue worker, `/load`, `/infer`
  ✓ Data-plane (Zig): multi-tier loader (DRAM/NVMe/S3 → CUDA DMA)
  ✓ Inference µService (Python + vLLM)

* CI/CD + Deployment
  ✓ GitHub Actions to convert & push optimized artifacts
  ✓ Helm/Terraform for Orchestrator, Agent, etcd/Redis

* Metrics & Dashboards
  ✓ Prometheus export (load times, kv\_hit\_rate, queue\_len)
  ✓ Grafana basic panels

**Milestone:** Serve first-token in <2s for 7B models; P50 cold-start <3s.

---

## Phase 1 (Weeks 5–8): Scheduling & Downloader Optimization

* Fan-out Downloader
  ✓ Parallel HTTP-Range chunking in Zig
  ✓ Benchmark NVMe ingest speed, tune chunk size

* Enhanced Scheduler
  ✓ Real agent-observed bandwidth metrics
  ✓ Runtime CLI for weight tuning

* KV-cache Hit Awareness
  ✓ Agent tracks per-token GPU KV-cache hits
  ✓ Scheduler biases cached agents

* Retry Logic & Timeouts
  ✓ Orchestrator retry flow for agent failure

* Integration Tests
  ✓ Simulate bursty workloads; verify P95 cold-start <5s

**Milestone:** P95 cold-start <2s for 7B models at 50 RPS.

---

## Phase 2 (Weeks 9–12): Live Migration

* Agent Migration API
  ✓ `POST /migrate { dest_agent }` RPC
  ✓ Multi-round token streaming (source → dest)
  ✓ Dest recomputes KV-cache; source drains final tokens

* Orchestrator Support
  ✓ Migration-aware scheduler

* End-to-End Tests
  ✓ Cold transfer at 80% completion; measure pause <200ms

* Metrics
  ✓ Migration count, average pause, throughput impact

**Milestone:** <200ms pause during live migration for 7B LLM.

---

## Phase 3 (Weeks 13–16): Layered/Shard Streaming

* Checkpoint Sharder
  ✓ `.bin` → per-transformer-block shards
  ✓ Update `.idx` with shard offsets

* Incremental Loader
  ✓ `load_and_serve(shard_list)` Zig API
  ✓ Load first block + embeddings, launch µService
  ✓ Background load remaining shards

* µService Integration
  ✓ Dynamic weight injection during inference

* Benchmarks
  ✓ First-token latency <100ms; P50 full response <1s

**Milestone:** First token <100ms; full response <1s for small prompts.

---

## Phase 4 (Weeks 17–20): Snapshot & P2P RDMA Mesh

* Container+GPU Snapshot
  ✓ CRIU + CUDA IPC to SSD snapshot
  ✓ `POST /restore {model_id}` <100ms restore

* Peer-to-Peer RDMA
  ✓ Gossip shard ownership
  ✓ Pull shards from peer NVMe → fallback to S3

* Tests & Metrics
  ✓ Snapshot latency
  ✓ Peer-cache hit ratio
  ✓ Reduced S3 usage

**Milestone:** Cold-start <100ms from snapshot; >80% shard fetch via P2P.

---

## Phase 5 (Weeks 21–24): Quantization & Decompression

* Dynamic Quantization Converter
  ✓ `.comp` (4/8-bit blobs) + `.idx` extension

* GPU Decompress Kernel
  ✓ Zig FFI wrapper for CUDA decompression
  ✓ Integrate into tiered loader

* Benchmarks
  ✓ Loader throughput, quantized accuracy delta

**Milestone:** 4-bit quantized loading at 2× throughput; <1% accuracy loss.

---

## Phase 6 (Weeks 25–28): Autoscaling & ML-Driven Scheduler

* Hot-Model Predictor
  ✓ Sliding window popularity model preload

* Agent Autoscaling Hooks
  ✓ Orchestrator emits GPU scale signals

* eBPF Telemetry
  ✓ Syscall & CUDA-level stats to Prometheus

* Online RL Scheduler
  ✓ Reinforcement learning to tune scheduling weights

**Milestone:** Dynamic scheduling reduces P95 by 20% vs. static weights.

---

## Phase 7 (Weeks 29+): QoS, Heterogeneous Fallback, Observability

* Multi-Tenant QoS
  ✓ Priority tiers, rate limits, per-tenant SLAs

* Heterogeneous Fallback
  ✓ CPU/TPU fallback for low-priority routes

* Full Observability
  ✓ Advanced Grafana + Jaeger + alerting

* Documentation & SDKs
  ✓ OpenAPI + Python/Go/JS clients

* Community & Open Source
  ✓ GitHub org, contribution flow, roadmap visibility

**Long-Term Vision:** A production-grade, globally distributed serverless LLM platform with sub-50ms cold starts, multi-backend scalability, and full QoS guarantees.
