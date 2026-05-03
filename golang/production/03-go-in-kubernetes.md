---
title: "Go in Kubernetes"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, production, kubernetes, graceful-shutdown, gomaxprocs, distroless]
---

# Go in Kubernetes

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `production` `kubernetes` `graceful-shutdown` `gomaxprocs` `distroless`

---

## Table of Contents

1. [The deployment story — single static binary](#1-the-deployment-story--single-static-binary)
2. [Multi-stage Dockerfile](#2-multi-stage-dockerfile)
3. [Image base options: scratch, distroless, alpine](#3-image-base-options-scratch-distroless-alpine)
4. [Container hardening: non-root, read-only fs, drop caps](#4-container-hardening-non-root-read-only-fs-drop-caps)
5. [`GOMAXPROCS` in containers — the host/quota mismatch](#5-gomaxprocs-in-containers--the-hostquota-mismatch)
6. [`GOMEMLIMIT` for cgroup-aware memory ceilings](#6-gomemlimit-for-cgroup-aware-memory-ceilings)
7. [Graceful shutdown — signals, drain, preStop](#7-graceful-shutdown--signals-drain-prestop)
8. [Liveness vs readiness vs startup probes](#8-liveness-vs-readiness-vs-startup-probes)
9. [Resource requests and limits sizing](#9-resource-requests-and-limits-sizing)
10. [Pod lifecycle and the rolling update sequence](#10-pod-lifecycle-and-the-rolling-update-sequence)
11. [Logging on stdout in the k8s context](#11-logging-on-stdout-in-the-k8s-context)

## Summary

Go is one of the friendliest languages to deploy on Kubernetes. The default toolchain produces a single statically-linked binary (with `CGO_ENABLED=0`) that runs on a 2 MB distroless image and starts in tens of milliseconds. The footguns are not in the build — they're in the runtime: `GOMAXPROCS` defaults to the host's CPU count, ignoring the cgroup quota the pod actually has; `GOMEMLIMIT` (Go 1.19+) needs to be set explicitly to align with the pod's memory limit; graceful shutdown must hook `SIGTERM` and call `http.Server.Shutdown` to drain in-flight requests within `terminationGracePeriodSeconds`.

This doc covers the build, the image, the container hardening, the runtime tuning, the signal handling, and the k8s probe semantics that you need to put a Go service on a cluster correctly. It is the operational counterpart to the [pprof](01-profiling-with-pprof.md) and [observability](02-observability.md) docs — you can ship perfectly observable code that still rolls badly because shutdown is wrong.

## 1. The deployment story — single static binary

A Go program built with the default toolchain produces an executable that depends only on `glibc` (or `musl`, depending on your build host) for system calls. Add `CGO_ENABLED=0` and the binary becomes **fully static** — no dynamic linker, no libc, nothing.

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w" -o app ./cmd/api
```

What each flag does:

- `CGO_ENABLED=0` — disable CGo. The resulting binary makes raw syscalls and does not require libc. This is the difference between "runs on scratch" and "needs a base image with libc."
- `-trimpath` — remove the build host's filesystem path from the binary. Reproducible builds, no `/Users/quan/...` leaking into stack traces.
- `-ldflags="-s -w"` — strip the symbol table and DWARF debug info. Roughly 30% smaller binary. Tradeoff: stack traces still work (they use the runtime's own metadata) but you can't run a debugger against the binary. For production this is the right call; keep an unstripped copy for forensics.
- `GOOS=linux GOARCH=amd64` — explicit target. Necessary if you're cross-compiling from macOS, harmless on a Linux build host.

The `CGO_ENABLED=0` choice has a few specific gotchas:

- The default `net` package uses CGo on some platforms for DNS lookups. With CGO disabled, Go uses its **pure-Go resolver**, which reads `/etc/resolv.conf` and queries DNS servers directly. This is fine for most production cases and is in fact what most Kubernetes clusters expect.
- The default `os/user` package similarly uses CGo to call `getpwuid`. With CGo disabled it falls back to parsing `/etc/passwd`. In a distroless container without `/etc/passwd`, calls like `user.Current()` will return errors. Most services don't need this.

If you ship a service that needs CGo (SQLite, Oracle drivers, some crypto libs), you cannot use `scratch` — you need at least `distroless/base` (which includes glibc) or alpine.

## 2. Multi-stage Dockerfile

The canonical pattern: a "builder" stage that has the Go toolchain and a "final" stage that has only the binary.

```dockerfile
# syntax=docker/dockerfile:1.7

# --- builder ---
FROM golang:1.22-alpine AS builder

WORKDIR /src

# Copy module files first for better layer caching
COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w" \
    -o /out/app ./cmd/api

# --- final ---
FROM gcr.io/distroless/static-debian12:nonroot

# Copy the binary
COPY --from=builder /out/app /app

# Run as the pre-existing nonroot user (uid 65532)
USER nonroot:nonroot

ENTRYPOINT ["/app"]
```

Three patterns worth noting:

- **`go mod download` is a separate layer.** As long as `go.mod` and `go.sum` don't change, the dependency download layer is cached. Subsequent rebuilds only re-run the `go build` stage.
- **The `:nonroot` distroless variant** ships with a pre-created non-root user `nonroot` (uid 65532). Using `USER nonroot:nonroot` in the Dockerfile, plus `runAsNonRoot: true` in the pod's `securityContext`, gives you a Go container that runs unprivileged by default.
- **No package manager, no shell.** The final image cannot run `apk add ...` after the fact. This is intentional and is a security feature — there is nothing to exploit if a process drops shell.

For a typical Go service this Dockerfile produces a final image around 10–40 MB, dominated by the Go binary itself.

## 3. Image base options: scratch, distroless, alpine

Three reasonable choices for the final image. The tradeoffs:

| Base | Size | Has shell? | Has libc? | Has TLS roots? | Use when |
|---|---|---|---|---|---|
| `scratch` | 0 bytes | No | No | No | Fully-static binary, no outbound TLS, or you embed your own roots |
| `gcr.io/distroless/static-debian12` | ~2 MB | No | No (musl/static-only) | Yes (`/etc/ssl/certs`) | Default for static Go binaries — best of all worlds |
| `gcr.io/distroless/base-debian12` | ~20 MB | No | Yes (glibc) | Yes | CGo-required binaries, dynamically linked code |
| `alpine:3.19` | ~7 MB | Yes (`ash`) | Yes (musl) | Yes | When you need a shell for debugging or run scripts |

**Why distroless/static is the default recommendation for Go:**

- It is ~2 MB — about 50% the size of alpine and less than 2% of debian.
- It includes `/etc/ssl/certs/ca-certificates.crt`, so outbound TLS works without adding anything.
- It includes `/etc/passwd` and `/etc/group` with a `nonroot` user, so `USER nonroot:nonroot` and `runAsNonRoot: true` work.
- It does not include a shell. There is nothing to `kubectl exec` into the way you would with alpine — which is a security feature, but an operational change.

`scratch` is theoretically smaller but loses the TLS root certificates. If your Go service makes any outbound HTTPS call, scratch will fail with `x509: certificate signed by unknown authority`. The fix is `COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`, but at that point you are reinventing distroless/static. Just use distroless/static.

`alpine` is fine for debugging-friendly images. The musl libc differs from glibc in some edge cases (DNS, threading), and a CGo-built Go binary on alpine has historically had subtle issues. For pure-Go (CGO_ENABLED=0) binaries this doesn't matter, but then you might as well use distroless.

## 4. Container hardening: non-root, read-only fs, drop caps

The pod-level controls that match a hardened image:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: billing-api
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532          # the distroless nonroot uid
        runAsGroup: 65532
        fsGroup: 65532
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: registry.example.com/billing-api:v1.4.2
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: tmp
              mountPath: /tmp     # Go's tempfile use, if any
      volumes:
        - name: tmp
          emptyDir: {}
```

What each control does:

- **`runAsNonRoot: true`** — the kubelet refuses to start the container if its image is configured to run as root. A defense in depth in case someone forgets `USER nonroot` in the Dockerfile.
- **`readOnlyRootFilesystem: true`** — mount `/` read-only. Attackers who get RCE cannot drop a binary onto the disk. Most Go services don't write outside `/tmp`, so this is free; provide a writable `emptyDir` mount for `/tmp` if you do need temp files.
- **`capabilities.drop: ["ALL"]`** — drop every Linux capability. Plain HTTP servers don't need any; if you're not binding to a privileged port (<1024), there's nothing to drop. Compose with `bindHostPort`-style approaches if you must bind to 80/443.
- **`allowPrivilegeEscalation: false`** — disable `setuid` etc. so a child process cannot gain more privileges than the parent.
- **`seccompProfile: RuntimeDefault`** — apply the runtime's default seccomp filter, which blocks dangerous syscalls. Go programs use a small subset of syscalls and don't trip the default filter.

The combination of distroless image + these controls puts you near the pod-security-admission **restricted** baseline by default.

## 5. `GOMAXPROCS` in containers — the host/quota mismatch

The single most common Go-on-Kubernetes performance bug:

```yaml
resources:
  limits:
    cpu: "2"   # 2 cores
```

A pod with `cpu: "2"` running on a 64-core node has the kernel cgroup CPU quota set to 200,000 µs per 100,000 µs period — 2 cores worth. But `runtime.NumCPU()` reads `/proc/cpuinfo`, which the container shares with the host. So `GOMAXPROCS` defaults to **64**.

The runtime spins up 64 P structures and tries to schedule 64 OS threads to run Go code in parallel. The kernel CFS scheduler then throttles those threads down to the cgroup's 2 cores worth of CPU time. The result: lots of context switches, lots of P↔M handoff, lots of cache thrashing, and tail latency that's much worse than if `GOMAXPROCS` had matched the quota in the first place.

The fix is [`go.uber.org/automaxprocs`](https://github.com/uber-go/automaxprocs) — a blank import that reads the cgroup CPU quota at startup and sets `GOMAXPROCS` accordingly:

```go
import _ "go.uber.org/automaxprocs"

func main() {
    // GOMAXPROCS is now set to the cgroup quota
    // ...
}
```

The Uber benchmarks in the README show the impact concretely: a service limited to 2 cores went from 22k RPS / 76 ms p999 latency (with the default GOMAXPROCS=NumCPU) to 45k RPS / 26 ms p999 (with automaxprocs). Roughly 2× throughput and 3× tail latency improvement, from one blank import.

The explanation of how this maps to the Go scheduler — Ps, Ms, work stealing, and why oversubscription hurts — is in the [Goroutines & the Scheduler doc](../concurrency/01-goroutines-and-scheduler.md).

There is an open Go proposal ([#73193](https://github.com/golang/go/issues/73193)) to make the runtime cgroup-aware natively. Until that ships, automaxprocs is the standard fix.

## 6. `GOMEMLIMIT` for cgroup-aware memory ceilings

`GOMEMLIMIT` (Go 1.19+) is the memory analogue of `GOMAXPROCS` — a soft limit on how much memory the Go runtime will use, expressed as a runtime hint to the GC.

The problem it solves: without a memory limit, the GC's pacing target is purely time-based. If your service has a 512 MB cgroup memory limit and the GC decides to let the heap grow to 600 MB before collecting, the kernel OOM-kills the pod. Setting `GOMEMLIMIT=450MiB` tells the runtime to start collecting more aggressively as the heap approaches that ceiling, even at the cost of higher GC CPU usage.

Set it via the environment:

```yaml
containers:
  - name: app
    env:
      - name: GOMEMLIMIT
        value: "450MiB"
    resources:
      limits:
        memory: "512Mi"
```

Or programmatically:

```go
import "runtime/debug"

debug.SetMemoryLimit(450 * 1024 * 1024)
```

Two important details:

- **Set GOMEMLIMIT lower than the cgroup limit.** A common rule of thumb is 90% of the pod limit. Stack memory, mmap'd binary pages, and CGo-allocated memory are not counted by GOMEMLIMIT, so you need headroom.
- **GOMEMLIMIT is a soft limit.** The runtime will sacrifice GC CPU to stay under it, but it can still go over for short bursts. To prevent runaway GC consuming all CPU when the heap is permanently above the limit, the runtime caps GC at 50% of CPU. The remaining 50% goes to your application, which keeps making progress (and may eventually OOM if memory pressure is real).

There is a [`GOGC=off` trick](https://go.dev/doc/gc-guide) to use only `GOMEMLIMIT` for pacing, useful for batch jobs where you want the GC to do nothing until the heap approaches the limit. Don't use it for latency-sensitive services — by the time the GC kicks in, you've used your whole memory budget and pause time will be long.

## 7. Graceful shutdown — signals, drain, preStop

When a pod is deleted, the kubelet sends `SIGTERM` to the container's PID 1, then waits up to `terminationGracePeriodSeconds` (default 30s) before sending `SIGKILL`. A correctly-behaved Go service:

1. Receives `SIGTERM`.
2. Marks itself as not-ready (the readiness probe starts failing).
3. Waits a few seconds for the kube-proxy / load balancer to drain it from endpoints.
4. Calls `http.Server.Shutdown(ctx)` to stop accepting new connections and wait for in-flight requests to complete.
5. Exits cleanly.

The canonical pattern using `signal.NotifyContext` (Go 1.16+):

```go
package main

import (
    "context"
    "errors"
    "log/slog"
    "net/http"
    "os/signal"
    "sync/atomic"
    "syscall"
    "time"
)

func main() {
    ctx, stop := signal.NotifyContext(context.Background(),
        syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    var ready atomic.Bool
    ready.Store(true)

    mux := http.NewServeMux()
    mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
    mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
        if !ready.Load() {
            w.WriteHeader(http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
    })
    mux.HandleFunc("/", appHandler)

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 5 * time.Second,
    }

    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            slog.Error("server failed", "err", err)
        }
    }()

    // Block until SIGTERM
    <-ctx.Done()
    slog.Info("shutdown signal received")

    // 1. Flip readiness off so the load balancer drains us.
    ready.Store(false)

    // 2. Give kube-proxy / the LB a few seconds to notice.
    time.Sleep(5 * time.Second)

    // 3. Stop accepting new conns, drain in-flight requests, with a deadline.
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        slog.Error("graceful shutdown failed", "err", err)
    }
    slog.Info("shutdown complete")
}
```

A few details that get this wrong in practice:

- **The `time.Sleep` between flipping readiness off and calling `Shutdown` is essential.** Kubernetes has no synchronization between "readiness probe fails" and "Service stops sending traffic"; there's a window of seconds while kube-proxy iptables rules update across the cluster. Without the sleep, you `Shutdown` while the LB is still sending you requests, and they get connection-refused.
- **The shutdown deadline plus the readiness-drain sleep must be less than `terminationGracePeriodSeconds`.** With 5s sleep + 20s shutdown = 25s, and the default 30s grace period, you have 5s of slack. If `Shutdown` blocks past the grace period, the kubelet sends `SIGKILL` and pending requests die mid-flight.
- **`http.Server.Shutdown` does not interrupt active handlers.** It stops accepting new connections and waits for active ones to finish. If a handler runs for longer than your shutdown deadline, the deadline expires, `Shutdown` returns an error, the handler keeps running, and `SIGKILL` eventually kills it. Long-running handlers should respect `ctx.Done()` and bail out when shutdown starts.

For the kubelet side of this — the preStop hook, the `terminationGracePeriodSeconds` field, and the timing of probe state updates — see [the Kubernetes pod lifecycle documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) and §10 below.

A `preStop` hook in the pod spec can give you extra time before SIGTERM (useful for things like external load balancer deregistration), but for in-cluster services the readiness-flip-then-sleep pattern is sufficient.

## 8. Liveness vs readiness vs startup probes

The three probes that Kubernetes supports, and what they should each check:

| Probe | What kubelet does on failure | What you should check | What you must NOT check |
|---|---|---|---|
| **Liveness** | Restart the container | "Is the process responsive?" — accept TCP, return 200 | DB, Redis, downstream services |
| **Readiness** | Remove from Service endpoints | "Am I ready to serve traffic?" — DB connection pool ready, warmup done | Long-running checks that block the response |
| **Startup** | Restart the container during startup; suppresses liveness/readiness until passing | "Has initial setup completed?" — slow migrations, cache warmup | Anything that should be checked steady-state |

The most important rule, and the one that's most often violated:

> **The liveness probe must NOT exercise dependencies.**

A liveness probe that hits the database makes "the database is down for 30 seconds" become "every Go pod gets restarted, the database's recovery is now blocked behind 100 simultaneous reconnect storms, and the service is down for 5 minutes instead of 30 seconds." This is the canonical bad-probe failure mode.

Correct shapes:

```go
// Liveness — process is alive, that's it.
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
})

// Readiness — am I prepared to serve traffic? Check internal state, not externals.
mux.HandleFunc("/readyz", func(w http.ResponseWriter, r *http.Request) {
    if !warmedUp.Load() {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    if shuttingDown.Load() {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    // optionally: check that the local DB connection pool has any non-broken conns
    w.WriteHeader(http.StatusOK)
})
```

Pod spec:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  periodSeconds: 5
  failureThreshold: 1

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 5
  failureThreshold: 60   # up to 5 minutes of startup
```

The startup probe is what you use for Go services that load a large dataset on boot or run migrations. It defers liveness checks until the startup probe succeeds the first time, then never runs again. Set `failureThreshold` × `periodSeconds` to your worst-case startup time.

## 9. Resource requests and limits sizing

Sizing for Go services has a few specifics worth knowing.

**CPU requests.** Set `resources.requests.cpu` to your typical CPU usage, not your peak. Kubernetes uses the request for scheduling (which node the pod lands on) and for proportional CPU shares under contention. A service that averages 100m CPU but spikes to 500m should request 100–200m, not 500m. Over-requesting wastes cluster capacity.

**CPU limits.** **Be careful.** A `cpu: "2"` limit translates to a CFS bandwidth limit that **throttles** the pod when it tries to use more than 2 cores' worth of CPU. The throttling happens in 100ms windows by default — if your pod uses all its quota in 30ms, it sits idle for 70ms before the next window. For latency-sensitive Go services, this manifests as periodic tail-latency spikes that look mysterious in pprof. Many teams run Go services with **no CPU limit** (just a request) for this reason; check your cluster's policy. If you do set a limit, set GOMAXPROCS via automaxprocs to match.

**Memory requests.** Set to your typical RSS plus some headroom. Memory is not compressible; over-provisioning here is less wasteful than over-provisioning CPU, because if the pod isn't using the memory the kernel will use it for page cache.

**Memory limits.** Set tightly. Pods that exceed their memory limit are OOM-killed by the kernel. Pair with `GOMEMLIMIT` set to ~90% of the pod limit so the GC starts working harder before the kernel kills you.

A reasonable starting point for a small HTTP service:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    memory: "256Mi"
    # cpu deliberately omitted — see note above
env:
  - name: GOMEMLIMIT
    value: "230MiB"
```

Then watch metrics, and tune. Right-sizing is a feedback loop, not a guess.

## 10. Pod lifecycle and the rolling update sequence

The full sequence Kubernetes runs when you `kubectl apply` a Deployment update, from the perspective of one old pod being replaced:

```text
T+0       New ReplicaSet scaled up; new pod scheduled on a node.
T+0       New pod's image pulled; container started; startup probe begins.
T+slow    Startup probe passes → liveness + readiness probes start.
T+ready   Readiness probe passes → new pod added to Service endpoints.

(now there's an old pod and a new pod both serving traffic)

T+0       Old ReplicaSet scaled down; old pod marked Terminating.
T+0       preStop hook executed (if any).
T+0       SIGTERM sent to PID 1 of old pod.
T+0       Old pod's readiness probe should start failing (your /readyz returns 503).
T+~few    kube-proxy / LB notices the endpoint is unhealthy; stops sending new requests.
          (But this propagation is NOT synchronous with the SIGTERM.)
T+drain   In-flight requests finish.
T+exit    Process exits cleanly; container terminated.
T+30s     If process still running: SIGKILL (default terminationGracePeriodSeconds=30s).
```

The thing this picture makes clear: there's a window between SIGTERM and "the load balancer stops sending traffic" where new requests can still arrive. That window is what the readiness-flip-then-sleep pattern in §7 covers. Without it, you drop a few requests every rolling update.

For very-high-RPS services the additional pattern is a `preStop` hook that does the readiness flip + sleep before SIGTERM is sent:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "kill -TERM 1; sleep 5"]
```

But this only works on images that have a shell, which distroless does not. For distroless, a pre-stop HTTP call to your own service can work:

```yaml
lifecycle:
  preStop:
    httpGet:
      path: /admin/begin-shutdown
      port: 8080
```

Or simply rely on signal handling — flip readiness as soon as `SIGTERM` arrives and sleep before calling `Shutdown`, as in §7.

## 11. Logging on stdout in the k8s context

Kubernetes captures container stdout and stderr to per-pod log files (typically `/var/log/pods/...`) which the kubelet then exposes via `kubectl logs` and forwards to whatever log collector you have running (Fluent Bit, Vector, Promtail, the cloud provider's agent). The contract is: **your service writes structured logs to stdout, one JSON object per line, and the cluster handles the rest.**

For Go this is the `slog.NewJSONHandler(os.Stdout, ...)` pattern from the [observability doc](02-observability.md):

```go
handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelInfo,
})
slog.SetDefault(slog.New(handler))

slog.Info("server starting", "port", 8080)
```

A few k8s-specific log discipline points:

- **One JSON object per line.** Multi-line stack traces broken across lines often get parsed into separate log records by the collector. Either keep stack traces on a single line (the slog default with `AddSource` does this), or include them as a string attribute.
- **Log to stdout, not stderr.** Both are captured, but most clusters tag stderr lines as "error severity" by default. Use slog's level field (`"level":"INFO"`) for severity and route everything through stdout to keep the routing predictable.
- **Don't write log files.** Container filesystems are ephemeral, often read-only (§4), and inaccessible to log collectors. Stdout is the only cluster-friendly destination.
- **Include `trace_id` in every log line that runs inside a span.** See the [observability doc](02-observability.md) §9 for the slog handler wrapper that does this.

Once these logs hit your log backend (Loki, Cloud Logging, Elasticsearch), the standard query is "give me all log lines for trace ID X" — which, combined with the OpenTelemetry traces, gives you full per-request correlation.

## Related

- [Goroutines & the Scheduler (G-M-P)](../concurrency/01-goroutines-and-scheduler.md) — why GOMAXPROCS matters for the scheduler
- [Slices, Arrays & Maps](../fundamentals/04-slices-arrays-maps.md) — allocation patterns that affect GOMEMLIMIT pacing
- [The Garbage Collector](../runtime/01-garbage-collector.md) — how GOMEMLIMIT influences GC pacing
- [Profiling Go Services with pprof](01-profiling-with-pprof.md) — diagnosing CPU throttling and memory pressure in containers
- [Observability for Go Services](02-observability.md) — slog, OpenTelemetry, trace propagation
- [Kubernetes — INDEX](../../kubernetes/INDEX.md) — pod lifecycle, networking, security in depth
- [TypeScript — Node.js in Kubernetes](../../typescript/production/nodejs-in-kubernetes.md) — the Node analogue of this doc

## References

- distroless container images — https://github.com/GoogleContainerTools/distroless
- `uber-go/automaxprocs` — https://github.com/uber-go/automaxprocs
- Go 1.19 release notes (`GOMEMLIMIT`) — https://go.dev/doc/go1.19
- `runtime/debug.SetMemoryLimit` — https://pkg.go.dev/runtime/debug#SetMemoryLimit
- `net/http.Server.Shutdown` — https://pkg.go.dev/net/http#Server.Shutdown
- `os/signal.NotifyContext` (Go 1.16+) — https://pkg.go.dev/os/signal#NotifyContext
- Kubernetes pod lifecycle — https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
- Kubernetes liveness, readiness, and startup probes — https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Kubernetes resource management for pods and containers — https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
- Pod Security Standards — https://kubernetes.io/docs/concepts/security/pod-security-standards/
- The Go GC Guide — https://go.dev/doc/gc-guide
- Go proposal #73193 (cgroup-aware GOMAXPROCS) — https://github.com/golang/go/issues/73193
