# Helios: Serverless LLM Inference Platform — Architecture Overview

### Objective

Deliver scalable, performant, and flexible LLM inference with efficient multi-tier model storage and dynamic model management.

---

## Architecture Components

| Component              | Language | Responsibility                                                    |
| ---------------------- | -------- | ----------------------------------------------------------------- |
| **Inference Server**   | Python   | - Serve LLM inference requests via LitServe API                   |
|                        |          | - Load models from local cache or storage                         |
|                        |          | - Handle batching, streaming, request/response lifecycle          |
|                        |          | - Interface with Hugging Face or custom LLM frameworks            |
| **Storage Layer**      | Zig      | - Manage multi-tier storage (NVMe, SSD, cloud cache)              |
|                        |          | - Perform efficient async file IO, caching, eviction, prefetch    |
|                        |          | - Expose safe high-performance bindings callable from Python      |
| **Model Registry API** | Go       | - Interface with external model hubs (Hugging Face, custom repos) |
|                        |          | - Handle model download, versioning, metadata                     |
|                        |          | - Manage blob storage uploads/downloads (S3, GCS, Azure Blob)     |
|                        |          | - Serve model metadata and availability info to Inference Server  |

---

## Workflow

1. **Client Request:**
   User sends inference request → hits Python LitServe inference server.

2. **Cache Check:**
   Inference server queries Zig storage bindings to check local multi-tier cache for model files.

3. **Model Fetch:**
   If model is missing/stale → inference server calls Go Model Registry API to download and upload model artifacts to cloud blob storage, triggering cache update.

4. **Model Load:**
   Zig bindings load model weights/files asynchronously from local NVMe or fallback tiers.
ok do a final improvement of this I want to know each service and the language it needs to be high performance
5. **Inference:**
   Python LitServe runs model inference, applying batching and streaming results back to client.

---

## Design Highlights

* **Monolithic inference server (Python)** for ease of development and leveraging ML ecosystem.
* **Zig for storage layer** ensures fast, safe, and efficient multi-tier storage management with minimal overhead.
* **Go-based Model Registry API** provides scalable and robust cloud integrations for model lifecycle management.
* **Multi-tier caching** optimizes cost and latency (NVMe → SSD → Cloud Blob).
* **Open and extensible:** easy to integrate new storage tiers, model sources, or inference optimizations.

---

## Technology Rationale

| Technology | Reason                                                                             |
| ---------- | ---------------------------------------------------------------------------------- |
| Python     | Rich ML libraries, flexible inference orchestration, rapid prototyping             |
| Zig        | High-performance, safe low-level storage ops, easy FFI with Python                 |
| Go         | Network efficiency, concurrency, cloud SDKs, ease of building robust microservices |
| LitServe   | Flexible, general-purpose inference serving framework tailored for AI models       |

---

## Summary Diagram (Textual)

```
Client --> Python Inference Server (LitServe)
    |          |
    |          --> Zig Storage Bindings (Local multi-tier cache)
    |                     |
    |                     --> NVMe SSD <---> SSD <---> Cloud Blob Storage (S3/GCS/Azure)
    |
    --> Go Model Registry API
          |
          --> Hugging Face Hub / External Model Repos
          --> Cloud Blob Storage
```

