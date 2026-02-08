# GPU Infrastructure, Model Serving, and API Gateway Patterns for AI Workloads

## GPU Infrastructure Patterns

### Dedicated GPU Hosts

The simplest model. Bare-metal or VM hosts with passthrough GPUs dedicated to specific workloads.

- **When it fits:** Inference for a single high-priority model, training jobs that saturate the GPU
- **Trade-off:** Low utilization if the workload is bursty. A single A100 at 20% average utilization is expensive idle metal
- **vSphere angle:** VMware DirectPath I/O (passthrough) gives near-native performance but pins the GPU to one VM, defeating vMotion

### GPU Sharing / Time-Slicing

Multiple workloads share a physical GPU via NVIDIA MPS (Multi-Process Service), MIG (Multi-Instance GPU) on A100/H100, or vGPU profiles.

- **MIG:** Physically partitions the GPU into isolated instances (e.g., an A100 splits into up to 7 instances). Each gets dedicated memory and compute. Best for inference where you need isolation guarantees.
- **MPS:** Software-level sharing. Multiple CUDA contexts share the GPU with less overhead than time-slicing. Good for many small inference requests but no memory isolation.
- **vGPU (NVIDIA GRID):** VMware-native. Lets you carve a physical GPU into virtual GPU profiles assigned to VMs. Adds a licensing cost but integrates with vSphere management, DRS (with constraints), and HA.

### GPU Pools / Clusters

Kubernetes-based GPU scheduling using the NVIDIA GPU Operator and device plugin. Workloads request GPU resources like any other Kubernetes resource.

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

- Works with both on-prem (Tanzu, RKE, vanilla k8s) and cloud (GKE, EKS, AKS)
- NVIDIA's GPU Operator handles driver lifecycle, toolkit, and device plugin
- DRA (Dynamic Resource Allocation) in Kubernetes 1.30+ enables finer-grained GPU sharing

---

## Model Serving Patterns

### Single-Model Server

One model per process/container. The model loads into GPU memory at startup and stays resident.

- **Frameworks:** TorchServe, TensorFlow Serving, Triton Inference Server (single-model mode)
- **When it fits:** Production models with consistent traffic. Predictable memory footprint, simple scaling
- **Scaling:** Horizontal -- add replicas behind a load balancer

### Multi-Model Server

Multiple models share a single serving process and GPU. Models are loaded/unloaded based on demand.

- **Frameworks:** Triton Inference Server, BentoML, Seldon Core
- **When it fits:** Many models with low individual traffic (e.g., per-customer fine-tuned models)
- **Risk:** Cold-start latency when a model needs to be loaded. Memory pressure if too many models are hot simultaneously

### LLM-Specific Serving

Large language models have unique requirements -- large memory footprint, KV-cache management, batching strategies.

- **Frameworks:** vLLM, TGI (Text Generation Inference), TensorRT-LLM
- **Key techniques:**
  - **Continuous batching:** Don't wait for a full batch. Add new requests to in-flight batches as tokens complete
  - **PagedAttention (vLLM):** Manages KV-cache like virtual memory pages, reducing waste from pre-allocated cache
  - **Tensor parallelism:** Splits a single model across multiple GPUs on the same node for models that don't fit in one GPU's memory
  - **Speculative decoding:** Uses a small draft model to propose tokens, verified in parallel by the large model

### Model Pipeline / Ensemble

Chain multiple models together -- e.g., preprocessing, embedding, ranking, post-processing.

- **Frameworks:** Triton ensemble models, Seldon Core inference graphs, KServe InferenceGraph
- **Example flow:** `Text Input -> Tokenizer -> Embedding Model -> Reranker -> Post-processor`
- **When it fits:** RAG pipelines, multi-stage recommendation systems

---

## API Gateway Patterns

### Standard Reverse Proxy

Route `/v1/models/{model_id}/predict` to the appropriate model server. Nothing AI-specific.

- **Tools:** NGINX, Envoy, Kong, HAProxy
- **Handles:** TLS termination, rate limiting, auth, request routing
- **Limitation:** No awareness of model-specific concerns like token counting or queue depth

### Token-Aware Gateway

Purpose-built for LLM workloads. Understands token counts, manages per-user quotas in tokens-per-minute rather than requests-per-minute.

- **Key capabilities:**
  - Token counting on ingest (estimate cost before forwarding)
  - Token-based rate limiting (RPM and TPM)
  - Streaming response passthrough (SSE for `/chat/completions`)
  - Request priority queuing based on token budget
- **Tools:** LiteLLM Proxy, Portkey, custom Envoy filters

### Model Router

Routes requests to different model backends based on request characteristics -- model version, input complexity, cost tier.

```
User Request
    |
    v
+---------------+
|  Model Router |
+-------+-------+
        |----> Large model (complex queries)
        |----> Small model (simple queries)
        +----> Local fine-tuned model (domain-specific)
```

- **When it fits:** Cost optimization. Route simple queries to cheaper/faster models, complex ones to capable models
- **Complexity:** Requires a classifier or heuristic to decide routing, which itself adds latency

### Inference Queue Pattern

Decouple request ingestion from model execution. Useful when inference time is long or variable.

```
Client -> API Gateway -> Message Queue -> Worker Pool (GPU) -> Result Store -> Client polls/webhook
```

- **Tools:** Redis/RabbitMQ/SQS as queue, Celery/custom workers as consumers
- **When it fits:** Batch inference, image/video generation, training jobs -- anything where sub-second response isn't expected
- **Client integration:** Return a job ID immediately, client polls a status endpoint or registers a webhook

### Caching Layer

Cache model responses for identical (or semantically similar) inputs.

- **Exact match:** Hash the input, cache the output. Simple and effective for classification or embedding endpoints
- **Semantic cache:** Use an embedding similarity threshold to return cached results for "close enough" inputs. GPTCache does this
- **Where it breaks:** Generative models with temperature > 0, or where freshness matters (RAG with live data)

---

## Reference Architecture

```
                    +----------------+
                    |    Clients     |
                    +-------+--------+
                            |
                    +-------v--------+
                    |   API Gateway  |  Auth, rate limit, token metering
                    +-------+--------+
                            |
                    +-------v--------+
                    |  Model Router  |  Route by model/complexity/cost
                    +-------+--------+
                       +----+----+
                       v    v    v
                    +----+----+------+
                    |vLLM| TGI|Triton|  Model serving tier
                    +--+-+--+-+--+---+
                       |    |    |
                    +--v----v----v----+
                    |    GPU Pool     |  MIG / vGPU / passthrough
                    |  (k8s + GPU Op) |
                    +-----------------+
```
