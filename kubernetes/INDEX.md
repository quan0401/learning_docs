# Kubernetes Documentation Index — Learning Path

A progressive path from cluster architecture through workloads, networking, and security to production operations. Practical and backend-oriented — you already know how to deploy Spring Boot and Node.js apps into K8s (see the Java and TypeScript paths); this path gives you the platform-level understanding of Kubernetes itself.

Cross-references to the [TypeScript learning path](../typescript/INDEX.md), the [Java learning path](../java/INDEX.md), the [Database learning path](../database/INDEX.md), the [Networking learning path](../networking/INDEX.md), the [Low-Level Design learning path](../low-level-design/INDEX.md), the [System Design learning path](../system-design/INDEX.md), the [Operating Systems learning path](../operating-systems/INDEX.md) (cgroups, namespaces, schedulers — what K8s actually orchestrates), the [Observability learning path](../observability/INDEX.md) (metrics/logs/traces in cluster), the [Performance Engineering learning path](../performance/INDEX.md), and the [Security learning path](../security/INDEX.md) where topics overlap.

**Markers:** **★** = core must-learn (everyday cluster work, common in interviews and production debugging). **○** = supporting deep-dive (specialized ops, security hardening, or advanced production topics). Internalize all ★ before going deep on ○.

---

## Tier 1 — Architecture & Core Concepts

The mental model everything else builds on. What runs where, how the control loop works, and how the API ties it all together.

1. [★ Kubernetes Cluster Architecture — Control Plane, Nodes, and the Reconciliation Loop](core-concepts/cluster-architecture.md) — API server, etcd, scheduler, controller-manager, kubelet, kube-proxy, container runtime, declarative model, control plane HA _(2026-04-24)_
2. [★ The Kubernetes API — Resources, Objects, and What Happens on `kubectl apply`](core-concepts/api-and-objects.md) — API groups/versioning, resource vs object vs kind, spec vs status, admission chain, server-side apply, CRDs intro _(2026-04-24)_
3. [★ Namespaces, Labels, Selectors, and Resource Organization](core-concepts/namespaces-and-labels.md) — namespaces as soft multi-tenancy, label conventions, equality/set-based selectors, how selectors drive Services and Deployments, annotations, resource quotas intro _(2026-04-24)_

---

## Tier 2 — Workloads

How you tell Kubernetes to run your containers. Stateless, stateful, per-node, and batch patterns.

4. [★ Pods, ReplicaSets, and Deployments — Running Stateless Applications](workloads/pods-and-deployments.md) — pod model, init/sidecar containers, ReplicaSet, Deployment strategies (RollingUpdate/Recreate), rollout/rollback, scaling _(2026-04-24)_
5. [★ StatefulSets and DaemonSets — Stateful and Per-Node Workloads](workloads/statefulsets-and-daemonsets.md) — stable identity, volumeClaimTemplates, headless Services, DaemonSet use cases, rolling updates _(2026-04-24)_
6. [○ Jobs and CronJobs — Batch and Scheduled Workloads](workloads/jobs-and-cronjobs.md) — completions/parallelism, backoffLimit, CronJob concurrencyPolicy, practical patterns (migrations, scheduled reports) _(2026-04-24)_

---

## Tier 3 — Networking

Kubernetes networking from the platform perspective. How Services route traffic, how Ingress exposes it externally, and how policies restrict it.

7. [★ Kubernetes Services — ClusterIP, NodePort, LoadBalancer, and Service Discovery](networking/services-and-discovery.md) — service types, EndpointSlices, kube-proxy modes, headless Services, topology-aware routing _(2026-04-24)_
8. [★ Ingress, Gateway API, and External Traffic Routing](networking/ingress-and-gateway-api.md) — Ingress resource, controllers (nginx, Traefik, ALB), TLS termination, Gateway API (HTTPRoute/GRPCRoute), cert-manager _(2026-04-24)_
9. [○ Network Policies and CoreDNS — Segmentation and Service Discovery](networking/network-policies-and-dns.md) — NetworkPolicy spec, default deny, CNI requirements, CoreDNS resolution, service FQDN, ndots:5, DNS debugging _(2026-04-24)_

---

## Tier 4 — Configuration & Storage

How you inject configuration and persist data. Platform-level perspective complementing the application-level coverage in the Java and TypeScript paths.

10. [★ ConfigMaps and Secrets — Injecting Configuration into Pods](configuration/configmaps-and-secrets.md) — creation patterns, volume vs env mounting, immutable ConfigMaps, encryption at rest, external secrets operators _(2026-04-24)_
11. [★ Persistent Volumes, PVCs, and StorageClasses — Stateful Storage in Kubernetes](configuration/persistent-volumes.md) — PV/PVC abstraction, access modes, reclaim policies, dynamic provisioning, CSI, volume snapshots _(2026-04-24)_
12. [★ Resource Requests, Limits, QoS Classes, and LimitRanges](configuration/resource-management.md) — CPU vs memory, requests vs limits, QoS classes, OOMKill, CPU throttling, LimitRange, ResourceQuota, JVM/Node.js tuning _(2026-04-24)_

---

## Tier 5 — Security

Cluster and workload security. RBAC, pod hardening, and supply chain integrity.

