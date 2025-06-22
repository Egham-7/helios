# ServerlessLLM Roadmap

Features to tackle post-MVP, each scoped for its own sprint:

1. **Live Migration**  
   • Multi-round token streaming, recompute KV-cache on dest, zero downtime.

2. **Layered/Shard Streaming**  
   • Break `.bin` into transformer-block shards; start inference on layer-0 while loading rest.

3. **Container + GPU Snapshot/Restore**  
   • CRIU + CUDA IPC to checkpoint warmed containers → SSD; restore in <100 ms.

4. **GPUDirect-RDMA P2P Mesh**  
   • Agent ring: fetch model chunks via RDMA from nearest peer instead of S3.

5. **Dynamic Quantization & Decompression**  
   • Store compressed weights (4/8-bit); on-GPU decompress kernel during load.

6. **Tail-Latency-Aware Scheduler**  
   • eBPF-driven telemetry + online RL to minimize P95/P99 under bursty loads.

7. **Autoscaling & Hot-Model Pre-Cache**  
   • Popularity predictor preloads top-K models into DRAM/NVMe; elastic agent pool.

8. **QoS & Multi-Tenant Fairness**  
   • Per-tenant rate limits, priority lanes, backpressure streaming.

9. **Heterogeneous Fallback**  
   • CPU (AVX-512) or TPU fallback for small models or GPU shortages.

10. **Adaptive Tuning & Observability**  
    • Prometheus + Grafana dashboards; eBPF-based dynamic adjustment of chunk/thread parameters.
