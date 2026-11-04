# Waypoint

**A distributed LLM serving platform built from scratch in Python.**

Waypoint serves small open-source language models to many users at once. It has two layers:

1. **An inference engine** with continuous batching, a paged KV cache, and prefix caching, the core techniques behind production systems like vLLM.
2. **A cluster layer**: an API gateway that handles incoming traffic, and a cache-aware router that load-balances requests across engine workers and reroutes around failures.

> Every request passes through Waypoint on its way to the worker best suited to answer it.

🚧 **Status:** in early development.

---

## Architecture

```
                         ┌──────────────────────────────┐
  Clients ── HTTP/SSE ─▶ │  Gateway                     │   north-south traffic
                         │  auth · rate limits · cache  │
                         └──────────────┬───────────────┘
                                        │
                         ┌──────────────▼───────────────┐
                         │  Router                      │
                         │  cache-aware load balancing  │
                         │  health checks · failover    │
                         └──┬───────────┬────────────┬──┘
                            │           │            │      east-west traffic
                     ┌──────▼───┐ ┌─────▼────┐ ┌─────▼────┐
                     │ Worker 1 │ │ Worker 2 │ │ Worker 3 │
                     │ engine   │ │ engine   │ │ engine   │
                     └──────────┘ └──────────┘ └──────────┘
                     each worker: scheduler → continuous batching
                                  → paged KV cache → model (PyTorch)
```

### Lifecycle of a request

1. The **gateway** authenticates the API key, applies per-key rate limits, and checks the response cache.
2. The **router** picks a worker, preferring one that already has the prompt's prefix cached, unless it's overloaded.
3. The worker's **scheduler** adds the request to the running batch on the next model step.
4. The **engine** stores attention state in a block-based **KV cache**, reusing blocks shared with other requests.
5. Tokens **stream** back to the client as they're generated.
6. If a worker fails, the router detects it and **retries** the request on a healthy worker.

## Features

**Inference engine**
- Custom generation loop with greedy, temperature, top-k, and top-p sampling
- Custom forward pass for Llama-style models, loading pretrained weights
- Continuous (iteration-level) batching: requests join and leave the batch every step
- Paged KV cache with a block allocator, block tables, admission control, and preemption
- Prefix caching with chained block hashing, reference counting, and LRU eviction

**Cluster**
- Routing policies: round-robin, least-outstanding-requests, and cache-aware
- Health checks, retries, and mid-stream failover
- Token-bucket rate limiting per API key
- Exact-match response cache and request coalescing
- Deployment on Kubernetes (k3d) with Prometheus metrics and Grafana dashboards

## Tech stack

| Layer | Tools |
|---|---|
| Language | Python 3.12 |
| Model | PyTorch (Apple `mps` backend), Hugging Face `transformers` for weights and tokenizer |
| API | FastAPI, `asyncio`, Server-Sent Events |
| Packaging | `uv`, Docker |
| Orchestration | Kubernetes (k3d), kubectl |
| Observability | Prometheus, Grafana |
| Testing | `pytest`, load testing with Locust / k6 |

## Benchmarks

Results will be published here as each component is completed, including:

- Throughput (tokens/s): sequential vs. static vs. continuous batching
- Time to first token (TTFT) and time per output token (TPOT)
- p50 / p95 / p99 latency under increasing load
- KV cache memory utilization: paged vs. naive preallocation
- Prefix cache hit rate and its effect on TTFT
- Cache-aware vs. round-robin routing
- Recovery time when a worker is killed under load

## Repository layout

```
engine/     inference engine: model, sampler, scheduler, KV cache, step loop
worker/     HTTP server wrapping one engine
router/     cache-aware load balancer
gateway/    auth, rate limiting, response cache
bench/      load tests and benchmark scripts
deploy/     Dockerfiles, Kubernetes manifests, dashboards
docs/       design doc and results
tests/      unit and integration tests
```

## References

- Kwon et al., *Efficient Memory Management for Large Language Model Serving with PagedAttention* (SOSP 2023)
- Yu et al., *Orca: A Distributed Serving System for Transformer-Based Generative Models* (OSDI 2022)
