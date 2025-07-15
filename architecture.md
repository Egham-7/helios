# **Helios Specification**
---
## Objective
Scalable, high-performance LLM inference with fast cold start, dynamic GPU-aware scheduling, multi-tier model caching, intelligent memory estimation, and robust, declarative Kubernetes orchestration.
---
## Core Components
| Component | Language | Key Responsibilities |
| :--- | :--- | :--- |
| **Proxy** | Zig | - Route requests by model ID (<1ms overhead) |
| | | - Authenticate requests and gather data-plane metrics |
| | | - Query **Helios Scheduler** for worker availability and placement |
| | | - Circuit breaking, retries, and request batching |
| **Helios Scheduler** | Go | - Maintain warm pool of model-loaded worker pods via Operator |
| | | - **Perform Kubernetes-aware GPU bin-packing scheduling** |
| | | - Make real-time resource allocation decisions using **Memory Estimator** |
| | | - Drive predictive and reactive autoscaling logic |
| **Helios Operator** | Go | - Watch for `ManagedModel` Custom Resources (CRDs) |
| | | - **Reconcile desired state from CRDs with cluster state** |
| | | - Manage pod/service lifecycle (creation, deletion, updates) |
| | | - Report status back to the `ManagedModel` CRD |
| **Helios Prefill Workers** | Python | - **Specialized Pool:** Handle only the prefill stage |
| | | - Generate KV cache with continuous batching (e.g., using vLLM) |
| | | - Optimized for high-throughput, bursty computation |
| **Helios Decode Workers**| Python | - **Specialized Pool:** Handle only token decoding from a given KV cache |
| | | - Optimized for high-concurrency, long-running streaming tasks |
| | | - Can be scaled independently and packed densely on GPUs |
| **Storage Layer** | Zig | - Multi-tier caching: NVMe, SSD, RAM, Cloud |
| | | - Async zero-copy mmap model loading (safetensors) |
| | | - Parallel chunked downloads and prefetching |
| | | - Content-addressable storage with deduplication |
| **Model Registry** | Go | - Content-addressable caching with versioning |
| | | - Predictive prefetch and size-aware eviction |
| | | - Track usage patterns to inform the **Helios Scheduler** |
| | | - Integrate with HF Hub for automatic model discovery |
| **Memory Estimator** | Go | - Real-time GPU memory requirement calculation |
| | | - Dynamic config parsing from model registry |
| | | - Support inference, training, and fine-tuning modes |
| | | - Cache estimation results with TTL |
| | | - Provide resource allocation recommendations to the **Helios Scheduler** |
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
4. **Resource Planning:** Provide recommendations for GPU allocation, parallelism strategy, and optimization techniques to the **Helios Scheduler**.

### API Endpoints
* `POST /v1/estimate/memory` - Calculate memory requirements for given model and config
* `GET /v1/estimate/model/{model_id}` - Get cached memory profile for specific model
* `POST /v1/estimate/batch` - Batch estimation for multiple models
* `GET /v1/estimate/gpu-recommendations` - Get optimal GPU configurations for model list

---
## Workflow
1.  **Request Handling:** **Proxy** receives a request, authenticates it, and forwards a scheduling query to the **Helios Scheduler**.
2.  **Resource Planning:** The **Helios Scheduler** consults the **Memory Estimator** for resource requirements. It checks its view of the current warm pools and cluster state. If capacity is insufficient, it decides to scale up.
3.  **Declarative Deployment:** The **Helios Scheduler** creates or updates a `ManagedModel` Custom Resource in Kubernetes, declaring the desired state (e.g., "I need 3 replicas of `Llama-3-8B` with these resource requirements").
4.  **Lifecycle Management:** The **Helios Operator** detects the change to the `ManagedModel` CRD and takes action. It creates the necessary Kubernetes `Deployments` and `Services` for the prefill and decode workers, ensuring they are scheduled on appropriate GPU nodes.
5.  **Inference Execution:** The **Proxy** routes the request to an available **Prefill Worker**, which generates the KV cache. The task is then handed off to a **Decode Worker** for streaming token generation. Model weights are loaded zero-copy from the **Storage Layer**.
6.  **Dynamic Scaling:** The **Helios Scheduler** continuously monitors utilization metrics (Prometheus) and queue depths. It updates the `ManagedModel` CRDs to scale worker pools up or down, and the **Helios Operator** executes these changes.
---
## Performance Enhancements
* **Memory-Aware Scheduling:** Use memory estimator to prevent OOM errors and optimize resource allocation.
* **Disaggregated Scaling:** Independently scale Prefill and Decode worker pools to perfectly match workload characteristics and optimize cost.
* **Intelligent Batching:** Dynamic batching considers memory constraints and sequence length distribution.
* **Precision Optimization:** Automatic mixed-precision selection based on memory availability and model requirements.
* **Continuous dynamic batching** reduces latency and maximizes throughput.
* **Multi-tier Memory Management:** Intelligent KV cache placement across GPU RAM, host RAM, NVMe based on access patterns.
* **CUDA graph warm-up** and fused kernels (FlashAttention, BetterTransformer) improve efficiency.
* **Zero-copy mmap of safetensors** accelerates model loading.
* **Multi-GPU support** via tensor and pipeline parallelism with memory-aware partitioning.
* **GPU bin-packing scheduling** with memory constraints and warm pool optimization.
* **Predictive model preloading** based on usage patterns and available memory.
---
## API
### Core Inference
* `POST /v1/completions` (streaming supported)
* `POST /v1/chat/completions`
* `GET /v1/models` (with resource requirements, memory profiles, and warm pool status from the **Helios Scheduler**)

