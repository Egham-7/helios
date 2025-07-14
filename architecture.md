# Helios: Serverless Disaggregated LLM Inference Platform Architecture

### Objective
Deliver scalable, high-performance disaggregated LLM inference with efficient multi-tier model storage, dynamic model management via Kubernetes orchestration, and intelligent model registry caching.

---

## Core Architecture Components

| Component              | Language | Responsibility                                                    |
| ---------------------- | -------- | ----------------------------------------------------------------- |
| **Disaggregated Proxy** | **Go**   | - Route requests based on model ID to appropriate worker pools    |
|                        |          | - Load balance across prefill/decode instances                    |
|                        |          | - Integrate with Kubernetes API for dynamic scaling               |
|                        |          | - Handle service discovery and health checking                     |
|                        |          | - Provide observability metrics and request tracing               |
| **vLLM Workers**       | **Python** | - Execute LLM inference via vLLM framework                      |
|                        |          | - Prefill workers: Generate KV caches from prompts               |
|                        |          | - Decode workers: Generate tokens using shared KV caches          |
|                        |          | - Handle batching, streaming, and request lifecycle               |
|                        |          | - Interface with Zig storage layer for model loading              |
| **Storage Layer**      | **Zig**  | - Manage multi-tier storage (NVMe, SSD, cloud cache)              |
|                        |          | - Perform efficient async file IO, caching, eviction, prefetch    |
|                        |          | - Expose safe high-performance bindings callable from Python      |
|                        |          | - Handle model weight distribution across worker nodes            |
| **Model Registry**     | **Go**   | - Cache Hugging Face models locally to avoid repeated downloads   |
|                        |          | - Manage model metadata, versioning, and availability             |
|                        |          | - Handle blob storage operations (S3, GCS, Azure Blob)            |
|                        |          | - Serve model artifacts to storage layer efficiently              |
|                        |          | - Track model usage patterns and optimize caching strategies      |

---

## System Workflow

### Request Processing Pipeline
1. **Client Request Ingestion**
   - Requests arrive at Go proxy with model ID and inference parameters
   - Request validation, rate limiting, and authentication occur at edge
   - Model ID resolution to canonical format with version pinning

2. **Model Availability Verification**
   - Model Registry confirms model availability in distributed storage
   - Checks span local cache, blob storage, and Hugging Face fallback
   - Triggers asynchronous model download if not locally available

3. **Worker Pool Resolution**
   - Kubernetes service discovery locates healthy prefill/decode workers
   - Load balancing algorithm considers queue depth, latency, and resource utilization
   - Circuit breaker patterns prevent cascading failures

4. **Disaggregated Inference Execution**
   - Prefill workers generate KV caches from input prompts
   - Zig storage layer delivers model weights with sub-100ms latency
   - vLLM manages internal KV cache transfer between prefill and decode stages
   - Decode workers stream token generation back through proxy

5. **Dynamic Resource Management**
   - Real-time monitoring of worker health, queue depth, and performance metrics
   - Kubernetes HPA triggers automatic scaling based on demand patterns
   - Graceful worker lifecycle management with proper health checks

---

## Model Registry Strategy

### Intelligent Caching Architecture
The Model Registry implements a sophisticated multi-tier caching strategy designed to minimize Hugging Face API calls and optimize model availability across the cluster.

**Cache Hierarchy:**
- **Memory Cache:** Hot model metadata for sub-millisecond lookup
- **Local Blob Storage:** Frequently accessed models on cluster storage
- **Cloud Blob Storage:** Complete model repository with versioning
- **Hugging Face Fallback:** Source of truth for new models

**Caching Intelligence:**
- **Predictive Prefetching:** Machine learning models predict usage patterns
- **LRU Eviction:** Intelligent removal of least recently used models
- **Locality Optimization:** Ensures popular models exist on high-speed storage tiers
- **Version Management:** Handles model updates while maintaining backward compatibility

