**roadmap.md**

# ServerlessLLM Development Roadmap

A phased plan to take ServerlessLLM from MVP to a full-featured, high-performance serverless LLM platform.

---

## Phase 0 (Weeks 1–4): MVP Completion

Deliver a working end-to-end prototype with OpenAI-spec API.

• Checkpoint Conversion CLI  
 – HF/PyTorch → `.idx` + `.bin`  
 – Python + Rust library  
• Model Ingestion Service  
 – On-demand raw HF download + conversion  
 – Python container  
• Metadata KV Store  
 – etcd/Redis schema + agent heartbeat  
• Orchestrator (Rust)  
 – OpenAI API endpoints  
 – Scheduling by cache_level + kv_hit + queue_len + size/bw  
 – Fan-out downloader trigger  
 – Proxy SSE/gRPC streams  
• Agent Daemon  
 – Control-plane (Rust): heartbeat, queue worker, `/load`, `/infer`  
 – Data-plane (Zig): multi-tier loader (DRAM/NVMe/S3 → CUDA DMA)  
 – Inference µService (Rust + vLLM/Triton adapter)  
• CI/CD + Deployment  
 – GitHub Actions to convert & push optimized artifacts  
 – Helm/Terraform for Orchestrator, Agent, etcd/Redis  
• Metrics & Dashboards  
 – Prometheus export (load times, kv_hit_rate, queue_len)  
 – Grafana basic panels

**Milestone:** Serve first-token in <2s for 7B models; P50 cold-start <3s.

---

## Phase 1 (Weeks 5–8): Scheduling & Downloader Optimization

Improve startup latency and resource efficiency.

• **Fan-out Downloader**  
 – Parallel HTTP-Range chunking in Zig  
 – Benchmark NVMe ingest speed, tune chunk size  
• **Enhanced Scheduler**  
 – Incorporate real agent‐observed bandwidth metrics  
 – Weight tuning for α,β,γ,δ parameters  
 – CLI to adjust scheduling weights at runtime  
• **KV-cache Hit Awareness**  
 – Agent tracks per-token GPU KV-cache hits  
 – Scheduler uses `kv_hit_rate` to bias cached agents  
• **Retry Logic & Timeouts**  
 – Orchestrator handles agent failures, retries to next best agent  
• **Integration Tests**  
 – Simulate bursty workloads; verify P95 cold-start under 5s

**Milestone:** P95 cold-start <2s for 7B models at 50 RPS.

---

## Phase 2 (Weeks 9–12): Live Migration

Enable moving in-flight inference with minimal interruption.

• **Agent Migration API**  
 – `POST /migrate { dest_agent }` RPC  
 – Multi-round token streaming (source → dest)  
 – Dest recomputes KV-cache; source drains final tokens  
• **Orchestrator Support**  
 – Scheduler considers migration cost vs. load time  
 – Route updates mid-inference  
• **End-to-End Tests**  
 – Cold transfer at 80% completion; measure pause <200ms  
• **Metrics**  
 – Migration count, average pause, throughput impact

**Milestone:** <200ms pause during live migration for 7B LLM.

---

## Phase 3 (Weeks 13–16): Layered/Shard Streaming

Overlap loading and inference to reduce first-token latency.

• **Checkpoint Sharder**  
 – Convert `.bin` → per-transformer-block shards  
 – Update `.idx` with shard offsets  
• **Incremental Loader**  
 – Zig loader API: `load_and_serve(shard_list)`  
 – Stage-0: load embeddings + first block → launch µService  
 – Background load remaining shards  
• **µService Integration**  
 – Inference engine must accept dynamic weight injection  
• **Benchmarks**  
 – First-token latency <100ms on 7B; P50 total <1s

**Milestone:** First token <100ms; full response <1s for small prompts.

---

## Phase 4 (Weeks 17–20): Snapshot & P2P RDMA Mesh

Dramatically reduce cold-start via snapshot restore and peer fetch.

• **Container+GPU Snapshot**  
 – Integrate CRIU + CUDA IPC to dump loaded container → SSD  
 – `POST /restore {model_id}` → restore in <100ms  
• **Peer-to-Peer RDMA**  
 – Agents gossip shard ownership via KV  
 – On cache miss → RDMA pull shards from peer NVMe → local load  
 – Fallback to S3 if peer unavailable  
• **Tests & Metrics**  
 – Snapshot restore latency  
 – Peer-cache hit ratio  
 – Reduced S3 bandwidth usage

**Milestone:** Cold-start <100ms from snapshot; >80% shard fetch via P2P.

---

## Phase 5 (Weeks 21–24): Quantization & Decompression

Cut bandwidth & memory by 4×–8× using low-bit weights.

• **Dynamic Quantization Converter**  
 – Extend Checkpoint Converter: produce `.comp` (4/8-bit blobs) + updated `.idx`  
• **GPU Decompress Kernel**  
 – Zig FFI for CUDA kernel that expands `.comp` → float16/32  
 – Integrate into loader pipeline  
• **Benchmarks**  
 – Loader throughput with compressed weights  
 – Model accuracy validation

**Milestone:** 4-bit quantized loading at 2× throughput; <1% accuracy loss.

---

## Phase 6 (Weeks 25–28): Autoscaling & ML-Driven Scheduler

Optimize resource use and tail-latency via learning.

• **Hot-Model Predictor**  
 – Simple sliding-window popularity service → orchestrator preloads top-K models  
• **Agent Autoscaling Hooks**  
 – Orchestrator emits scale-up/down signals to GPU pool manager  
• **eBPF Telemetry**  
 – Per-syscall, per-CUDA transfer stats from agents → Prometheus  
• **Online RL Scheduler**  
 – Feed real latency metrics to lightweight RL agent  
 – Continuously tune α,β,γ,δ weights to minimize P95 under live load

**Milestone:** Dynamic scheduling reduces P95 by 20% vs. static weights.

---

## Phase 7 (Weeks 29+): QoS, Heterogeneous Fallback, Observability

• **Multi-Tenant QoS**  
 – Rate limits, priority lanes, per-tenant SLAs  
• **Heterogeneous Fallback**  
 – CPU (AVX-512) or TPU edge inference for low-priority or small models  
• **Full Observability**  
 – Enhanced Grafana dashboards, alerting, tracing (Jaeger)  
• **Documentation & SDKs**  
 – Publish client SDKs (Python, Go, JS) for easy integration  
• **Community & Open Source**  
 – Public GitHub org, contribution guidelines, roadmap transparency

**Long-Term Vision:** A production-grade, globally distributed serverless LLM platform with sub-50ms cold starts, multi-backend scalability, and full QoS guarantees.