### Memory Estimation
* `POST /v1/estimate/memory` - Memory requirement calculation
* `GET /v1/estimate/gpu-compatibility/{model_id}` - GPU compatibility check
* `POST /v1/estimate/optimize` - Get optimization recommendations

### Admin & Monitoring
* Admin actions are now declarative via the Kubernetes API: `kubectl apply -f my-managed-model.yaml`
* `kubectl get managedmodels` - List all models managed by the platform and their status.
* `GET /v1/admin/memory-stats` - Real-time memory utilization from a monitoring endpoint.
* `GET /v1/health` - Health checks including memory pressure.
* `GET /v1/metrics` - Prometheus metrics with memory usage details.
---
## Deployment
### Infrastructure
* **Kubernetes** with heterogeneous GPU nodes (A100, H100, V100) and NVMe SSDs, 10Gbps+ networking.
* **Node Types:** Memory-optimized instances for large models, compute-optimized for smaller models. Can use different node pools for Prefill vs. Decode workers.
* **Storage:** Distributed storage layer with automatic tiering and prefetching.

### Resource Allocation
* **Proxy:** 2-4 CPU cores, 4-8GB RAM
* **Helios Scheduler:** 4 CPU cores, 8GB RAM (for caching cluster state)
* **Helios Operator:** 1 CPU core, 2GB RAM (lightweight reconciliation)
* **Memory Estimator:** 2 CPU cores, 4GB RAM, SSD storage for config caching
* **Registry:** 4 CPU cores, 8GB RAM, high-IOPS storage
* **Workers:** 8-32 CPU cores, 64-512GB RAM, 1-8 GPUs (allocated based on memory estimator recommendations and worker type: Prefill vs. Decode).

### Security & Networking
* **Service mesh** or mTLS for secure inter-service communication.
* **GPU bin-packing scheduler** with memory constraint awareness.
* **Pod affinity** and gang scheduling for multi-GPU workloads.
* **Network policies** for isolation and security.
* **Kubernetes RBAC** on `ManagedModel` CRDs to control who can deploy or modify models.
---
## Observability
### Metrics & Monitoring
* **Memory Metrics:** GPU memory utilization, host memory usage, KV cache hit rates, memory pressure alerts.
* **Performance:** Request latency, throughput, **prefill vs. decode queue depth**, batch efficiency.
* **Resource Utilization:** GPU/CPU/memory utilization across all nodes.
* **Operator Metrics:** Reconciliation loop latency, errors, number of managed resources.
* **Scheduler Metrics:** Scheduling decision latency, bin-packing efficiency.
* **Estimation Accuracy:** Track prediction vs actual memory usage for model improvements.

### Distributed Tracing
* **End-to-end request tracing** from Proxy -> **Scheduler** -> **Operator** -> **Prefill Worker** -> **Decode Worker**.
* **Memory allocation tracking** throughout request lifecycle.
* **Cache hit/miss patterns** and storage I/O performance.

### Alerting
* **Memory pressure** warnings and OOM prevention.
* **Resource saturation** across GPU/CPU/storage.
* **Model loading failures** and cache eviction events.
* **Estimation accuracy** degradation alerts.
* **Service health** and dependency failures.
* **Helios Operator** reconciliation failures.

### Dashboards
* **Real-time resource utilization** with memory breakdown by model and worker type (Prefill/Decode).
* **Cost optimization** recommendations based on usage patterns.
* **Capacity planning** with growth projections.
* **Performance analysis** with memory-aware optimization suggestions.