### Model Distribution Strategy
Models are distributed across worker nodes using a content-addressable storage approach. Each model is identified by a cryptographic hash of its contents, enabling:
- **Deduplication:** Identical model components shared across deployments
- **Integrity Verification:** Automatic detection of corrupted model files
- **Incremental Updates:** Only changed components are redistributed
- **Parallel Downloads:** Concurrent fetching of model components across nodes

---

## Kubernetes Orchestration

### Worker Pool Management
The platform leverages Kubernetes for dynamic worker pool orchestration, treating each model as a separate deployment with independent scaling characteristics.

**Deployment Strategy:**
- **Model-Specific Deployments:** Each model gets dedicated prefill/decode deployments
- **Resource Tagging:** GPU type, memory, and storage requirements specified per model
- **Affinity Rules:** Co-locate prefill/decode workers for optimal KV cache transfer
- **Anti-Affinity:** Distribute replicas across availability zones for resilience

**Scaling Intelligence:**
- **Multi-Metric HPA:** CPU, memory, GPU utilization, queue depth, and latency
- **Predictive Scaling:** Historical usage patterns inform pre-emptive scaling
- **Cost Optimization:** Scale-to-zero for unused models with configurable warmup
- **Burst Handling:** Rapid scale-up for traffic spikes with overprovisioning

### Service Mesh Integration
Utilizes Istio service mesh for advanced traffic management and observability:
- **Circuit Breaking:** Automatic failure isolation and recovery
- **Retry Policies:** Intelligent retry with exponential backoff
- **Load Balancing:** Advanced algorithms including consistent hashing
- **Security:** mTLS between all components with certificate rotation

---

## Storage Layer Architecture

### Multi-Tier Storage Hierarchy
The Zig-based storage layer implements a sophisticated caching hierarchy optimized for LLM inference workloads.

**Storage Tiers:**
- **NVMe (L1):** Ultra-fast local storage for active model weights
- **SSD (L2):** High-speed local storage for recently used models
- **Memory Pool (L3):** Shared memory cache across worker nodes
- **Cloud Storage (L4):** Distributed blob storage for model repository

**Performance Characteristics:**
- **NVMe Tier:** <100ms model loading, 95% cache hit rate target
- **SSD Tier:** <500ms model loading, serves cache misses from NVMe
- **Memory Pool:** <50ms for small model components and metadata
- **Cloud Storage:** <5s for cold model loading with background warming

### Intelligent Caching Logic
**Admission Policies:**
- **Frequency-Based:** Models accessed multiple times within time window
- **Size-Aware:** Prefer smaller models for cache efficiency
- **Prediction-Driven:** Machine learning models predict future access patterns

**Eviction Strategies:**
- **LRU with Aging:** Combines recency with frequency of access
- **Size-Weighted:** Consider storage footprint in eviction decisions
- **Priority Classes:** Critical models marked for cache persistence

---

## Observability and Monitoring

### Metrics Collection
**Application Metrics:**
- **Request Latency:** P50, P95, P99 latencies across all endpoints
- **Throughput:** Requests per second, tokens per second per model
- **Error Rates:** HTTP error codes, inference failures, timeout rates
- **Queue Metrics:** Queue depth, wait times, worker utilization

**Infrastructure Metrics:**
- **Resource Utilization:** CPU, memory, GPU utilization per worker
- **Storage Performance:** Cache hit rates, I/O latency across tiers
- **Network Metrics:** Bandwidth, connection counts, packet loss
- **Kubernetes Metrics:** Pod lifecycle, scaling events, resource allocation

### Distributed Tracing
Implements OpenTelemetry for end-to-end request tracing across all components:
- **Request Flow:** Complete trace from client to final response
- **Component Latency:** Individual latency breakdown per service
- **Error Attribution:** Precise error location and stack traces
- **Performance Profiling:** Hotspot identification and optimization opportunities

---

## Performance Characteristics

### Latency Targets
**End-to-End Performance:**
- **Request Routing:** <1ms proxy overhead for request dispatch
- **Model Loading:** <100ms for cached models, <5s for cold models
- **Prefill Stage:** 50-500ms depending on prompt length and model complexity
- **Decode Stage:** 10-100ms per token based on model size and hardware
- **First Token Time:** <200ms for cached models, <2s for cold starts

