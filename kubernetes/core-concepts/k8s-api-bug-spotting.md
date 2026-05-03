---
title: "Kubernetes API — Bug Spotting"
date: 2026-05-03
updated: 2026-05-03
tags: [bug-spotting, kubernetes, api, admission, rbac, crd, webhooks]
---

# Kubernetes API — Bug Spotting

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `bug-spotting` `kubernetes` `api` `admission`

---

## Table of Contents
1. [How to use this doc](#how-to-use-this-doc)
2. [Easy (warm-up traps)](#1-easy-warm-up-traps)
3. [Subtle (review-passers)](#2-subtle-review-passers)
4. [Senior trap (production-only failures)](#3-senior-trap-production-only-failures)
5. [Solutions](#4-solutions)
6. [Related](#related)
7. [References](#references)

## Summary

Active-recall practice for the Kubernetes API surface: server-side apply field-manager conflicts, admission webhook ordering and side-effect declarations, `resourceVersion`/410-Gone handling on watches, finalizers and ownerReferences, CRD schema evolution, RBAC traps, and ListAndWatch pitfalls. 22 broken snippets — YAML manifests, kubectl invocations, and controller-runtime Go — graded easy → subtle → senior trap. Cite the official-doc gotcha for each fix; don't just memorize the answer.

## How to use this doc

- Try to spot the bug before opening `<details>`.
- The hint inside `<details>` is one line; the full root-cause + fix is in §4 Solutions, keyed by bug number.
- Difficulty sections are cumulative — Easy traps are the ones reviewers should never miss, Subtle are the ones that pass review, Senior traps usually only show up in postmortems.

---

## 1. Easy (warm-up traps)

### Bug 1 — `kubectl apply` after `kubectl create`
```bash
kubectl create -f deploy.yaml
# ...edit deploy.yaml replicas: 3 -> 5...
kubectl apply -f deploy.yaml
```
<details><summary>Hint</summary>
The first command never wrote a last-applied annotation; what does apply diff against?
</details>

### Bug 2 — Secret stored as `data:`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  password: hunter2
```
<details><summary>Hint</summary>
`data:` and `stringData:` do not have the same encoding contract.
</details>

### Bug 3 — Empty `matchLabels` selector
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app
  namespace: prod
spec:
  podSelector:
    matchLabels: {}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {}
```
<details><summary>Hint</summary>
What does an empty selector match in Kubernetes?
</details>

### Bug 4 — `imagePullPolicy: Always` with private registry, no secret
```yaml
spec:
  containers:
  - name: app
    image: registry.internal.example.com/app:1.4.0
    imagePullPolicy: Always
```
<details><summary>Hint</summary>
Where does the kubelet get the credentials when no `imagePullSecrets` are referenced?
</details>

### Bug 5 — RBAC wildcard verb
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-reader
  namespace: prod
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["*"]
```
<details><summary>Hint</summary>
"Read-only" is in the role name but not in the rule.
</details>

### Bug 6 — Default ServiceAccount token mounted in unrelated pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-resizer
spec:
  containers:
  - name: resize
    image: ghcr.io/example/resize:1.0.0
```
<details><summary>Hint</summary>
A pod that doesn't talk to the API still gets a token by default unless you opt out.
</details>

### Bug 7 — `hostPath` volume in an untrusted workload
```yaml
spec:
  containers:
  - name: ci-runner
    image: ghcr.io/example/ci:latest
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
```
<details><summary>Hint</summary>
What can a process do with the host's container runtime socket?
</details>

---

## 2. Subtle (review-passers)

### Bug 8 — Server-side apply conflict from two field managers
```bash
# CI pipeline
kubectl apply --server-side --field-manager=ci-pipeline -f deploy.yaml

# Later, an operator reconciles the same Deployment
kubectl apply --server-side --field-manager=ops-operator -f patch.yaml
# error: Apply failed with 1 conflict: conflict with "ci-pipeline"
```
<details><summary>Hint</summary>
Two managers want the same field; SSA wants you to choose, not retry blindly.
</details>

### Bug 9 — Watch resumed at stale resourceVersion
```go
// controller startup
list, _ := client.CoreV1().Pods(ns).List(ctx, metav1.ListOptions{})
rv := list.ResourceVersion
time.Sleep(2 * time.Hour) // controller paused / GC pressure
w, _ := client.CoreV1().Pods(ns).Watch(ctx, metav1.ListOptions{ResourceVersion: rv})
for ev := range w.ResultChan() { handle(ev) }
```
<details><summary>Hint</summary>
The etcd compaction window is finite; what status does the apiserver send when your RV is gone?
</details>

### Bug 10 — Mutating webhook reorders fields the validating webhook checks
```yaml
# MutatingWebhookConfiguration (runs first)
- name: inject-sidecar.example.com
  reinvocationPolicy: Never
  rules: [{ operations: ["CREATE"], resources: ["pods"] }]

# ValidatingWebhookConfiguration
- name: require-resource-limits.example.com
  rules: [{ operations: ["CREATE"], resources: ["pods"] }]
```
<details><summary>Hint</summary>
What if a *later* mutating webhook adds a container after this one already ran?
</details>

### Bug 11 — `failurePolicy: Ignore` with critical security webhook
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: enforce-pod-security
webhooks:
- name: psa.example.com
  failurePolicy: Ignore
  timeoutSeconds: 30
  sideEffects: None
```
<details><summary>Hint</summary>
What happens if your security gatekeeper webhook is unreachable?
</details>

### Bug 12 — CRD v1 → v2 adds required field, old objects break
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
spec:
  versions:
  - name: v1
    served: true
    storage: false
    schema: { openAPIV3Schema: { type: object, properties: { spec: { properties: { size: { type: integer } } } } } }
  - name: v2
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        required: [spec]
        properties:
          spec:
            type: object
            required: [size, region]   # NEW required field
            properties:
              size:   { type: integer }
              region: { type: string }
```
<details><summary>Hint</summary>
Existing v1 objects don't have `region`; what happens when they're served as v2 or written back?
</details>

### Bug 13 — Finalizer never removed
```go
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var obj v1.Foo
    _ = r.Get(ctx, req.NamespacedName, &obj)
    if obj.DeletionTimestamp.IsZero() {
        if !controllerutil.ContainsFinalizer(&obj, "foo.example.com/cleanup") {
            controllerutil.AddFinalizer(&obj, "foo.example.com/cleanup")
            r.Update(ctx, &obj)
        }
        return ctrl.Result{}, nil
    }
    if err := r.cleanupExternal(ctx, &obj); err != nil {
        return ctrl.Result{}, err
    }
    return ctrl.Result{}, nil
}
```
<details><summary>Hint</summary>
Cleanup ran. Did the finalizer come off?
</details>

### Bug 14 — Cross-namespace ownerReference
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: app
  ownerReferences:
  - apiVersion: v1
    kind: ConfigMap
    name: parent
    namespace: shared        # owner in different namespace
    uid: 6f0e...
```
<details><summary>Hint</summary>
The garbage collector has a rule about scopes here.
</details>

### Bug 15 — Webhook with side effects in dry-run
```yaml
webhooks:
- name: provision-bucket.example.com
  sideEffects: None        # webhook actually creates an S3 bucket
  rules:
  - operations: ["CREATE"]
    resources: ["buckets"]
```
<details><summary>Hint</summary>
What does `sideEffects: None` promise the apiserver?
</details>

### Bug 16 — HPA on a metric that lags traffic
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
  - type: External
    external:
      metric: { name: queue_depth_avg_5m }   # 5-minute moving average
      target: { type: AverageValue, averageValue: "100" }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
```
<details><summary>Hint</summary>
By the time the metric reacts, the queue is already on fire.
</details>

### Bug 17 — ConfigMap update doesn't propagate to pods
```yaml
spec:
  containers:
  - name: app
    image: app:1.0
    envFrom:
    - configMapRef:
        name: app-config
```
<details><summary>Hint</summary>
`envFrom` and `volumeMounts` differ on whether updates are seen by a running pod.
</details>

### Bug 18 — Pod Security Admission level mismatch
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
---
apiVersion: v1
kind: Pod
metadata:
  name: legacy-agent
  namespace: payments
spec:
  containers:
  - name: agent
    image: legacy/agent:3
    securityContext:
      privileged: true
```
<details><summary>Hint</summary>
The namespace label and the pod's securityContext are about to disagree.
</details>

---

## 3. Senior trap (production-only failures)

### Bug 19 — Reflector crash-loops on 410 Gone
```go
informer := cache.NewSharedIndexInformer(
    &cache.ListWatch{
        ListFunc:  func(o metav1.ListOptions) (runtime.Object, error) { return cs.CoreV1().Pods("").List(ctx, o) },
        WatchFunc: func(o metav1.ListOptions) (watch.Interface, error) { return cs.CoreV1().Pods("").Watch(ctx, o) },
    },
    &v1.Pod{}, 0, cache.Indexers{},
)
go informer.Run(stop)
// log spam: "watch chan error: too old resource version: 84512 (90211)"
```
<details><summary>Hint</summary>
Cluster-wide pod watch + a slow consumer = compaction event horizon.
</details>

### Bug 20 — Lease held past pod death (leader stuck)
```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-controller
spec:
  holderIdentity: controller-7d8f-pod-xyz
  leaseDurationSeconds: 600
  renewTime: "2026-05-03T10:00:00Z"
```
```go
leaderelection.LeaderElectionConfig{
    LeaseDuration: 600 * time.Second,
    RenewDeadline: 590 * time.Second,
    RetryPeriod:   2 * time.Second,
}
```
<details><summary>Hint</summary>
Pod OOM-killed at 10:00:01. When does anyone else acquire?
</details>

### Bug 21 — CRD conversion webhook returns wrong storage version
```go
func Convert(req *apix.ConversionRequest) *apix.ConversionResponse {
    out := []runtime.RawExtension{}
    for _, obj := range req.Objects {
        u := &unstructured.Unstructured{}
        _ = u.UnmarshalJSON(obj.Raw)
        u.SetAPIVersion("example.com/v1")  // bug: ignores req.DesiredAPIVersion
        raw, _ := u.MarshalJSON()
        out = append(out, runtime.RawExtension{Raw: raw})
    }
    return &apix.ConversionResponse{ConvertedObjects: out, Result: metav1.Status{Status: "Success"}}
}
```
<details><summary>Hint</summary>
The apiserver asked for a specific target version; you ignored it.
</details>

### Bug 22 — APF (API Priority and Fairness) starvation from a chatty controller
```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata: { name: workload-low }
spec:
  priorityLevelConfiguration: { name: workload-low }
  matchingPrecedence: 9000
  rules:
  - subjects: [{ kind: ServiceAccount, serviceAccount: { namespace: ops, name: noisy-controller } }]
    resourceRules:
    - verbs: ["*"]
      apiGroups: ["*"]
      resources: ["*"]
```
```go
// controller doing cluster-wide LIST every 10s
wait.Until(func() {
    _, _ = cs.CoreV1().Pods("").List(ctx, metav1.ListOptions{})  // no resourceVersion="0"
}, 10*time.Second, stop)
```
<details><summary>Hint</summary>
Two compounding mistakes: a list-from-etcd loop, and a flow schema that doesn't isolate its blast radius.
</details>

### Bug 23 — Cluster-wide informer cache memory blow-up
```go
mgr, _ := ctrl.NewManager(cfg, ctrl.Options{
    Scheme: scheme,
    // (no Cache.ByObject restriction)
})
ctrl.NewControllerManagedBy(mgr).For(&corev1.Pod{}).Complete(r)
```
<details><summary>Hint</summary>
Controller cares about one namespace; cache decided to slurp the planet.
</details>

---

## 4. Solutions

### Bug 1 — `kubectl apply` after `kubectl create`
**Root cause:** Client-side `kubectl apply` diffs against the `kubectl.kubernetes.io/last-applied-configuration` annotation. `kubectl create` doesn't write it, so the first `apply` mis-computes the three-way merge — fields you removed from the manifest may not be removed from the live object.
**Fix:** Use `kubectl apply` from the start, or migrate to server-side apply which doesn't depend on the annotation:
```bash
kubectl apply --server-side --field-manager=ops -f deploy.yaml
```
**Reference:** [Server-Side Apply — kubernetes.io](https://kubernetes.io/docs/reference/using-api/server-side-apply/) and [`kubectl apply` declarative management](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/).

### Bug 2 — Secret stored as `data:`
**Root cause:** `data:` requires base64-encoded values. `hunter2` is not valid base64 (it parses, but decodes to garbage bytes). The Secret object will be created and consumers get the wrong password.
**Fix:**
```yaml
type: Opaque
stringData:
  password: hunter2
```
**Reference:** [Secrets — kubernetes.io](https://kubernetes.io/docs/concepts/configuration/secret/#secret-types) ("`stringData` field is provided for convenience").

### Bug 3 — Empty `matchLabels` selector
**Root cause:** An empty `matchLabels: {}` selects *every* pod in the namespace, and `from: [{ podSelector: {} }]` allows ingress from every pod. The policy says "allow all" not "allow app".
**Fix:**
```yaml
podSelector:
  matchLabels: { app: app }
ingress:
- from:
  - podSelector: { matchLabels: { app: frontend } }
```
**Reference:** [Network Policies — kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/network-policies/) ("An empty `podSelector` selects all pods in the namespace").

### Bug 4 — `imagePullPolicy: Always` with private registry
**Root cause:** No `imagePullSecrets` and the kubelet has no node-level credential helper for `registry.internal.example.com`. The pull silently fails with `ErrImagePull`/`ImagePullBackOff`. Worse, `Always` masks any locally cached image from a previous pull.
**Fix:** Reference an `imagePullSecrets` entry, or configure the kubelet credential provider for the registry.
**Reference:** [Images — kubernetes.io](https://kubernetes.io/docs/concepts/containers/images/#using-a-private-registry).

### Bug 5 — RBAC wildcard verb
**Root cause:** `verbs: ["*"]` includes `delete`, `update`, `patch`, `deletecollection`. The role allows full CRUD on configmaps in the namespace, including overwriting any operator-managed configmap.
**Fix:**
```yaml
verbs: ["get", "list", "watch"]
```
**Reference:** [RBAC — kubernetes.io](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) ("Use the principle of least privilege"; the wildcard is called out explicitly).

### Bug 6 — Default ServiceAccount token mounted in unrelated pod
**Root cause:** Pre-1.24 a default token Secret was auto-mounted; from 1.24+ a projected token is mounted unless `automountServiceAccountToken: false`. Either way, a workload that never talks to the API gets a credential that an attacker can exfiltrate.
**Fix:**
```yaml
spec:
  automountServiceAccountToken: false
```
**Reference:** [Configure ServiceAccount — kubernetes.io](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting); [KEP-2799 (BoundServiceAccountTokenVolumes)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/2799-reduction-of-secret-based-service-account-token).

### Bug 7 — `hostPath` to docker.sock
**Root cause:** Mounting the runtime socket gives the container root on the host: it can launch privileged containers, mount host paths, and pivot. This is a documented escape primitive.
**Fix:** Don't. Use a sandboxed builder (Kaniko, Buildah rootless) or BuildKit with rootless mode in a dedicated namespace with PSA `restricted`.
**Reference:** [Pod Security Standards — kubernetes.io](https://kubernetes.io/docs/concepts/security/pod-security-standards/) (`hostPath` is forbidden in `restricted`); CIS Benchmark 5.2.x.

### Bug 8 — Server-side apply conflict
**Root cause:** Two field managers (`ci-pipeline`, `ops-operator`) both declared ownership of the same field. SSA refuses to silently overwrite. Blindly retrying does not help; you must either drop the field from one manager's manifest or take ownership explicitly.
**Fix:** Decide who owns the field. If the operator is authoritative:
```bash
kubectl apply --server-side --field-manager=ops-operator --force-conflicts -f patch.yaml
```
And remove that field from the CI manifest so CI stops re-claiming it.
**Reference:** [Server-Side Apply — Conflicts](https://kubernetes.io/docs/reference/using-api/server-side-apply/#conflicts).

### Bug 9 — Watch resumed at stale resourceVersion
**Root cause:** etcd compacts old revisions. After a long pause the apiserver returns `410 Gone`; if the controller doesn't relist, it loops forever or, worse, silently misses events.
**Fix:** Handle `410` by re-listing to obtain a fresh `resourceVersion`, then re-establish the watch. Use the standard `Reflector`/informer machinery — it does this for you.
```go
// On 410 Gone: relist, take new RV, re-watch.
```
**Reference:** [API concepts — efficient detection of changes](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes) ("if the requested resourceVersion is not available, the server returns 410 Gone").

### Bug 10 — Mutating webhook ordering / reinvocation
**Root cause:** Mutating webhooks run alphabetically (then by configuration order). If webhook A injects a sidecar but webhook B (running later) adds another container, A's invariants are no longer enforced — and the validating webhook runs only once at the end.
**Fix:** Set `reinvocationPolicy: IfNeeded` on the mutator that needs to re-validate after later mutations, and rely on the validating webhook for invariants. Ordering across configurations is *not* configurable.
**Reference:** [Dynamic Admission Control — reinvocation policy](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#reinvocation-policy).

### Bug 11 — `failurePolicy: Ignore` on a security webhook
**Root cause:** If the webhook is down or times out, requests are *admitted unmodified*. A security gate becomes a no-op the moment the webhook crashes — which is exactly when you need it.
**Fix:** Use `failurePolicy: Fail` on enforcement webhooks, with a low `timeoutSeconds` (1–3s), an HA webhook deployment, and a `namespaceSelector` excluding `kube-system` so a webhook outage can't deadlock the control plane.
**Reference:** [Dynamic Admission Control — failure policy](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#failure-policy); [Avoiding deadlocks in self-hosted webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#avoiding-deadlocks-in-self-hosted-webhooks).

### Bug 12 — CRD adds required field
**Root cause:** Existing v1 objects lack `region`. Reading them as v2 (or any write-back round trip) fails OpenAPI validation. The storage version bump effectively bricks the existing fleet.
**Fix:** Either keep `region` optional with a default in v2's schema, or add a conversion webhook that synthesizes a default when round-tripping v1 → v2.
```yaml
region: { type: string, default: "us-east-1" }
```
**Reference:** [CRD versioning — schema changes](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/); [structural schemas](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema).

### Bug 13 — Finalizer never removed
**Root cause:** The deletion branch runs `cleanupExternal` and returns, but never calls `RemoveFinalizer` + `Update`. The object is stuck `Terminating` forever; `kubectl delete` hangs.
**Fix:**
```go
if err := r.cleanupExternal(ctx, &obj); err != nil {
    return ctrl.Result{}, err
}
controllerutil.RemoveFinalizer(&obj, "foo.example.com/cleanup")
return ctrl.Result{}, r.Update(ctx, &obj)
```
**Reference:** [Finalizers — kubernetes.io](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/); [controller-runtime book — finalizers](https://book.kubebuilder.io/reference/using-finalizers.html).

### Bug 14 — Cross-namespace ownerReference
**Root cause:** `ownerReferences` must be in the same namespace for namespaced→namespaced, and a namespaced object cannot own a cluster-scoped one. Cross-namespace owners are silently treated as already-deleted by GC, which can wipe the dependent.
**Fix:** Co-locate owner and dependent in the same namespace, or use a label-based association instead of an owner reference.
**Reference:** [Owners and dependents — kubernetes.io](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) ("cross-namespace owner references are disallowed by design").

### Bug 15 — Webhook with side effects in dry-run
**Root cause:** Declaring `sideEffects: None` tells the apiserver "safe to call during dry-run". If the webhook actually mutates external state (creates an S3 bucket), every `--dry-run=server` request leaks resources.
**Fix:** Use `sideEffects: NoneOnDryRun` and check `request.dryRun` inside the handler — skip the external call when true.
**Reference:** [Dynamic Admission Control — side effects](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#side-effects).

### Bug 16 — HPA on lagging metric
**Root cause:** A 5-minute moving average smooths the signal; by the time it crosses the target, the queue is already deep. With `stabilizationWindowSeconds: 0` you also flap on noise.
**Fix:** Use a faster metric (1-minute or instantaneous queue depth) and keep a small stabilization window. Combine HPA with KEDA for queue-depth-driven scaling, and consider a predictive trigger.
**Reference:** [HorizontalPodAutoscaler — kubernetes.io](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/); [HPA stabilization](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#stabilization-window).

### Bug 17 — ConfigMap update doesn't propagate
**Root cause:** `envFrom` (and `env`) are baked into the container at start time. ConfigMap edits don't reach a running pod. Only volume-mounted ConfigMaps are updated in place (eventually-consistent, kubelet sync period).
**Fix:** Mount the ConfigMap as a volume and make the app re-read on change, or trigger a rollout on ConfigMap hash change (e.g., `kubectl rollout restart`, Reloader, or annotate the PodTemplate with a checksum).
**Reference:** [ConfigMaps — kubernetes.io](https://kubernetes.io/docs/concepts/configuration/configmap/#mounted-configmaps-are-updated-automatically).

### Bug 18 — Pod Security Admission level mismatch
**Root cause:** Namespace enforces `restricted`, but the pod requests `privileged: true`. The admission plugin rejects creation; if you only labeled `warn`/`audit` (not `enforce`), the pod silently runs with privileges and your audit log fills with violations.
**Fix:** Either run the workload in a namespace with `baseline`/`privileged` enforcement (and isolate it), or refactor to drop privileges. Verify with `kubectl auth can-i` and dry-run admission.
**Reference:** [Pod Security Admission — kubernetes.io](https://kubernetes.io/docs/concepts/security/pod-security-admission/); [PSA labels](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-admission-labels-for-namespaces).

### Bug 19 — Reflector crash-loops on 410 Gone
**Root cause:** A cluster-wide pod watch falls behind compaction (etcd default keeps ~5 minutes of revisions). Naive watchers never catch up. The fix is to relist *with `resourceVersion=""`* (most recent) and resume.
**Fix:** Use `client-go`'s informer/Reflector — it handles 410 by relisting. If hand-rolling, on `errors.IsResourceExpired(err)` clear the bookmark and re-list.
```go
if errors.IsResourceExpired(err) {
    list, _ := cs.CoreV1().Pods("").List(ctx, metav1.ListOptions{ResourceVersion: ""})
    rv = list.ResourceVersion
}
```
**Reference:** [API concepts — 410 Gone](https://kubernetes.io/docs/reference/using-api/api-concepts/#410-gone-responses); [client-go Reflector source](https://github.com/kubernetes/client-go/blob/master/tools/cache/reflector.go).

### Bug 20 — Lease held past pod death
**Root cause:** With `LeaseDuration: 600s`, a peer can only acquire after the holder's lease expires — up to 10 minutes after the pod died. During that window, no leader reconciles.
**Fix:** Tune `LeaseDuration` to seconds-not-minutes (commonly 15s lease / 10s renew / 2s retry for fast failover), and ensure the holder process exits cleanly so `controller-runtime` releases the Lease via `holderIdentity` clear.
**Reference:** [Leases — kubernetes.io](https://kubernetes.io/docs/concepts/architecture/leases/); [client-go leaderelection package](https://pkg.go.dev/k8s.io/client-go/tools/leaderelection).

### Bug 21 — Conversion webhook ignores `desiredAPIVersion`
**Root cause:** The webhook always returns `example.com/v1` regardless of what the apiserver requested. When apiserver asks for v2 (storage version), it gets v1 back — apiserver rejects with a conversion error and the object becomes unreadable.
**Fix:** Honor `req.DesiredAPIVersion` and convert per-version:
```go
u.SetAPIVersion(req.DesiredAPIVersion)
```
**Reference:** [Webhook conversion — kubernetes.io](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/#webhook-conversion); see also [kubernetes/kubernetes#92622](https://github.com/kubernetes/kubernetes/issues/92622) for related conversion bugs.

### Bug 22 — APF starvation
**Root cause:** A controller doing `List("")` without `resourceVersion: "0"` reads from etcd every 10 seconds, drowning the priority level. The `FlowSchema` `verbs/apiGroups/resources: ["*"]` means *every* request from that SA shares the same low-priority bucket — including its watch maintenance.
**Fix:** (a) fix the controller to use informers (cache reads, `List` with `ResourceVersion: "0"` for cache-served reads), (b) split FlowSchemas so the noisy controller's writes/lists don't starve its own watches.
```go
List(ctx, metav1.ListOptions{ResourceVersion: "0"})  // served from cache
```
**Reference:** [API Priority and Fairness — kubernetes.io](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/); [resourceVersion semantics](https://kubernetes.io/docs/reference/using-api/api-concepts/#resource-versions).

### Bug 23 — Cluster-wide informer cache memory blow-up
**Root cause:** controller-runtime's default cache watches *every* object of the For/Owns/Watches kind cluster-wide. For Pods in a 10k-node cluster this can pin hundreds of MB or more.
**Fix:** Scope the cache:
```go
mgr, _ := ctrl.NewManager(cfg, ctrl.Options{
    Scheme: scheme,
    Cache: cache.Options{
        ByObject: map[client.Object]cache.ByObject{
            &corev1.Pod{}: { Namespaces: map[string]cache.Config{ "myns": {} } },
        },
    },
})
```
For label-selectable subsets, use `cache.ByObject{Label: labels.SelectorFromSet(...)}`.
**Reference:** [controller-runtime cache options](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache#Options); [Kubebuilder book — watching resources](https://book.kubebuilder.io/reference/watching-resources/operator-managed.html).

---

## Related

- [Kubernetes API and Objects](api-and-objects.md) — concept primer this doc tests.
- [Cluster Architecture](cluster-architecture.md) — apiserver, etcd, controller-manager interactions referenced in the senior traps.
- [RBAC and ServiceAccounts](../security/rbac-and-service-accounts.md) — Bugs 5, 6, 22 deepen here.
- [Pod Security](../security/pod-security.md) — Bugs 7, 18 cross-reference PSA and Pod Security Standards.

## References

- Kubernetes docs — Server-Side Apply: https://kubernetes.io/docs/reference/using-api/server-side-apply/
- Kubernetes docs — Dynamic Admission Control: https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
- Kubernetes docs — API Concepts (resourceVersion, 410 Gone, watches): https://kubernetes.io/docs/reference/using-api/api-concepts/
- Kubernetes docs — Finalizers: https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/
- Kubernetes docs — Owners and Dependents: https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/
- Kubernetes docs — RBAC: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- Kubernetes docs — Pod Security Admission: https://kubernetes.io/docs/concepts/security/pod-security-admission/
- Kubernetes docs — Pod Security Standards: https://kubernetes.io/docs/concepts/security/pod-security-standards/
- Kubernetes docs — CRD versioning and conversion: https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/
- Kubernetes docs — API Priority and Fairness: https://kubernetes.io/docs/concepts/cluster-administration/flow-control/
- Kubernetes docs — ConfigMaps: https://kubernetes.io/docs/concepts/configuration/configmap/
- Kubernetes docs — Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
- Kubernetes docs — HPA: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- KEP-2799 — Reduction of Secret-Based ServiceAccount Tokens: https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/2799-reduction-of-secret-based-service-account-token
- controller-runtime cache options: https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache
- client-go Reflector (handles 410 Gone): https://github.com/kubernetes/client-go/blob/master/tools/cache/reflector.go
- Kubebuilder book — finalizers: https://book.kubebuilder.io/reference/using-finalizers.html
- Kubebuilder book — watching resources: https://book.kubebuilder.io/reference/watching-resources/operator-managed.html
