# PAX Router — Request Routing & Load Balancing

**Status:** Production | **Version:** 1.0.0 | **Author:** PAX Infrastructure Team  
**Domain:** 0-1.gg/pax/router

---

## What Is PAX Router?

PAX Router is the intelligent request multiplexing layer that distributes inference workloads across available PAX engines and backend serving frameworks. It provides dynamic load balancing, request queuing, and priority-based dispatch to maximize throughput and minimize latency across the entire PAX system.

```
Incoming Request (OpenAI format, custom PAX requests)
    ↓
PAX Router (smart dispatcher)
    ├─→ Health check & capacity assessment
    ├─→ Backend selection (vLLM, TensorRT, llama.cpp)
    ├─→ Load balancing (least-connections, response-time aware)
    └─→ Request queueing with priority levels
    ↓
[PAX_INFERENCE_CORE | PAX_MATH_SOLVER | PAX_SEMANTIC_ENGINE | PAX_VISION_ENGINE]
    ↓
Response stream back to client
```

---

## Key Specifications

| Aspect | Details |
|--------|---------|
| **Protocol** | HTTP/2, gRPC, WebSocket (SSE streaming) |
| **Max Concurrent Requests** | 1000+ (configurable per deployment) |
| **Request Queue Depth** | 10,000 pending (with priority levels) |
| **Latency (router overhead)** | <5ms (P95) for routing decision |
| **Load Balancing Strategies** | Least-connections, weighted round-robin, response-time aware, hash-based (sticky sessions) |
| **Health Check Interval** | 100ms (adaptive backoff on failure) |
| **Backend Support** | vLLM, TensorRT, llama.cpp, custom engines |
| **Request Priority Levels** | 5 tiers (critical, high, normal, low, batch) |
| **Failover** | Automatic redispatch on backend failure |

---

## Architecture

### Layer 1: Request Ingress
- Protocol translation (OpenAI API → internal PAX format)
- Request validation and schema checking
- Priority extraction (from headers or body)
- Rate limiting enforcement per client/tenant

### Layer 2: Load Balancer
- Real-time capacity monitoring (GPU memory, queue depth)
- Backend health tracking (latency, error rate, availability)
- Dynamic weight adjustment based on performance
- Request affinity for stateful operations

### Layer 3: Queue Management
- Per-priority queuing (5 tiers)
- Request timeout enforcement
- SLA tracking (P50, P95, P99 latency targets)
- Backpressure signaling to clients

### Layer 4: Backend Dispatch
- Connection pooling to backend services
- Request streaming for long-running operations
- Response aggregation and formatting
- Error handling and retry logic

---

## Performance Characteristics

### Throughput (Verified from PAX_RESULTS.md)
- **Single vLLM backend:** ~100 tok/sec on H100
- **Dual RTX 3090 cluster:** ~30 tok/sec sustained
- **Load-balanced across 3 backends:** ~2.8x throughput scaling (sublinear due to coordination overhead)

### Latency
- **Router decision latency:** <5ms P95
- **Queue wait (normal priority, 50% load):** <100ms P95
- **Queue wait (high load):** <2s P95
- **Total end-to-end (with inference):** Dominated by backend inference time

### Scaling Characteristics
- **Horizontal:** Add backends, router redistributes load automatically
- **Vertical:** Increase max concurrent requests with more router instances
- **Multi-region:** Federation mode routes across geographically distributed backends

---

## Quick Start

### Installation
```bash
pip install pax-router

# Or build from source
git clone https://github.com/0-1-gg/pax-router.git
cd pax-router
pip install -e .
```

