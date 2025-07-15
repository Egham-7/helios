# Helios MVP: Spec
---
## Objective
Scalable, high-performance LLM inference with fast cold start, dynamic GPU-aware scheduling, multi-tier model caching, intelligent memory estimation, and Kubernetes orchestration.
---
## Core Components
| Component              | Language | Key Responsibilities                                      |
| ---------------------- | -------- | --------------------------------------------------------- |
| **Proxy**              | Go       | - Route requests by model ID (<1ms overhead)              |
|                        |          | - Cache memory profiles and resource requirements          |
|                        |          | - Maintain warm pool of model-loaded worker pods          |
|                        |          | - Circuit breaking, retries, batching hints               |
|                        |          | - Kubernetes-aware GPU bin-packing scheduling             |
|                        |          | - Real-time resource allocation decisions                  |
| **vLLM Workers**       | Python   | - Prefill: KV cache generation with continuous batching   |
|                        |          | - Decode: Streaming token generation                      |
|                        |          | - Multi-tenant GPU sharing via MIG slices                 |
|                        |          | - Support mixed-precision and multi-GPU parallelism       |
|                        |          | - Dynamic memory management and overflow handling          |
| **Storage Layer**      | Zig      | - Multi-tier caching: NVMe, SSD, RAM, Cloud               |
|                        |          | - Async zero-copy mmap model loading (safetensors)        |
|                        |          | - Parallel chunked downloads and prefetching              |
|                        |          | - Content-addressable storage with deduplication          |
| **Model Registry**     | Go       | - Content-addressable caching with versioning             |
|                        |          | - Predictive prefetch and size-aware eviction             |
|                        |          | - Track usage patterns and maintain warm pools            |
|                        |          | - Integrate with HF Hub for automatic model discovery     |
| **Memory Estimator**   | Go       | - Real-time GPU memory requirement calculation            |
|                        |          | - Dynamic config parsing from model registry              |
|                        |          | - Support inference, training, and fine-tuning modes      |
|                        |          | - Cache estimation results with TTL                       |
|                        |          | - Provide resource allocation recommendations              |
---
## Memory Estimation Service Integration
### Data Flow
1. **Config Retrieval:** Memory estimator queries model registry for cached model configs (local disk lookup, ~0.1ms)
2. **Fallback Strategy:** If not cached, fetch from HF Hub API and cache locally
3. **Dynamic Calculation:** Compute memory requirements based on:
   - Model architecture (layers, hidden size, attention heads)
   - Precision settings (FP32/FP16/INT8/INT4)
   - Inference parameters (batch size, sequence length, KV cache)
   - GPU type and memory constraints
4. **Resource Planning:** Provide recommendations for GPU allocation, parallelism strategy, and optimization techniques

### API Endpoints
* `POST /v1/estimate/memory` - Calculate memory requirements for given model and config
* `GET /v1/estimate/model/{model_id}` - Get cached memory profile for specific model
* `POST /v1/estimate/batch` - Batch estimation for multiple models
* `GET /v1/estimate/gpu-recommendations` - Get optimal GPU configurations for model list

