---
date: 2026-04-24
tags: [kubernetes, services, networking, discovery, kube-proxy]
series: kubernetes
doc: 7
tier: 3
---

# Kubernetes Services — ClusterIP, NodePort, LoadBalancer, and Service Discovery

**Audience:** Backend developers (TypeScript/Node, Java/Spring Boot) who deploy to Kubernetes and need to understand how pod-to-pod and external-to-pod routing actually works beneath `kubectl expose`.

**Depends on:** [Pods, ReplicaSets & Deployments](../workloads/pods-and-deployments.md), [Namespaces, Labels & Selectors](../core-concepts/namespaces-and-labels.md)

---

## Table of Contents

1. [The Service Abstraction](#the-service-abstraction)
2. [Service Types](#service-types)
3. [How Service Routing Works](#how-service-routing-works)
4. [EndpointSlices](#endpointslices)
5. [kube-proxy Modes](#kube-proxy-modes)
6. [Headless Services](#headless-services)
7. [Session Affinity](#session-affinity)
8. [Service DNS](#service-dns)
9. [Traffic Policies](#traffic-policies)
10. [Services Without Selectors](#services-without-selectors)
11. [Practical Debugging](#practical-debugging)
12. [Summary](#summary)
13. [Related](#related)
14. [References](#references)

---

## The Service Abstraction

Pods are ephemeral. A Deployment's ReplicaSet creates replacement pods with new IPs every time one dies. You cannot hard-code pod IPs in a Spring Boot `application.yml` or a Node `fetch()` call — they are not stable.

A **Service** gives you two things no individual pod can:

1. **A stable virtual IP (ClusterIP)** that survives pod restarts and reschedules.
2. **A DNS name** (`<svc>.<ns>.svc.cluster.local`) that resolves to that VIP — or, for headless Services, directly to pod IPs.

The Service selects its backend pods through **labels**. Any pod whose labels match the Service's `spec.selector` becomes an endpoint, automatically.

```
                   +-----------+
   DNS lookup      |  Service  |   stable ClusterIP: 10.96.44.12
   my-api.prod --> |  (VIP)    |   port: 80 --> targetPort: 8080
                   +-----+-----+
                         |
             +-----------+-----------+
             |           |           |
          Pod-A       Pod-B       Pod-C
        10.244.1.5  10.244.2.8  10.244.3.2
        app=my-api  app=my-api  app=my-api
```

---

## Service Types

### ClusterIP (Default)

Internal-only. Reachable from within the cluster. This is what your Spring Boot microservice uses to call another microservice.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: prod
spec:
  type: ClusterIP          # default — can be omitted
  selector:
    app: user-service
  ports:
    - name: http
      port: 80             # port the Service listens on
      targetPort: 8080     # port the container exposes
      protocol: TCP
```

From any pod in the cluster:

```bash
curl http://user-service.prod.svc.cluster.local/api/users
# or just:
curl http://user-service.prod/api/users     # search domain fills the rest
```

### NodePort

Exposes the Service on a static port (range **30000-32767**) on **every node's** IP. Traffic arriving at `<NodeIP>:<NodePort>` is forwarded to the ClusterIP, then to a backend pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: NodePort
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080      # optional — K8s assigns one if omitted
```

Useful for dev/test or bare-metal clusters. In production, you almost always use LoadBalancer or an Ingress controller instead.

### LoadBalancer

Extends NodePort. The cloud-controller-manager provisions an external L4 load balancer (AWS NLB/CLB, GCP TCP LB, Azure LB) that routes into the NodePort.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  annotations:
    # AWS-specific: request an NLB
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
spec:
  type: LoadBalancer
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 8080
```

After reconciliation, `status.loadBalancer.ingress[0].hostname` (AWS) or `.ip` (GCP/Azure) gives you the external address.

### ExternalName

A CNAME alias. No proxying, no ClusterIP, no endpoints — pure DNS rewrite.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: legacy-db
spec:
  type: ExternalName
  externalName: db-primary.us-east-1.rds.amazonaws.com
```

Pods calling `legacy-db.prod.svc.cluster.local` get a CNAME pointing to the RDS hostname. Useful for migrating an external dependency behind a stable in-cluster name.

> **Caveat:** ExternalName does not support ports and can break HTTP Host headers. Prefer a ClusterIP Service without selectors (see below) if you need port mapping.

---

## How Service Routing Works

```mermaid
flowchart LR
    subgraph Control Plane
        EP[EndpointSlice Controller]
    end

    subgraph Data Plane — each Node
        KP[kube-proxy]
        IPT[iptables / IPVS / nftables rules]
    end

    SVC[Service] -->|label selector| EP
    EP -->|creates/updates| ES[EndpointSlice objects]
    KP -->|watches| ES
    KP -->|programs| IPT

    subgraph Pod Traffic
        CLIENT[Client Pod] -->|dst: ClusterIP:port| IPT
        IPT -->|DNAT to pod IP:targetPort| BACKEND[Backend Pod]
    end
```

The flow in detail:

1. You create a Service with a `selector`.
2. The **EndpointSlice controller** in the control plane watches pods matching that selector and creates **EndpointSlice** objects listing their IPs, ports, and readiness.
3. **kube-proxy** on every node watches EndpointSlice objects and programs kernel-level routing rules (iptables, IPVS, or nftables).
4. When a client pod sends traffic to `ClusterIP:port`, the kernel rules **DNAT** (destination NAT) the packet to one of the backend pod IPs.

No userspace proxying involved in the default path — it is all kernel-level packet rewriting.

---

## EndpointSlices

EndpointSlices replaced the legacy `Endpoints` resource for all practical purposes. As of **Kubernetes 1.33**, the `Endpoints` API is officially **deprecated**.

### Why EndpointSlices Scale Better

The legacy Endpoints object was **one object per Service**, containing every pod IP. A Service backing 5,000 pods meant a single Endpoints object with 5,000 entries. Every time one pod changed readiness, the **entire** object had to be updated and distributed to every node.

EndpointSlices break that list into chunks:

| Property | Endpoints (legacy) | EndpointSlices |
|---|---|---|
| Objects per Service | 1 | Many (one per ~100 endpoints) |
| Max endpoints per object | Unbounded | **100** (default, configurable to 1000) |
| Update granularity | Full object replace | Single slice update |
| Dual-stack support | No | Yes |
| New features (traffic distribution) | Not supported | Supported |

When pod-42 becomes unready, only the single EndpointSlice containing pod-42 is updated. The other 49 slices stay untouched — reducing etcd writes, API server load, and kube-proxy work.

```yaml
# Auto-generated by the control plane — you rarely create these manually
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: user-service-abc12
  namespace: prod
  labels:
    kubernetes.io/service-name: user-service
addressType: IPv4
endpoints:
  - addresses: ["10.244.1.5"]
    conditions:
      ready: true
    nodeName: worker-1
  - addresses: ["10.244.2.8"]
    conditions:
      ready: true
    nodeName: worker-2
ports:
  - name: http
    port: 8080
    protocol: TCP
```

---

## kube-proxy Modes

kube-proxy translates EndpointSlice data into kernel-level packet routing rules. It runs on every node as a DaemonSet (or static pod) and operates in one of three modes.

### iptables Mode (Default)

- Programs **iptables** rules (NAT table) for each Service/endpoint combination.
- Backend selection: **random**, using iptables probability rules.
- Well-supported, mature, works everywhere.
- Scales poorly past ~10,000 Services because rule evaluation is **linear** — every packet walks the chain.

### IPVS Mode

- Uses the Linux **IPVS** (IP Virtual Server) kernel module — a purpose-built L4 load balancer.
- Scheduling algorithms: **round-robin**, **least-connection**, **weighted round-robin**, **consistent hashing**, and more.
- **O(1)** connection routing via hash tables — far better for large clusters.
- Still depends on iptables for some edge-case rules (masquerade, packet filtering).

```bash
# Enable IPVS mode in kube-proxy ConfigMap
kubectl edit configmap kube-proxy -n kube-system
# set mode: "ipvs"
# then restart kube-proxy pods
```

### nftables Mode (GA in 1.33)

- The modern replacement for both iptables and IPVS modes.
- Uses the **nftables** kernel subsystem — successor to iptables in the Linux kernel.
- Processes endpoint changes and kernel packet routing **more efficiently** than iptables.
- Performance advantage becomes significant at tens of thousands of Services.
- The Kubernetes project recommends nftables as the long-term replacement for IPVS users.
- Requires a recent Linux kernel (5.13+). May not be compatible with all CNI plugins yet.

```bash
# Enable nftables mode
kubectl edit configmap kube-proxy -n kube-system
# set mode: "nftables"
```

**Choosing a mode:**

| Scenario | Recommended mode |
|---|---|
| Small–medium clusters (<5,000 Services) | iptables (default) |
| Large clusters, need scheduling algorithms | IPVS or nftables |
| New clusters on recent kernels | nftables |
| Legacy kernels or constrained CNI compatibility | iptables |

---

## Headless Services

Set `clusterIP: None` and the Service gets no VIP. Instead, DNS returns the **individual pod IPs** as A/AAAA records.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cassandra
spec:
  clusterIP: None          # headless
  selector:
    app: cassandra
  ports:
    - port: 9042
      targetPort: 9042
```

### Why Headless Matters

StatefulSets rely on headless Services for **stable per-pod DNS**. Each pod gets a DNS record:

```
cassandra-0.cassandra.prod.svc.cluster.local → 10.244.1.5
cassandra-1.cassandra.prod.svc.cluster.local → 10.244.2.8
cassandra-2.cassandra.prod.svc.cluster.local → 10.244.3.2
```

This is how Cassandra nodes discover peers, Kafka brokers advertise themselves, and ZooKeeper forms quorums. In the Java world, Spring Boot's Cassandra driver or Kafka client can be configured with the headless Service name and will resolve all pod IPs for direct connections.

See [StatefulSets and DaemonSets](../workloads/statefulsets-and-daemonsets.md) for the full StatefulSet + headless Service pattern.

---

## Session Affinity

By default, kube-proxy distributes connections across backends without stickiness. If your app stores session state in memory (express-session without Redis, or a Java HttpSession without Spring Session), you can pin clients to the same pod:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web-app
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800    # 3 hours (default)
  ports:
    - port: 80
      targetPort: 3000
```

kube-proxy tracks the client source IP and routes subsequent connections to the same backend for the configured timeout.

> **Better pattern:** Store session state externally (Redis, database) and keep your pods stateless. Session affinity breaks graceful scaling and rolling updates.

---

## Service DNS

CoreDNS (the cluster DNS addon) creates DNS records for every Service.

### ClusterIP Services

```
user-service.prod.svc.cluster.local  →  A  10.96.44.12  (the ClusterIP)
```

### Headless Services

```
cassandra.prod.svc.cluster.local     →  A  10.244.1.5
                                         A  10.244.2.8
                                         A  10.244.3.2
```

Each pod also gets an individual record:

```
cassandra-0.cassandra.prod.svc.cluster.local  →  A  10.244.1.5
```

### SRV Records

For named ports, DNS provides SRV records:

```
_http._tcp.user-service.prod.svc.cluster.local  →  SRV  0 33 8080 user-service.prod.svc.cluster.local
```

### DNS Search Domains

Pod `/etc/resolv.conf` has (by default):

```
search prod.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

This means `curl user-service` from a pod in the `prod` namespace resolves by appending `prod.svc.cluster.local`. Cross-namespace calls need at least `user-service.other-ns`.

> **Performance note:** `ndots:5` means any hostname with fewer than 5 dots triggers search-domain expansion — so `api.example.com` causes 4 DNS queries before the final one succeeds. For pods making heavy external DNS calls, consider overriding `dnsConfig.options` to lower ndots or add the FQDN with a trailing dot (`api.example.com.`).

---

## Traffic Policies

Two independent knobs control where traffic lands.

### externalTrafficPolicy (NodePort / LoadBalancer only)

Controls routing for traffic entering through the external load balancer or NodePort.

**`Cluster` (default):**

- Traffic arriving at any node is forwarded to any backend pod in the cluster.
- Even distribution across pods.
- **Client source IP is lost** — replaced by the forwarding node's IP (SNAT).
- Extra network hop if the receiving node has no local pod.

**`Local`:**

- Traffic is delivered only to pods on the **same node** that received it.
- **Preserves client source IP** — no SNAT.
- If the node has no local endpoints, the health check fails and the LB stops sending traffic to it.
- Can lead to **uneven distribution** if pods are not spread evenly across nodes.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # preserve client IP
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 8080
```

Use `Local` when you need source IP for rate limiting, geo-routing, or access logs. Use `Cluster` when even distribution matters more.

### internalTrafficPolicy

Controls routing for cluster-internal traffic (pod-to-Service calls).

**`Cluster` (default):**

- Routes to any ready endpoint in the cluster.

**`Local`:**

- Routes only to endpoints on the **same node** as the caller.
- If no local endpoints exist, the traffic is **dropped** (the Service behaves as if it has zero endpoints for that node).
- Useful for node-local DaemonSet services (log collectors, monitoring agents) to avoid cross-node hops.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
spec:
  internalTrafficPolicy: Local   # only route to local DaemonSet pod
  selector:
    app: node-exporter
  ports:
    - port: 9100
      targetPort: 9100
```

---

## Services Without Selectors

Sometimes you need a Service that points to something outside Kubernetes — an external database, a legacy VM, or a service in another cluster. Omit the selector and create EndpointSlice objects manually.

```yaml
# Service (no selector)
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: prod
spec:
  ports:
    - port: 5432
      targetPort: 5432
---
# Manual EndpointSlice
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-db-1
  namespace: prod
  labels:
    kubernetes.io/service-name: external-db   # must match
addressType: IPv4
endpoints:
  - addresses: ["192.168.1.100"]
ports:
  - port: 5432
    protocol: TCP
```

Now pods call `external-db.prod.svc.cluster.local:5432` and traffic routes to `192.168.1.100`. When you later migrate the database into the cluster, swap the Service to use a selector — no application config changes needed.

---

## Practical Debugging

### port-forward for Quick Access

Forward a local port to a Service (or directly to a pod) without exposing anything externally:

```bash
# Forward local 8080 to Service port 80
kubectl port-forward svc/user-service 8080:80 -n prod

# Forward to a specific pod
kubectl port-forward pod/user-service-7d9b5c6f4-abc12 8080:8080 -n prod
```

Then `curl http://localhost:8080/api/users` from your dev machine.

### Verify Service and Endpoints

```bash
# Describe the Service — check selector, type, ClusterIP, ports
kubectl describe svc user-service -n prod

# List EndpointSlices — are pods registered?
kubectl get endpointslices -l kubernetes.io/service-name=user-service -n prod

# Check that pods match the selector
kubectl get pods -l app=user-service -n prod -o wide

# DNS resolution from inside the cluster
kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- \
  nslookup user-service.prod.svc.cluster.local
```

### Common Failure Modes

| Symptom | Likely cause | Check |
|---|---|---|
| Service has no endpoints | Selector does not match any pod labels | `kubectl get ep <svc>` and compare selector with pod labels |
| Connection refused | targetPort does not match container port | `kubectl describe svc` + `kubectl describe pod` |
| DNS not resolving | CoreDNS pods down, or wrong namespace in URL | `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| External LB not provisioned | cloud-controller-manager not running or missing IAM permissions | `kubectl describe svc` events section |
| Traffic drops with `internalTrafficPolicy: Local` | No local endpoints on the calling node | Check pod topology vs node placement |

### kube-proxy Rule Inspection

```bash
# iptables mode: list Service NAT rules
iptables-save | grep user-service

# IPVS mode: show virtual servers
ipvsadm -Ln

# nftables mode: list rules
nft list ruleset | grep user-service
```

---

## Summary

| Concept | Key takeaway |
|---|---|
| **Service** | Stable VIP + DNS for a set of label-selected pods |
| **ClusterIP** | Internal only — the default for microservice-to-microservice calls |
| **NodePort** | Expose on `<NodeIP>:30000-32767` — dev/test or bare metal |
| **LoadBalancer** | Cloud LB provisioned automatically — the standard for external exposure |
| **ExternalName** | DNS CNAME alias — no proxying, no endpoints |
| **EndpointSlice** | Scalable replacement for Endpoints — 100 entries per slice, fine-grained updates |
| **kube-proxy** | iptables (default), IPVS (large clusters), nftables (modern replacement for both) |
| **Headless Service** | `clusterIP: None` — DNS returns pod IPs, used by StatefulSets |
| **externalTrafficPolicy: Local** | Preserves client source IP, avoids extra hop |
| **internalTrafficPolicy: Local** | Node-local routing for DaemonSet services |

---

## Related

- [Container Networking](../../networking/advanced/container-networking.md) — covers the same Service types from the network-plumbing perspective (CNI, veth pairs, overlay networks)
- [Load Balancing](../../networking/infrastructure/load-balancing.md) — L4/L7 load balancing concepts that underpin LoadBalancer Services and Ingress
- [StatefulSets and DaemonSets](../workloads/statefulsets-and-daemonsets.md) — headless Services for StatefulSet peer discovery
- [Ingress and Gateway API](./ingress-and-gateway-api.md) — L7 routing that builds on top of Services
- [Network Policies and CoreDNS](./network-policies-and-dns.md) — DNS resolution details and network segmentation

---

## References

1. [Services — Kubernetes Docs](https://kubernetes.io/docs/concepts/services-networking/service/)
2. [EndpointSlices — Kubernetes Docs](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
3. [Virtual IPs and Service Proxies — Kubernetes Docs](https://kubernetes.io/docs/reference/networking/virtual-ips/)
4. [Service Internal Traffic Policy — Kubernetes Docs](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)
5. [nftables mode for kube-proxy — Kubernetes Blog](https://kubernetes.io/blog/2025/02/28/nftables-kube-proxy/)
6. [Endpoints Deprecation in v1.33 — Kubernetes Blog](https://kubernetes.io/blog/2025/04/24/endpoints-deprecation/)
7. [DNS for Services and Pods — Kubernetes Docs](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
8. [Scaling Kubernetes Networking with EndpointSlices — Kubernetes Blog](https://kubernetes.io/blog/2020/09/02/scaling-kubernetes-networking-with-endpointslices/)