### Configuration (YAML)
```yaml
router:
  port: 8001
  protocol: http2
  max_concurrent: 256
  queue_depth: 5000
  
load_balancer:
  strategy: "weighted_round_robin"  # or least_connections, response_time_aware
  health_check_interval_ms: 100
  backend_timeout_s: 300
  
backends:
  - name: "vllm-h100"
    url: "http://vllm-1:8000"
    weight: 2
    capacity: 100  # max concurrent on this backend
    
  - name: "tensorrt-h100"
    url: "http://tensorrt-1:8002"
    weight: 1.5
    capacity: 50
    
  - name: "llamafile-cpu"
    url: "http://cpu-cluster:8003"
    weight: 0.5
    capacity: 20

priority_levels:
  critical:
    timeout_s: 600
    queue_max: 100
  high:
    timeout_s: 300
    queue_max: 500
  normal:
    timeout_s: 300
    queue_max: 2000
  low:
    timeout_s: 600
    queue_max: 1000
  batch:
    timeout_s: 3600
    queue_max: 1000
```

### Python API
```python
from pax_router import Router

# Initialize with config
router = Router.from_config("config.yaml")

# Start server
router.start(port=8001)

# Or use programmatically
response = router.route_request(
    model="pax-one-l5-narrow-27b",
    messages=[{"role": "user", "content": "What is 42 * 37?"}],
    priority="normal"
)
print(response)
```

### Docker Deployment
```bash
docker run -p 8001:8001 \
  -v $(pwd)/config.yaml:/etc/pax/config.yaml \
  pax-router:latest \
  --config /etc/pax/config.yaml
```

---

## Integration Points

### Primary Consumers
- **PAX_INFERENCE_CORE** — Dispatches requests across multiple inference backends
- **PAX_API_GATEWAY** — Router sits between gateway and backend engines
- **PAX_MONITOR_SYSTEM** — Publishes metrics (queue depth, latency, throughput)
- **PAX_SCHEDULER** — Coordinates with priority-based dispatch

### Complementary Systems
- **ANTICLOUD_AGENT** (Tier 1) — Submits coding tasks via router
- **api-oss-gateway** (Tier 3) — Enterprise wrapper around router
- **PAX_CACHE** — Router passes requests through cache layer
- **PAX_SEMANTIC_ENGINE** — Route requests for symbolic reasoning

### Deployment (Tier 3)
- **api-oss-monitor** — Health and performance monitoring
- **PAX_SCHEDULER** — SLA enforcement
- **Kubernetes** — Horizontal scaling of router instances
- **SOVEREIGN_OS** — Air-gapped router deployment

---

## Configuration Reference

### Environment Variables
```bash
PAX_ROUTER_PORT=8001
PAX_ROUTER_WORKERS=8
PAX_ROUTER_LOG_LEVEL=info
PAX_ROUTER_METRICS_PORT=9091
```

### Health Check Endpoint
```
GET /health
Response: {"status": "ok", "uptime_s": 3600, "backends": [...]}
```

### Metrics Endpoint (Prometheus)
```
GET /metrics
- pax_router_requests_total
- pax_router_request_latency_seconds
- pax_router_queue_depth
- pax_router_backend_health_status
- pax_router_priority_distribution
```

---

## Security & Compliance

- **Input validation:** Schema checking, injection prevention
- **Rate limiting:** Per-client, per-priority quotas
- **Request tracing:** Full trace ID propagation
- **Authentication:** JWT/API key integration with api-oss-gateway
- **Audit logging:** All routing decisions logged via AIOSS
- **Privacy:** Request content isolation, no cross-client data leakage

---

## Roadmap

- **Q4 2026:** Predictive load balancing (request size estimation)
- **Q1 2027:** Multi-region federation protocol
- **Q2 2027:** GPU-aware scheduling (tensor-parallel coordination)
- **Q3 2027:** Speculative routing (concurrent backend probing)

---

## References

- **Source:** 0-1.gg/pax/router
- **GitHub:** github.com/0-1-gg/pax-router
- **Architecture:** TIER_2_MASTER_INDEX.md
- **Integration:** ANTICLOUD_TIER_CROSS_REFERENCE.md

---

**Next:** See APPENDIX/ for complementary projects and integration patterns
