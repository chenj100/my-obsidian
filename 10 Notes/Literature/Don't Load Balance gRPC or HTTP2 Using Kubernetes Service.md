---
title: "Don't Load Balance gRPC or HTTP2 Using Kubernetes Service"
author: Junrui Chen
source: https://medium.com/@lapwingcloud/dont-load-balance-grpc-or-http2-using-kubernetes-service-ae71be026d7f
published: 2024-03-03
added: 2026-04-13
tags:
  - type/literature
  - status/evergreen
  - domain/engineering
---

## Summary

Kubernetes `Service` (ClusterIP) is an L4 proxy — it only sees TCP connections, not HTTP2/gRPC streams. Because gRPC/HTTP2 clients reuse a single persistent TCP connection, each client pod ends up pinned to one server pod. No real load balancing occurs.

## The Core Problem

```
Client Pod → ClusterIP (10.0.0.20) → single Server Pod (always the same one)
```

- gRPC/HTTP2 opens **1 TCP connection** and multiplexes all requests over it
- Kubernetes Service picks a backend pod at **connection time**, not request time
- Result: if you have 3 client pods and 10 server pods, only 3 server pods ever receive traffic

**Root cause:** L4 proxies are connection-aware, not request-aware.

## Solutions

### 1. Headless Service + Client-Side Load Balancing
DNS resolves to all pod IPs directly (no ClusterIP). Client implements round-robin.

```go
conn, err := grpc.Dial(
    "dns:///grpc-server.namespace.svc.cluster.local:9090",
    grpc.WithInsecure(),
    grpc.WithBalancerName(roundrobin.Name),
)
```

- Use `dns:///` scheme — `passthrough` (default) won't re-resolve DNS
- Kubernetes removes unhealthy pod IPs from DNS automatically via readiness probes
- **Caveat:** DNS must not be cached; client code must be updated

### 2. Ingress NGINX (L7 proxy)
ingress-nginx subscribes to service endpoints directly and updates upstream config on changes. Supports gRPC natively since 2018.

```yaml
annotations:
  nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
```

- **Caveat 1:** Historically no gRPC over plaintext (h2c), may be fixed in nginx ≥ 1.25
- **Caveat 2:** `reuse-port` option causes worker-level independent round-robin, making distribution uneven when worker count is high (see [ingress-nginx#4054](https://github.com/kubernetes/ingress-nginx/issues/4054))

### 3. Service Mesh (e.g. Istio)
Sidecar proxy per pod acts as a local L7 proxy. Handles load balancing transparently.

- **Pro:** No client code changes; also gives tracing, circuit breaking, mTLS
- **Con 1:** Webhook injection — if webhook fails, pod creation fails
- **Con 2:** Sidecar termination ordering issues with Kubernetes Jobs (improving with native sidecar support)
- **Con 3:** Resource overhead per pod (mitigated by eBPF/sidecarless mesh)

## Decision Matrix

| Scenario | Recommended approach |
|---|---|
| Clients >> Servers (e.g. public TCP proxy) | L4 is fine |
| Internal service-to-service, want simplicity | Headless service + client-side LB |
| Internal service-to-service, want visibility/logs | Ingress NGINX |
| Complex cluster, need observability/security | Service mesh |

## Key Concepts

- [[L4 vs L7 proxy]] — L4 sees TCP connections; L7 sees HTTP requests
- [[gRPC connection multiplexing]] — one TCP connection carries many concurrent RPC streams
- [[Kubernetes headless service]] — `clusterIP: None`, DNS returns all pod IPs
- [[Kubernetes readiness probe]] — controls whether a pod IP appears in service DNS

## My Notes

> Author's own conclusion: still prefers Kubernetes Service or Ingress NGINX depending on scenario — service mesh overhead not worth it unless you need its other features.