13. [★ RBAC, ServiceAccounts, and Identity in Kubernetes](security/rbac-and-service-accounts.md) — authentication methods, ServiceAccount tokens, Role/ClusterRole, least-privilege patterns, audit logging _(2026-04-24)_
14. [○ Pod Security Standards, Security Contexts, and Container Hardening](security/pod-security.md) — Pod Security Admission, securityContext, distroless images, image scanning (Trivy), admission controllers (OPA/Kyverno) _(2026-04-24)_
15. [○ Secrets Management and Supply Chain Security in Kubernetes](security/secrets-and-supply-chain.md) — encryption at rest, ESO, Sealed Secrets, CSI secrets store, image provenance (SBOM/SLSA), Falco _(2026-04-24)_

---

## Tier 6 — Operations & Observability

Day-2 operations: deploying, packaging, debugging, monitoring, and scaling.

16. [★ kubectl Mastery — Debugging, Introspection, and Productivity](operations/kubectl-mastery.md) — essential commands, jsonpath/custom-columns, ephemeral debug containers, krew plugins, productivity patterns _(2026-04-24)_
17. [★ Helm and Kustomize — Packaging and Templating Kubernetes Manifests](operations/helm-and-kustomize.md) — Helm charts/values/hooks, Kustomize bases/overlays/patches, when to use each _(2026-04-24)_
18. [○ Kubernetes Monitoring and Logging — Prometheus, Grafana, and Log Aggregation](operations/monitoring-and-logging.md) — Prometheus stack, ServiceMonitor CRDs, key K8s metrics, Grafana, Fluent Bit/Loki, OpenTelemetry Collector _(2026-04-24)_
19. [★ Autoscaling in Kubernetes — HPA, VPA, Cluster Autoscaler, and KEDA](operations/autoscaling.md) — HPA v2, VPA, Cluster Autoscaler, Karpenter, KEDA for event-driven scaling _(2026-04-24)_

---

## Tier 7 — Production Patterns

Patterns for running Kubernetes in production at scale. GitOps, multi-tenancy, disaster recovery, and developer experience.

20. [○ GitOps and Continuous Delivery — ArgoCD, Flux, and Deployment Pipelines](production/gitops-and-cd.md) — GitOps principles, ArgoCD, Flux v2, progressive delivery (canary/blue-green), image update automation _(2026-04-24)_
21. [○ Multi-Tenancy, Cost Optimization, and Cluster Strategy](production/multi-tenancy-and-cost.md) — single vs multi-cluster, namespace isolation, vCluster, Kubecost, spot instances, FinOps _(2026-04-24)_
22. [○ Backup, Disaster Recovery, and Cluster Lifecycle](production/disaster-recovery.md) — etcd backup, Velero, cluster upgrades, node drain/cordon, PDBs, multi-region patterns _(2026-04-24)_
23. [○ Developer Experience on Kubernetes — Local Dev, Debugging, and Inner Loop](production/developer-experience.md) — local K8s (kind, minikube, k3d), Tilt/Skaffold, remote debugging, preview environments, when K8s is overkill _(2026-04-24)_

---

## Quick Reference by Topic

### Core Concepts

- [Cluster Architecture](core-concepts/cluster-architecture.md)
- [The Kubernetes API](core-concepts/api-and-objects.md)
- [Namespaces, Labels & Selectors](core-concepts/namespaces-and-labels.md)

### Workloads

- [Pods, ReplicaSets & Deployments](workloads/pods-and-deployments.md)
- [StatefulSets & DaemonSets](workloads/statefulsets-and-daemonsets.md)
- [Jobs & CronJobs](workloads/jobs-and-cronjobs.md)

### Networking

- [Services & Discovery](networking/services-and-discovery.md)
- [Ingress & Gateway API](networking/ingress-and-gateway-api.md)
- [Network Policies & CoreDNS](networking/network-policies-and-dns.md)

### Configuration & Storage

- [ConfigMaps & Secrets](configuration/configmaps-and-secrets.md)
- [Persistent Volumes & StorageClasses](configuration/persistent-volumes.md)
- [Resource Management](configuration/resource-management.md)

### Security

- [RBAC & ServiceAccounts](security/rbac-and-service-accounts.md)
- [Pod Security](security/pod-security.md)
- [Secrets & Supply Chain Security](security/secrets-and-supply-chain.md)

### Operations & Observability

- [kubectl Mastery](operations/kubectl-mastery.md)
- [Helm & Kustomize](operations/helm-and-kustomize.md)
- [Monitoring & Logging](operations/monitoring-and-logging.md)
- [Autoscaling](operations/autoscaling.md)

### Production Patterns

- [GitOps & CD](production/gitops-and-cd.md)
- [Multi-Tenancy & Cost](production/multi-tenancy-and-cost.md)
- [Disaster Recovery](production/disaster-recovery.md)
- [Developer Experience](production/developer-experience.md)

### Related Paths

- [Spring Boot on Kubernetes](../java/configurations/kubernetes-spring-boot.md) — application-level K8s config for Java
- [Node.js in Kubernetes](../typescript/production/nodejs-in-kubernetes.md) — application-level K8s config for Node
- [Container Networking](../networking/advanced/container-networking.md) — Docker/K8s CNI from the network layer
- [Service Mesh](../networking/advanced/service-mesh.md) — sidecar proxies, Istio/Linkerd