**Throughput Expectations:**
- **Proxy Capacity:** 50,000+ RPS routing capability with Go concurrency
- **Worker Throughput:** 100-1000 tokens/sec per GPU depending on model
- **Storage Bandwidth:** 10GB/s aggregate throughput across storage tiers
- **Network Efficiency:** <5% overhead for inter-service communication

### Scalability Boundaries
**Horizontal Scaling:**
- **Worker Pools:** 1-100 workers per model with automatic scaling
- **Storage Nodes:** Distributed across 3-50 Kubernetes nodes
- **Geographic Distribution:** Multi-region deployment with edge caching
- **Model Capacity:** Support for 100+ models simultaneously

---

## Deployment Architecture

### Infrastructure Requirements
**Minimum Production Setup:**
- **Control Plane:** 3-node Kubernetes cluster with HA control plane
- **Worker Nodes:** GPU-enabled nodes with NVMe storage
- **Storage Infrastructure:** Distributed storage with replication factor 3
- **Network Requirements:** 10Gbps inter-node connectivity minimum

**Resource Allocation:**
- **Proxy Service:** 2-4 CPU cores, 4-8GB RAM per replica
- **Model Registry:** 4-8 CPU cores, 8-16GB RAM with SSD storage
- **vLLM Workers:** 8-16 CPU cores, 32-128GB RAM, 1-8 GPUs per pod
- **Storage Layer:** Dedicated storage nodes with high-speed NVMe arrays

### Security Architecture
**Zero-Trust Principles:**
- **Service-to-Service:** mTLS encryption for all inter-service communication
- **API Security:** OAuth 2.0/JWT authentication with fine-grained RBAC
- **Model Protection:** Encrypted model storage with access auditing
- **Network Segmentation:** Kubernetes network policies isolate workloads

---

## Technology Rationale

### Language Selection Justification
**Go for Proxy and Registry:**
- Exceptional concurrency model ideal for high-throughput request routing
- Native Kubernetes integration with official client libraries
- Fast compilation and deployment suitable for CI/CD pipelines
- Strong ecosystem for cloud-native microservices development

**Python for vLLM Workers:**
- Required for vLLM framework and PyTorch ecosystem integration
- Rich machine learning libraries and community support
- Proven performance for GPU-accelerated inference workloads
- Extensive debugging and profiling tools for ML applications

**Zig for Storage Layer:**
- Zero-cost abstractions enable maximum I/O performance
- Memory safety without garbage collection overhead
- Excellent FFI capabilities for Python integration
- Compile-time optimizations crucial for storage-intensive operations

### Risk Mitigation
**Zig Maturity Concerns:**
- Implement comprehensive testing with fallback to C libraries
- Gradual rollout with A/B testing against baseline implementations
- Maintain abstraction layer allowing future storage backend swaps
- Continuous monitoring for stability and performance regressions

---

## Summary Architecture Diagram

```
                           ┌─────────────────┐
                           │   Client Apps   │
                           └─────────┬───────┘
                                     │ HTTPS/gRPC
                           ┌─────────▼───────┐
                           │  Go Proxy       │
                           │  (Load Balance) │
                           └─────────┬───────┘
                                     │ K8s Service Discovery
                    ┌────────────────┼────────────────┐
                    │                │                │
            ┌───────▼────────┐              ┌───────▼────────┐
            │ Prefill Pool   │              │ Decode Pool    │
            │ (vLLM Python)  │◄────KV──────►│ (vLLM Python)  │
            └───────┬────────┘   Cache      └───────┬────────┘
                    │                               │
                    └──────────┬────────────────────┘
                               │ Model Loading
                    ┌──────────▼──────────┐
                    │   Zig Storage       │
                    │   Multi-Tier Cache  │
                    └─────────┬───────────┘
                              │ Model Artifacts
                    ┌─────────▼───────────┐
                    │   Go Model Registry │
                    │   (HF Cache Layer)  │
                    └─────────────────────┘
```
