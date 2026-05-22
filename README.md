# Service Mesh in Production — Istio + Envoy on Kubernetes

> A production-oriented Istio Service Mesh implementation, covering xDS protocol internals, Pilot-Envoy interaction, traffic governance, and full-stack observability — with every design decision documented.

[![Istio](https://img.shields.io/badge/Istio-1.17-466BB0?logo=istio)](https://istio.io)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.26-326CE5?logo=kubernetes)](https://kubernetes.io)
[![Envoy](https://img.shields.io/badge/Envoy-Proxy-AC6199)](https://www.envoyproxy.io)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## What Problem Does This Solve

In a microservices architecture, inter-service communication management is a severely underestimated engineering challenge:

- Every service needs circuit breaking, retries, and timeouts — **how do you implement all of this without polluting application code with infrastructure logic?**
- Traffic governance rules (canary releases, fault injection) need to change at runtime — **how do you update routing rules without redeploying or restarting any service?**
- When a request fans out across 30 services and returns a 500, **how do you pinpoint exactly which hop in the chain caused the failure?**
- Certificate rotation across hundreds of sidecars used to require hot-restarts — **how do you push new TLS certificates to every proxy with zero downtime?**

This project addresses these challenges systematically using **Istio + Envoy**, with a focus on understanding the underlying mechanics — not just applying YAML.

✅ xDS protocol (LDS/RDS/CDS/EDS/SDS) — all config is dynamic, zero restarts required  
✅ DestinationRule `connectionPool` + `outlierDetection` — rate limiting and circuit breaking at the proxy layer  
✅ Prometheus + Grafana — golden signals (QPS, latency, error rate) with inbound/outbound split  
✅ Jaeger distributed tracing — full request waterfall across all microservices  
✅ mTLS + AuthorizationPolicy — zero-trust service-to-service security  

---

## Three Core Engineering Decisions

### Decision 1: xDS Protocol — Why All Config Is Dynamic

**Problem:** Traditional proxy configuration is file-based. Any change to routing rules, upstream endpoints, or TLS certificates requires a config reload or process restart — unacceptable in a dynamic Kubernetes environment where pods come and go constantly.

**Decision:** Adopt the xDS protocol (gRPC bidirectional streaming) as the single config delivery channel between Istio Pilot and every Envoy sidecar:

| xDS API | Resource | Dynamic Capability |
|---------|----------|--------------------|
| LDS | Listener | Add/update listening ports without restart |
| RDS | Route | Update routing rules and virtual hosts at runtime |
| CDS | Cluster | Add/remove upstream services dynamically |
| EDS | Endpoint | Track live pod IPs as they scale up/down |
| SDS | Secret | Rotate TLS certificates with zero downtime |

The ACK/NACK mechanism ensures every Envoy instance confirms successful config application. The Nonce field eliminates ambiguity between concurrent updates and acknowledgements.

**Outcome:** Config changes propagate to all sidecars in milliseconds. A canary release is a `kubectl apply` on a VirtualService — no service restarts, no deployment rollouts.

---

### Decision 2: DestinationRule TrafficPolicy — Resilience Without Code Changes

**Problem:** Implementing circuit breaking and rate limiting in application code means every service team must integrate a resilience library (Hystrix, Resilience4j, etc.), maintain it, and configure it consistently. In practice, this leads to inconsistent behavior and configuration drift across services.

**Decision:** Delegate all resilience logic to the Envoy sidecar via `DestinationRule.trafficPolicy`:

```yaml
trafficPolicy:
  connectionPool:
    tcp:
      maxConnections: 100        # Hard cap on TCP connections
    http:
      http1MaxPendingRequests: 100
      http2MaxRequests: 1000
  outlierDetection:
    consecutiveErrors: 5         # Eject after 5 consecutive 5xx errors
    interval: 10s
    baseEjectionTime: 30s        # Ejection time grows with each failure: 30s × N
    maxEjectionPercent: 10       # Never eject more than 10% of instances
    minHealthPercent: 50         # Disable CB if cluster health drops below 50%
```

Two safety nets prevent cascade failures:
- `maxEjectionPercent` caps how many instances can be ejected simultaneously
- `minHealthPercent` disables circuit breaking entirely when the cluster is already critically degraded — ejecting more instances would only make a systemic failure worse

**Outcome:** Circuit breaking and rate limiting are enforced uniformly across all services. Application teams write zero resilience code. Behavior is auditable via Git history on the DestinationRule manifest.

---

### Decision 3: Observability Stack — From "Something Is Wrong" to "Here's Exactly Why"

**Problem:** In a microservices system, a single slow API call may involve 10+ services. Metrics alone tell you *that* something is wrong. Logs tell you *what* happened in one service. Neither tells you *where in the call chain* the problem originated.

**Decision:** Deploy a three-layer observability stack where each layer answers a different question:

```
Prometheus + Grafana  →  "reviews-v3 p99 latency spiked to 2s at 14:30"
       ↓
Jaeger Tracing        →  "The spike is caused by ratings service at hop 4"
       ↓
ELK Logging           →  "Here's the exact DB query that timed out"
```

For distributed tracing, Envoy automatically generates spans and injects B3 headers into upstream requests. Application code only needs to **forward** the headers it receives — not generate them:

```python
def handle_request(request):
    headers = extract_trace_headers(request)  # forward, don't generate
    response = call_upstream(headers)
    return response
```

Grafana dashboards expose both **inbound** and **outbound** traffic dimensions per workload — critical for distinguishing whether a service is slow because of upstream pressure or a degraded downstream dependency.

**Outcome:** Mean time to root cause (MTTRC) drops from hours to minutes. Service dependency graphs are auto-generated from trace data — no manual diagram maintenance.

---

## System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         API Server                               │
│            (Services + Endpoints + Istio CRDs)                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │ client-go Informer
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                        Istio Pilot                               │
│   Service Discovery → Policy Translation → xDS Push             │
└────────────────────────────┬─────────────────────────────────────┘
                             │ gRPC streaming (xDS)
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Envoy Sidecar                              │
│   Listener ──Route──▶ Cluster ──LB──▶ Endpoint                  │
│   [Listener Filter] → [Network Filter] → [HTTP Filter]          │
└──────────────────────────────────────────────────────────────────┘
                             │
             ┌───────────────┼───────────────┐
             ▼               ▼               ▼
        Prometheus         Jaeger          Upstream
        (Metrics)         (Traces)        Services
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Service Mesh | Istio 1.17 |
| Data Plane Proxy | Envoy (sidecar injection) |
| Orchestration | Kubernetes 1.26 |
| Config Protocol | xDS (gRPC streaming) |
| Metrics | Prometheus + Grafana |
| Distributed Tracing | Jaeger |
| Service Visualization | Kiali |
| Security | mTLS + AuthorizationPolicy |

---

## Repository Structure

```
servicemesh/
├── traffic-management/
│   ├── virtual-service/        # Canary releases, traffic splitting, fault injection
│   ├── destination-rule/       # Circuit breaking, connection pool, load balancing
│   └── gateway/                # Ingress/Egress gateway configuration
├── security/
│   ├── mtls/                   # Mutual TLS (strict / permissive mode)
│   └── authorization-policy/   # Service-to-service RBAC
├── observability/
│   ├── prometheus/             # Scrape config & alerting rules
│   ├── grafana/                # Dashboard provisioning
│   └── jaeger/                 # Tracing setup & sampling config
└── bookinfo/                   # Sample app used in all demos
```

---

## Quick Start

```bash
# 1. Install Istio
istioctl install --set profile=demo -y

# 2. Enable sidecar injection
kubectl label namespace default istio-injection=enabled

# 3. Deploy sample app
kubectl apply -f bookinfo/

# 4. Apply traffic governance rules
kubectl apply -f traffic-management/destination-rule/

# 5. Open observability dashboards
istioctl dashboard grafana
istioctl dashboard jaeger
```

---

## Related Articles

Deep-dive articles documenting the internals behind this implementation:

| Article | Key Topics |
|---------|-----------|
| [xDS Protocol Deep Dive](#) | LDS/RDS/CDS/EDS/SDS data structures, gRPC streaming, ACK/NACK, Nonce |
| [Istio & Envoy Architecture](#) | Pilot-Envoy interaction, 4 core Envoy resources, Filter layer design |
| [Circuit Breaking & Rate Limiting](#) | connectionPool, outlierDetection, progressive ejection, safety nets |
| [Full Observability: Metrics + Tracing](#) | Prometheus data model, Grafana inbound/outbound, Jaeger trace propagation |
