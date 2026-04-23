---
date: 2026-04-24
title: "kubectl Mastery — Debugging, Introspection, and Productivity"
doc_number: 16
tier: 6
tags: [kubernetes, kubectl, debugging, operations, productivity]
audience: TypeScript/Node + Java/Spring Boot backend developer
---

# kubectl Mastery — Debugging, Introspection, and Productivity

**Doc 16** in the Kubernetes learning path. Covers the commands and techniques that separate "I can run `kubectl get pods`" from "I can diagnose a production incident in five minutes." Output formatting, filtering, ephemeral debug containers, resource introspection, context management, and the plugin ecosystem.

**Key subtopics:** Essential commands (get, describe, logs, exec, port-forward, apply, diff, delete), output formatting (jsonpath, custom-columns), label and field selectors, ephemeral containers with `kubectl debug`, resource introspection (explain, api-resources, top, events), context/namespace management (kubectx/kubens), productivity (krew plugins, shell completion, aliases, wait, diff).

---

## Table of Contents

- [Summary](#summary)
- [Essential Commands](#essential-commands)
  - [get — The Starting Point](#get--the-starting-point)
  - [describe — Rich Human-Readable Detail](#describe--rich-human-readable-detail)
  - [logs — Container Output](#logs--container-output)
  - [exec — Shell Into a Container](#exec--shell-into-a-container)
  - [port-forward — Local Access to Cluster Services](#port-forward--local-access-to-cluster-services)
  - [apply and diff — Declarative Changes](#apply-and-diff--declarative-changes)
  - [delete — Removing Resources](#delete--removing-resources)
- [Output Formatting](#output-formatting)
  - [Full Object Dumps](#full-object-dumps)
  - [JSONPath — Extracting Specific Fields](#jsonpath--extracting-specific-fields)
  - [Custom Columns — Tabular Custom Output](#custom-columns--tabular-custom-output)
  - [Resource Names for Scripting](#resource-names-for-scripting)
  - [Sorting Output](#sorting-output)
- [Filtering — Labels and Field Selectors](#filtering--labels-and-field-selectors)
  - [Label Selectors](#label-selectors)
  - [Field Selectors](#field-selectors)
  - [Combining Label and Field Selectors](#combining-label-and-field-selectors)
- [Debugging with Ephemeral Containers](#debugging-with-ephemeral-containers)
  - [Why Ephemeral Containers Exist](#why-ephemeral-containers-exist)
  - [Attach a Debug Container to a Running Pod](#attach-a-debug-container-to-a-running-pod)
  - [Debug Node-Level Issues](#debug-node-level-issues)
  - [Copy-Based Debugging](#copy-based-debugging)
- [Resource Introspection](#resource-introspection)
  - [explain — Built-In Field Documentation](#explain--built-in-field-documentation)
  - [api-resources and api-versions — What Exists in This Cluster](#api-resources-and-api-versions--what-exists-in-this-cluster)
  - [top — Resource Usage](#top--resource-usage)
  - [events — What Happened Recently](#events--what-happened-recently)
- [Context and Namespace Management](#context-and-namespace-management)
  - [Built-In Context Commands](#built-in-context-commands)
  - [kubectx and kubens — Fast Switching](#kubectx-and-kubens--fast-switching)
- [Productivity](#productivity)
  - [Krew — kubectl Plugin Manager](#krew--kubectl-plugin-manager)
  - [Shell Completion](#shell-completion)
  - [Aliases](#aliases)
  - [diff — Preview Before Apply](#diff--preview-before-apply)
  - [wait — Blocking Until a Condition](#wait--blocking-until-a-condition)
- [Spring Boot and Node.js Debugging Patterns](#spring-boot-and-nodejs-debugging-patterns)
- [Related](#related)
- [References](#references)

---

## Summary

`kubectl` is your primary interface to Kubernetes. The API server is the single source of truth, and kubectl is the CLI that talks to it. Most developers plateau at `get`, `apply`, and `logs` — but the tool is far deeper. JSONPath expressions, ephemeral debug containers, and the krew plugin ecosystem turn kubectl from a basic CRUD tool into a full diagnostic and productivity platform. Mastering output formatting and selectors alone can cut investigation time in half because you stop scrolling through walls of YAML and start extracting exactly the fields you need.

---

## Essential Commands

### get — The Starting Point

`get` lists resources. The default output is a condensed table. Add flags to control what you see:

```bash
# Basic pod listing
kubectl get pods

# Wide output — shows node, IP, nominated node, readiness gates
kubectl get pods -o wide

# Full YAML for a specific pod — the complete API object
kubectl get pod my-app-7b9d5c6f4d-x2k9p -o yaml

# Full JSON — useful for piping to jq
kubectl get pod my-app-7b9d5c6f4d-x2k9p -o json

# All resource types in a namespace (not just pods)
kubectl get all -n staging

# Multiple resource types in one call
kubectl get pods,services,deployments -n production
```

> **Tip:** `get all` does not actually return all resources. It misses ConfigMaps, Secrets, Ingresses, NetworkPolicies, and CRDs. Use `kubectl api-resources --verbs=list -o name | xargs -n1 kubectl get -n <ns>` if you truly need everything.

### describe — Rich Human-Readable Detail

`describe` gives a narrative view: events at the bottom (invaluable for diagnosing scheduling or image-pull failures), conditions, volumes, and container states all in one output:

```bash
# Pod details with events
kubectl describe pod my-app-7b9d5c6f4d-x2k9p

# Deployment rollout status and replica details
kubectl describe deployment my-app

# Node capacity, conditions, and allocatable resources
kubectl describe node worker-3

# Service endpoints — which pods are backing it
kubectl describe service my-app-svc
```

The events section at the bottom of `describe` output is the first place to look when something is wrong. Events tell you about image pulls, scheduling decisions, OOMKills, readiness probe failures, and volume mount issues.

### logs — Container Output

```bash
# Tail the last 100 lines
kubectl logs my-app-7b9d5c6f4d-x2k9p --tail=100

# Stream logs in real time (-f = follow)
kubectl logs -f my-app-7b9d5c6f4d-x2k9p

# Logs from a specific container in a multi-container pod
kubectl logs my-app-7b9d5c6f4d-x2k9p -c sidecar-proxy

# Logs from the PREVIOUS container instance (crashed/restarted)
kubectl logs my-app-7b9d5c6f4d-x2k9p --previous

# Logs from all pods matching a label
kubectl logs -l app=my-app --all-containers=true

# Logs since a specific time
kubectl logs my-app-7b9d5c6f4d-x2k9p --since=30m

# Logs with timestamps
kubectl logs my-app-7b9d5c6f4d-x2k9p --timestamps=true
```

> **Critical:** `--previous` retrieves logs from the last terminated container. When a Spring Boot app crashes on startup or a Node.js process exits with an unhandled rejection, the current container might be the new restart — the crash output is in `--previous`.

### exec — Shell Into a Container

```bash
# Interactive shell
kubectl exec -it my-app-7b9d5c6f4d-x2k9p -- /bin/sh

# Specific container in a multi-container pod
kubectl exec -it my-app-7b9d5c6f4d-x2k9p -c main-app -- /bin/bash

# Run a single command without interactive mode
kubectl exec my-app-7b9d5c6f4d-x2k9p -- env

# Check connectivity from inside the pod
kubectl exec my-app-7b9d5c6f4d-x2k9p -- curl -s http://other-service:8080/health

# Inspect the JVM in a Spring Boot container
kubectl exec -it my-app-7b9d5c6f4d-x2k9p -- jcmd 1 VM.flags

# Check Node.js memory usage
kubectl exec my-app-7b9d5c6f4d-x2k9p -- node -e "console.log(process.memoryUsage())"
```

> **Limitation:** `exec` only works if the container image has a shell. Distroless images and scratch-based images do not. That is where ephemeral debug containers come in (covered below).

### port-forward — Local Access to Cluster Services

```bash
# Forward local port 8080 to pod port 8080
kubectl port-forward pod/my-app-7b9d5c6f4d-x2k9p 8080:8080

# Forward to a Service (picks a backing pod automatically)
kubectl port-forward svc/my-app-svc 8080:80

# Forward to a different local port
kubectl port-forward svc/my-app-svc 9090:80

# Forward multiple ports
kubectl port-forward pod/my-app-7b9d5c6f4d-x2k9p 8080:8080 5005:5005

# Access a database for debugging (PostgreSQL)
kubectl port-forward svc/postgres 5432:5432 -n database
```

The second example (`svc/my-app-svc`) is useful when you don't want to look up the exact pod name. Port-forwarding a debug port (like JVM remote debug on 5005 or Node.js `--inspect` on 9229) is how you attach a local debugger to a running cluster workload.

### apply and diff — Declarative Changes

```bash
# Apply a manifest — creates or updates
kubectl apply -f deployment.yaml

# Apply all manifests in a directory
kubectl apply -f manifests/

# Apply from a URL
kubectl apply -f https://raw.githubusercontent.com/org/repo/main/deploy.yaml

# Dry run — show what would be sent to the server without applying
kubectl apply -f deployment.yaml --dry-run=server -o yaml

# Diff before apply — see exactly what will change
kubectl diff -f deployment.yaml
```

`diff` is underused and extremely valuable. It compares what the cluster has with what your manifest contains and shows the delta in unified diff format. Always run `diff` before `apply` in production.

### delete — Removing Resources

```bash
# Delete a specific pod
kubectl delete pod my-app-7b9d5c6f4d-x2k9p

# Delete using a manifest
kubectl delete -f deployment.yaml

# Delete by label
kubectl delete pods -l app=my-app

# Force delete a stuck pod (use sparingly)
kubectl delete pod my-app-7b9d5c6f4d-x2k9p --grace-period=0 --force

# Delete all completed jobs
kubectl delete jobs --field-selector status.successful=1
```

> **Warning:** Force-deleting (`--grace-period=0 --force`) skips the graceful shutdown process. The kubelet may not even have time to run `preStop` hooks or send `SIGTERM`. Use it only when a pod is stuck in `Terminating` state and you understand why.

---

## Output Formatting

### Full Object Dumps

```bash
# YAML — human-readable, preserves comments-like structure
kubectl get deployment my-app -o yaml

# JSON — machine-readable, pipe to jq
kubectl get deployment my-app -o json

# Pipe to jq for targeted extraction
kubectl get deployment my-app -o json | jq '.spec.template.spec.containers[0].resources'
```

### JSONPath — Extracting Specific Fields

JSONPath is built into kubectl. No external tools needed:

```bash
# All container images in a deployment
kubectl get deployment my-app -o jsonpath='{.spec.template.spec.containers[*].image}'

# Pod IPs for all running pods
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Node internal IPs
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\n"}{end}'

# All container images across all pods in a namespace
kubectl get pods -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}'

# Resource requests for a specific pod
kubectl get pod my-app-7b9d5c6f4d-x2k9p -o jsonpath='{.spec.containers[0].resources.requests}'

# Extract the restart count for each container
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.containerStatuses[*]}{.name}={.restartCount}{" "}{end}{"\n"}{end}'
```

**JSONPath syntax cheat sheet:**

| Expression | Meaning |
|---|---|
| `.metadata.name` | Scalar field |
| `.spec.containers[*]` | All items in an array |
| `.spec.containers[0]` | First item |
| `.items[?(@.status.phase=="Running")]` | Filter by condition |
| `{range .items[*]}...{end}` | Iterate and format |
| `{"\t"}`, `{"\n"}` | Tab and newline literals |

### Custom Columns — Tabular Custom Output

When you need a table but the default columns don't show what you want:

```bash
# Pods with their images and node placement
kubectl get pods -o custom-columns=\
NAME:.metadata.name,\
IMAGE:.spec.containers[0].image,\
NODE:.spec.nodeName,\
STATUS:.status.phase

# Deployments with replica counts and strategy
kubectl get deployments -o custom-columns=\
NAME:.metadata.name,\
DESIRED:.spec.replicas,\
READY:.status.readyReplicas,\
STRATEGY:.spec.strategy.type

# Services with type and cluster IP
kubectl get svc -o custom-columns=\
NAME:.metadata.name,\
TYPE:.spec.type,\
CLUSTER-IP:.spec.clusterIP,\
PORTS:.spec.ports[*].port
```

### Resource Names for Scripting

`-o name` returns just `<resource-type>/<name>` — perfect for piping:

```bash
# List pod names only
kubectl get pods -o name

# Delete all failed pods in one shot
kubectl get pods --field-selector status.phase=Failed -o name | xargs kubectl delete

# Restart all deployments in a namespace
kubectl get deployments -o name | xargs -I {} kubectl rollout restart {}
```

### Sorting Output

```bash
# Pods sorted by creation time (newest last)
kubectl get pods --sort-by=.metadata.creationTimestamp

# Pods sorted by restart count (find the crashloop)
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# Events sorted by last timestamp
kubectl get events --sort-by=.lastTimestamp

# PVCs sorted by storage capacity
kubectl get pvc --sort-by=.spec.resources.requests.storage
```

---

## Filtering — Labels and Field Selectors

### Label Selectors

Labels are the primary mechanism for grouping and filtering resources:

```bash
# Equality-based selectors
kubectl get pods -l app=my-app
kubectl get pods -l app=my-app,tier=backend    # AND (comma = intersection)
kubectl get pods -l 'app!=legacy'

# Set-based selectors
kubectl get pods -l 'tier in (frontend,backend)'
kubectl get pods -l 'environment notin (dev,staging)'
kubectl get pods -l 'release'                  # has the label (any value)
kubectl get pods -l '!canary'                  # does NOT have the label

# Combine multiple set-based conditions
kubectl get pods -l 'app in (web,api),tier=backend'

# Use with other commands
kubectl delete pods -l app=my-app,version=v1
kubectl logs -l app=my-app --all-containers=true
```

> **Pattern:** For Spring Boot microservices, common labels include `app.kubernetes.io/name`, `app.kubernetes.io/version`, `app.kubernetes.io/component`. For Node.js services, the same convention applies. Use these standard labels so that Helm, ArgoCD, and monitoring tools can discover your resources consistently.

### Field Selectors

Field selectors filter on the object's actual spec/status fields (not labels). The set of supported fields is limited:

```bash
# Only running pods
kubectl get pods --field-selector status.phase=Running

# Pods NOT in Succeeded phase
kubectl get pods --field-selector status.phase!=Succeeded

# Find a specific pod by name
kubectl get pods --field-selector metadata.name=my-app-7b9d5c6f4d-x2k9p

# Pods on a specific node
kubectl get pods --field-selector spec.nodeName=worker-3

# Events from a specific namespace (field selector, not flag)
kubectl get events --field-selector involvedObject.namespace=production

# Events for a specific object
kubectl get events --field-selector involvedObject.name=my-app-7b9d5c6f4d-x2k9p
```

**Commonly supported field selectors:**

| Resource | Fields |
|---|---|
| Pods | `metadata.name`, `metadata.namespace`, `spec.nodeName`, `spec.restartPolicy`, `spec.schedulerName`, `spec.serviceAccountName`, `status.phase`, `status.podIP`, `status.nominatedNodeName` |
| Events | `involvedObject.kind`, `involvedObject.namespace`, `involvedObject.name`, `involvedObject.uid`, `reason`, `type` |
| Nodes | `metadata.name`, `spec.unschedulable` |
| All resources | `metadata.name`, `metadata.namespace` |

### Combining Label and Field Selectors

Label and field selectors can be used together for precise filtering:

```bash
# Running pods with a specific label
kubectl get pods -l app=my-app --field-selector status.phase=Running

# Backend pods on a specific node
kubectl get pods -l tier=backend --field-selector spec.nodeName=worker-3

# Failed pods for a specific app — useful for incident response
kubectl get pods -l app=payment-service --field-selector status.phase=Failed
```

---

## Debugging with Ephemeral Containers

### Why Ephemeral Containers Exist

Modern container images are minimal. Distroless Java images have no shell. Node.js distroless images have no `curl`, no `dig`, no `strace`. When your Spring Boot app is behaving oddly in production, `kubectl exec` is useless if there is nothing to exec into.

Ephemeral containers (GA since Kubernetes 1.25) solve this by letting you inject a temporary debug container into a running pod without restarting it, without modifying the Deployment, and without disrupting traffic. The debug container shares the pod's network namespace (same `localhost`, same port access) and can optionally share the process namespace.

### Attach a Debug Container to a Running Pod

```bash
# Basic: attach a busybox debug container
kubectl debug -it pod/my-app-7b9d5c6f4d-x2k9p --image=busybox

# Share process namespace with the target container
# This lets you see processes in the app container with ps/top
kubectl debug -it pod/my-app-7b9d5c6f4d-x2k9p \
  --image=busybox \
  --target=main-container

# Use a full debug image with networking tools
kubectl debug -it pod/my-app-7b9d5c6f4d-x2k9p \
  --image=nicolaka/netshoot \
  --target=main-container

# Debug a CrashLoopBackOff pod (override the entrypoint)
kubectl debug -it pod/my-app-7b9d5c6f4d-x2k9p \
  --image=busybox \
  -- /bin/sh
```

**What `--target` does:** Without `--target`, the ephemeral container shares the pod's network namespace but has its own PID namespace. With `--target=main-container`, it also shares the PID namespace of the specified container, so `ps aux` will show the app's Java or Node.js processes.

**Practical debug session for a Spring Boot app:**

```bash
# Attach netshoot to debug networking
kubectl debug -it pod/payment-svc-6f8d4a7c9b-m3k2p \
  --image=nicolaka/netshoot \
  --target=app

# Inside the ephemeral container:
# Check DNS resolution
nslookup postgres-svc.database.svc.cluster.local

# Test connectivity to a database
nc -zv postgres-svc.database.svc.cluster.local 5432

# Trace HTTP calls
curl -v http://order-service.production.svc.cluster.local:8080/health

# Check process list (requires --target)
ps aux | grep java
```

### Debug Node-Level Issues

`kubectl debug` can also create a privileged pod on a specific node for host-level debugging:

```bash
# Start a debug pod on a node with host access
kubectl debug node/worker-3 --image=ubuntu

# This mounts the host filesystem at /host
# Inside the debug pod:
ls /host/var/log/
cat /host/var/log/syslog
chroot /host          # enter the node's root filesystem
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago"
crictl ps             # list containers via CRI
```

> **Security note:** Node debugging creates a privileged pod. This bypasses Pod Security Standards. Use it only in environments where you have cluster-admin access and only for active debugging.

### Copy-Based Debugging

Create a copy of a pod with modifications — useful when you want to change the command, add env vars, or swap the image without affecting the running pod:

```bash
# Copy the pod with a different container image
kubectl debug pod/my-app-7b9d5c6f4d-x2k9p \
  --copy-to=debug-pod \
  --container=app \
  --image=my-app:debug

# Copy and change the command (useful for CrashLoopBackOff)
kubectl debug pod/my-app-7b9d5c6f4d-x2k9p \
  --copy-to=debug-pod \
  --container=app \
  -- sleep 3600

# Copy with an additional debug container
kubectl debug pod/my-app-7b9d5c6f4d-x2k9p \
  --copy-to=debug-pod \
  --container=debug \
  --image=busybox

# Copy and share process namespaces
kubectl debug pod/my-app-7b9d5c6f4d-x2k9p \
  --copy-to=debug-pod \
  --share-processes
```

**When to use copy-based debugging:** The pod is in CrashLoopBackOff and restarts too fast to `exec` into. Copying with `-- sleep 3600` as the command gives you a stable container with the same volumes, secrets, and environment variables.

---

## Resource Introspection

### explain — Built-In Field Documentation

`explain` is the man page for Kubernetes resources. It pulls documentation directly from the OpenAPI schema of your connected cluster — so it works for CRDs too, not just built-in resources:

```bash
# Top-level fields of a Deployment
kubectl explain deployment

# Drill into nested fields with dot notation
kubectl explain deployment.spec.strategy
kubectl explain deployment.spec.strategy.rollingUpdate

# Pod spec containers
kubectl explain pod.spec.containers

# Explain with recursive output — shows the full tree
kubectl explain deployment.spec --recursive

# Works for CRDs too (if installed in the cluster)
kubectl explain certificate.spec           # cert-manager CRD
kubectl explain virtualservice.spec        # Istio CRD
```

> **Tip:** When you are writing YAML manifests and cannot remember a field name, `explain --recursive` dumps the entire structure. Pipe it to `grep` to find what you need: `kubectl explain pod.spec --recursive | grep -i affinity`.

### api-resources and api-versions — What Exists in This Cluster

```bash
# List all resource types — name, shortnames, API group, namespaced, kind
kubectl api-resources

# Filter to namespaced resources only
kubectl api-resources --namespaced=true

# Filter by API group
kubectl api-resources --api-group=apps

# Show verbs — which operations are supported
kubectl api-resources -o wide

# Short names cheat sheet
kubectl api-resources | grep -E "^NAME|deploy|svc|cm|pvc|ing|ns|no "

# List all API group/versions (useful for checking what's available)
kubectl api-versions
```

**Common short names you should know:**

| Short Name | Full Resource |
|---|---|
| `po` | pods |
| `deploy` | deployments |
| `svc` | services |
| `cm` | configmaps |
| `pvc` | persistentvolumeclaims |
| `ing` | ingresses |
| `ns` | namespaces |
| `no` | nodes |
| `sa` | serviceaccounts |
| `rs` | replicasets |
| `ds` | daemonsets |
| `sts` | statefulsets |
| `hpa` | horizontalpodautoscalers |
| `pdb` | poddisruptionbudgets |
| `ep` | endpoints |

### top — Resource Usage

Requires the [metrics-server](https://github.com/kubernetes-sigs/metrics-server) to be deployed in the cluster:

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage in current namespace
kubectl top pods

# Pod resource usage sorted by CPU
kubectl top pods --sort-by=cpu

# Pod resource usage sorted by memory
kubectl top pods --sort-by=memory

# Container-level resource usage within pods
kubectl top pods --containers

# All namespaces
kubectl top pods -A --sort-by=memory
```

**Reading the output:**

```
NAME                      CPU(cores)   MEMORY(bytes)
my-app-7b9d5c6f4d-x2k9p  250m         512Mi
my-app-7b9d5c6f4d-r8h3q  180m         487Mi
```

- `250m` = 250 millicores = 0.25 CPU cores
- `512Mi` = 512 mebibytes

Compare these values against the `requests` and `limits` in your pod spec (see [Resource Management](../configuration/resource-management.md)) to understand if your pods are right-sized or wasting resources.

### events — What Happened Recently

Events are the cluster's activity log. They explain scheduling decisions, image pull failures, OOMKills, and probe failures:

```bash
# All events in the current namespace, sorted by time
kubectl get events --sort-by=.lastTimestamp

# All events cluster-wide
kubectl get events -A --sort-by=.lastTimestamp

# Warning events only
kubectl get events --field-selector type=Warning

# Events for a specific pod
kubectl get events --field-selector involvedObject.name=my-app-7b9d5c6f4d-x2k9p

# Watch events in real time
kubectl get events -w

# Events in the last hour (combine with sort)
kubectl get events --sort-by=.lastTimestamp | tail -50
```

> **Retention:** Events are stored in etcd and default to a 1-hour TTL. For persistent event history, deploy an event exporter (like [Kubernetes Event Exporter](https://github.com/resmoio/kubernetes-event-exporter)) that ships events to your logging stack.

---

## Context and Namespace Management

### Built-In Context Commands

A **context** groups a cluster, user, and default namespace. Your kubeconfig (default: `~/.kube/config`) can hold many contexts:

```bash
# List all contexts — shows cluster, user, and namespace
kubectl config get-contexts

# See the current context
kubectl config current-context

# Switch to a different context
kubectl config use-context staging-cluster

# Set the default namespace for the current context
kubectl config set-context --current --namespace=production

# View the full kubeconfig
kubectl config view

# View kubeconfig with secrets redacted (default behavior)
kubectl config view --minify
```

**Practical workflow:**

```bash
# Morning: switch to the production cluster
kubectl config use-context prod-us-east

# Set namespace to the team's namespace
kubectl config set-context --current --namespace=payments

# Now all commands default to the payments namespace in prod
kubectl get pods    # → pods in payments namespace, prod cluster
```

### kubectx and kubens — Fast Switching

Install via krew or package manager. These tools eliminate the verbose `kubectl config` commands:

```bash
# Install via krew
kubectl krew install ctx
kubectl krew install ns

# Switch context (interactive fuzzy selector if fzf is installed)
kubectx staging-cluster

# Switch namespace
kubens production

# Previous context (like cd -)
kubectx -

# Previous namespace
kubens -

# List all contexts
kubectx

# List all namespaces
kubens
```

> **Why this matters:** In a typical backend developer workflow, you switch between dev, staging, and prod clusters dozens of times a day. Typing `kubectl config use-context gke_my-project_us-east1_prod-cluster` every time is an error-prone tax. `kubectx prod` solves that.

---

## Productivity

### Krew — kubectl Plugin Manager

[Krew](https://krew.sigs.k8s.io/) is the plugin manager for kubectl. It provides access to 200+ community-maintained plugins:

```bash
# Install krew (macOS/Linux)
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

# Add to PATH (add to .zshrc / .bashrc)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Verify
kubectl krew version

# Search for plugins
kubectl krew search

# Install a plugin
kubectl krew install ctx

# Update plugin index
kubectl krew update

# Upgrade all plugins
kubectl krew upgrade
```

**Essential plugins for backend developers:**

| Plugin | Install | Purpose |
|---|---|---|
| `ctx` | `kubectl krew install ctx` | Fast context switching (kubectx) |
| `ns` | `kubectl krew install ns` | Fast namespace switching (kubens) |
| `neat` | `kubectl krew install neat` | Remove clutter (managedFields, status, defaults) from YAML output |
| `tree` | `kubectl krew install tree` | Visualize owner-dependent resource hierarchy |
| `who-can` | `kubectl krew install who-can` | Show which subjects can perform an action on a resource |
| `deprecations` | `kubectl krew install deprecations` | Find deprecated API usage before cluster upgrades |
| `images` | `kubectl krew install images` | List all container images running in the cluster |
| `stern` | `kubectl krew install stern` | Multi-pod/container log tailing with color |
| `resource-capacity` | `kubectl krew install resource-capacity` | Node-level resource request/limit summary |

**Plugin usage examples:**

```bash
# neat — clean YAML output (removes managedFields, last-applied-config, status)
kubectl get deployment my-app -o yaml | kubectl neat

# tree — show the ownership tree for a Deployment
kubectl tree deployment my-app
# NAMESPACE  NAME                              READY  REASON  AGE
# default    Deployment/my-app                 -              3d
# default    └─ReplicaSet/my-app-7b9d5c6f4d   -              3d
# default      ├─Pod/my-app-7b9d5c6f4d-x2k9p  True           3d
# default      └─Pod/my-app-7b9d5c6f4d-r8h3q  True           3d

# who-can — RBAC investigation
kubectl who-can create deployments -n production
kubectl who-can delete pods -n staging
kubectl who-can get secrets -n kube-system

# deprecations — pre-upgrade check
kubectl deprecations --k8s-version=v1.31
# Shows any resources using deprecated or removed APIs in v1.31

# stern — tail logs from multiple pods
stern my-app -n production                    # all pods matching "my-app"
stern -l app=my-app --since 5m               # label selector, last 5 minutes
stern my-app -c main-app                     # specific container
```

### Shell Completion

Tab completion for kubectl is essential. It completes resource types, resource names, flags, and namespaces:

```bash
# Bash — add to ~/.bashrc
source <(kubectl completion bash)
# If kubectl is aliased to k:
complete -o default -F __start_kubectl k

# Zsh — add to ~/.zshrc
source <(kubectl completion zsh)
# If you get "command not found: compdef", add this first:
autoload -Uz compinit && compinit

# Verify
kubectl get pod<TAB>    # completes pod names
kubectl -n <TAB>        # completes namespace names
```

### Aliases

These aliases are widely adopted in the Kubernetes community:

```bash
# Add to ~/.zshrc or ~/.bashrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kgn='kubectl get nodes'
alias kge='kubectl get events --sort-by=.lastTimestamp'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kx='kubectl exec -it'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'
alias kns='kubectl config set-context --current --namespace'
alias kcx='kubectl config use-context'

# Quick pod status check
alias kgpw='kubectl get pods -o wide'
alias kgpa='kubectl get pods -A'
```

> **Alias vs kubectx/kubens:** Aliases like `kns` and `kcx` are zero-dependency alternatives to kubectx/kubens. If you prefer not to install plugins, simple aliases cover the most common switching needs.

### diff — Preview Before Apply

```bash
# See what would change before applying
kubectl diff -f deployment.yaml

# Diff an entire directory
kubectl diff -f manifests/

# Diff a kustomize overlay
kubectl diff -k overlays/staging/
```

The output is a unified diff showing additions (+), removals (-), and context lines. Fields like `metadata.managedFields` and `metadata.resourceVersion` will show up in the diff — focus on the spec changes.

**Practical pattern — safe production deploys:**

```bash
# Step 1: Preview
kubectl diff -f deployment.yaml

# Step 2: Review the diff output carefully

# Step 3: Apply if the diff looks correct
kubectl apply -f deployment.yaml

# Step 4: Watch the rollout
kubectl rollout status deployment/my-app
```

### wait — Blocking Until a Condition

`wait` blocks until a resource reaches a specified condition. Invaluable in CI/CD scripts:

```bash
# Wait for a pod to be ready
kubectl wait --for=condition=Ready pod/my-app-7b9d5c6f4d-x2k9p --timeout=120s

# Wait for all pods with a label to be ready
kubectl wait --for=condition=Ready pods -l app=my-app --timeout=120s

# Wait for a deployment rollout to complete
kubectl wait --for=condition=Available deployment/my-app --timeout=180s

# Wait for a job to complete
kubectl wait --for=condition=Complete job/db-migration --timeout=300s

# Wait for a pod to be deleted
kubectl wait --for=delete pod/my-app-7b9d5c6f4d-x2k9p --timeout=60s

# CI/CD pipeline example (bash)
kubectl apply -f manifests/
kubectl wait --for=condition=Available deployment/my-app --timeout=180s
if [ $? -ne 0 ]; then
  echo "Deployment failed to become available"
  kubectl rollout undo deployment/my-app
  exit 1
fi
```

---

## Spring Boot and Node.js Debugging Patterns

**Spring Boot on Kubernetes:**

```bash
# Check if actuator health endpoint is responding
kubectl exec pod/payment-svc-xyz -- curl -s localhost:8080/actuator/health | jq .

# JVM heap dump (if jcmd is present in the image)
kubectl exec pod/payment-svc-xyz -- jcmd 1 GC.heap_dump /tmp/heapdump.hprof
kubectl cp payment-svc-xyz:/tmp/heapdump.hprof ./heapdump.hprof

# Thread dump
kubectl exec pod/payment-svc-xyz -- jcmd 1 Thread.print

# Remote debug (add JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005)
kubectl port-forward pod/payment-svc-xyz 5005:5005
# Then attach IntelliJ/VS Code debugger to localhost:5005

# Environment variables (verify Spring profiles and config)
kubectl exec pod/payment-svc-xyz -- env | grep SPRING
```

**Node.js on Kubernetes:**

```bash
# Check if the health endpoint responds
kubectl exec pod/api-gateway-xyz -- curl -s localhost:3000/health

# Memory and event loop stats
kubectl exec pod/api-gateway-xyz -- node -e "
  const v8 = require('v8');
  console.log(JSON.stringify(v8.getHeapStatistics(), null, 2));
"

# Remote debug (start with --inspect=0.0.0.0:9229)
kubectl port-forward pod/api-gateway-xyz 9229:9229
# Then attach Chrome DevTools or VS Code debugger

# Check for open handles keeping the process alive
kubectl exec pod/api-gateway-xyz -- node -e "process._getActiveHandles().forEach(h => console.log(h.constructor.name))"

# When the image is distroless — use ephemeral containers
kubectl debug -it pod/api-gateway-xyz \
  --image=nicolaka/netshoot \
  --target=app \
  -- curl -s localhost:3000/health
```

---

## Related

- [Helm and Kustomize — Packaging and Templating Kubernetes Manifests](helm-and-kustomize.md) — managing manifests at scale with templating and overlays
- [Kubernetes Monitoring and Logging](monitoring-and-logging.md) — Prometheus, Grafana, Fluent Bit for ongoing observability beyond one-off debugging
- [Resource Requests, Limits, QoS Classes, and LimitRanges](../configuration/resource-management.md) — understanding the numbers from `kubectl top`
- [RBAC, ServiceAccounts, and Identity](../security/rbac-and-service-accounts.md) — the authorization model behind `who-can`
- [Pod Security Standards](../security/pod-security.md) — security constraints relevant to debug container privileges
- [Developer Experience on Kubernetes](../production/developer-experience.md) — local dev tooling (kind, Tilt, Skaffold) for inner-loop productivity

---

## References

- [kubectl Reference Documentation](https://kubernetes.io/docs/reference/kubectl/) — official command reference
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) — quick reference for common commands
- [Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/) — official guide to ephemeral containers and kubectl debug
- [Ephemeral Containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/) — concept documentation
- [JSONPath Support](https://kubernetes.io/docs/reference/kubectl/jsonpath/) — JSONPath syntax reference for kubectl
- [Krew — kubectl Plugin Manager](https://krew.sigs.k8s.io/) — plugin discovery and installation
