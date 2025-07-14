

# Helios MVP: Spec

---

## Objective

Scalable, high-performance LLM inference with fast cold start, dynamic GPU-aware scheduling, multi-tier model caching, and Kubernetes orchestration.

---

## Core Components

| Component          | Language | Key Responsibilities                                    |
| ------------------ | -------- | ------------------------------------------------------- |
| **Proxy**          | Go       | - Route requests by model ID (<1ms overhead)            |
|                    |          | - Cache Accelerate memory profiles                      |
|                    |          | - Maintain warm pool of model-loaded worker pods        |
|                    |          | - Circuit breaking, retries, batching hints             |
|                    |          | - Kubernetes-aware GPU bin-packing scheduling           |
| **vLLM Workers**   | Python   | - Prefill: KV cache generation with continuous batching |
|                    |          | - Decode: Streaming token generation                    |
|                    |          | - Multi-tenant GPU sharing via MIG slices               |
|                    |          | - Support mixed-precision and multi-GPU parallelism     |
| **Storage Layer**  | Zig      | - Multi-tier caching: NVMe, SSD, RAM, Cloud             |
|                    |          | - Async zero-copy mmap model loading (safetensors)      |
|                    |          | - Parallel chunked downloads and prefetching            |
| **Model Registry** | Go       | - Content-addressable caching with versioning           |
|                    |          | - Predictive prefetch and size-aware eviction           |
|                    |          | - Track usage and maintain warm pools                   |

---

## Workflow

1. **Request Handling:** Proxy routes by model ID, checks cached resource profiles, ensures warm worker availability, triggers scaling if needed.

2. **Worker Selection:** GPU-aware scheduling prefers warm workers, enables multi-tenant GPU sharing, supports tensor/pipeline parallelism.

3. **Inference Execution:** Prefill workers batch inputs to create KV caches; decode workers stream tokens; model weights loaded zero-copy from storage; KV caches spill over NVMe/RAM as needed.

4. **Dynamic Scaling:** Monitor utilization and queue depth; autoscale worker pools; use predictive scaling and cluster autoscaler to add GPU nodes.

---

## Performance Enhancements

* Continuous dynamic batching reduces latency and maximizes throughput.
* Mixed-precision inference (FP16 default, optional INT8/4-bit quantization).
* CUDA graph warm-up and fused kernels (FlashAttention, BetterTransformer) improve efficiency.
* Zero-copy mmap of safetensors accelerates model loading.
* Multi-GPU support via tensor and pipeline parallelism.
* GPU bin-packing scheduling and warm pool to minimize cold starts.
* Spill and prefetch KV caches across GPU RAM, host RAM, NVMe tiers for memory efficiency.

---

## API

* `POST /v1/completions` (streaming supported)
* `POST /v1/chat/completions`
* `GET /v1/models` (with resource and warm pool status)
* Admin endpoints for model preload, scaling, eviction, and monitoring.
* Health and Prometheus metrics endpoints.

---

## Deployment

* Kubernetes with GPU nodes and NVMe SSDs, 10Gbps networking.
* Proxy: 2 CPU cores, 4GB RAM; Registry: 4 CPU, 8GB RAM; Workers: 8-16 CPU cores, 64+GB RAM, 1-8 GPUs.
* Service mesh or mTLS for secure communication.
* Use GPU bin-packing scheduler, pod affinity, and gang scheduling for multi-GPU workloads.

---

## Observability

* Distributed tracing and detailed latency/throughput metrics.
* GPU/CPU/memory utilization tracking.
* Cache hit rates and storage I/O stats.
* Alerting on resource saturation and errors.