---
## Workflow
1. **Request Handling:** Proxy routes by model ID, consults memory estimator for resource requirements, checks cached profiles, ensures warm worker availability with sufficient memory, triggers scaling if needed.
2. **Resource Planning:** Memory estimator calculates optimal GPU allocation, determines if multi-GPU parallelism needed, estimates KV cache overflow requirements.
3. **Worker Selection:** GPU-aware scheduling considers memory requirements, prefers warm workers with adequate resources, enables intelligent multi-tenant GPU sharing, supports tensor/pipeline parallelism based on memory constraints.
4. **Inference Execution:** Prefill workers batch inputs to create KV caches within memory bounds; decode workers stream tokens; model weights loaded zero-copy from storage; KV caches managed across GPU/host memory/NVMe based on estimator recommendations.
5. **Dynamic Scaling:** Monitor utilization, queue depth, and memory pressure; autoscale worker pools based on memory estimator predictions; use predictive scaling and cluster autoscaler to add appropriate GPU nodes.
---
## Performance Enhancements
* **Memory-Aware Scheduling:** Use memory estimator to prevent OOM errors and optimize resource allocation
* **Intelligent Batching:** Dynamic batching considers memory constraints and sequence length distribution
* **Precision Optimization:** Automatic mixed-precision selection based on memory availability and model requirements
* **Continuous dynamic batching reduces latency and maximizes throughput
* **Multi-tier Memory Management:** Intelligent KV cache placement across GPU RAM, host RAM, NVMe based on access patterns
* **CUDA graph warm-up and fused kernels (FlashAttention, BetterTransformer) improve efficiency
* **Zero-copy mmap of safetensors accelerates model loading
* **Multi-GPU support via tensor and pipeline parallelism with memory-aware partitioning
* **GPU bin-packing scheduling with memory constraints and warm pool optimization
* **Predictive model preloading based on usage patterns and available memory
---
## API
### Core Inference
* `POST /v1/completions` (streaming supported)
* `POST /v1/chat/completions`
* `GET /v1/models` (with resource requirements, memory profiles, and warm pool status)

### Memory Estimation
* `POST /v1/estimate/memory` - Memory requirement calculation
* `GET /v1/estimate/gpu-compatibility/{model_id}` - GPU compatibility check
* `POST /v1/estimate/optimize` - Get optimization recommendations

### Admin & Monitoring
* `POST /v1/admin/preload` - Preload models with memory validation
* `GET /v1/admin/memory-stats` - Real-time memory utilization
* `POST /v1/admin/scale` - Manual scaling with memory considerations
* `DELETE /v1/admin/evict/{model_id}` - Intelligent model eviction
* `GET /v1/health` - Health checks including memory pressure
* `GET /v1/metrics` - Prometheus metrics with memory usage details
---
## Deployment
### Infrastructure
* **Kubernetes** with heterogeneous GPU nodes (A100, H100, V100) and NVMe SSDs, 10Gbps+ networking
* **Node Types:** Memory-optimized instances for large models, compute-optimized for smaller models
* **Storage:** Distributed storage layer with automatic tiering and prefetching

### Resource Allocation
* **Proxy:** 4 CPU cores, 8GB RAM (increased for memory estimation caching)
* **Memory Estimator:** 2 CPU cores, 4GB RAM, SSD storage for config caching
* **Registry:** 4 CPU cores, 8GB RAM, high-IOPS storage
* **Workers:** 8-32 CPU cores, 64-512GB RAM, 1-8 GPUs (allocated based on memory estimator recommendations)

### Security & Networking
* **Service mesh** or mTLS for secure inter-service communication
* **GPU bin-packing scheduler** with memory constraint awareness
* **Pod affinity** and gang scheduling for multi-GPU workloads
* **Network policies** for isolation and security
---
## Observability
### Metrics & Monitoring
* **Memory Metrics:** GPU memory utilization, host memory usage, KV cache hit rates, memory pressure alerts
* **Performance:** Request latency, throughput, queue depth, batch efficiency
* **Resource Utilization:** GPU/CPU/memory utilization across all nodes
* **Estimation Accuracy:** Track prediction vs actual memory usage for model improvements

### Distributed Tracing
* **End-to-end request tracing** from proxy through memory estimation to inference completion
* **Memory allocation tracking** throughout request lifecycle
* **Cache hit/miss patterns** and storage I/O performance

### Alerting
* **Memory pressure** warnings and OOM prevention
* **Resource saturation** across GPU/CPU/storage
* **Model loading failures** and cache eviction events
* **Estimation accuracy** degradation alerts
* **Service health** and dependency failures

### Dashboards
* **Real-time resource utilization** with memory breakdown by model
* **Cost optimization** recommendations based on usage patterns
* **Capacity planning** with growth projections
* **Performance analysis** with memory-aware optimization suggestions
