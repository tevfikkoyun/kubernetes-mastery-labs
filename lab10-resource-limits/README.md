# Lab 10 — Resource Limits and Requests

## Goal

Understand how Kubernetes manages CPU and memory resources through Requests and Limits, observe OOMKilled behavior when a container exceeds its memory limit, and set namespace-level defaults with LimitRange.

---

## The Problem

Without resource constraints, a single misbehaving Pod can consume all available CPU or memory on a Node — starving every other Pod running on it. Kubernetes solves this with two complementary mechanisms: Requests (for scheduling) and Limits (for runtime enforcement).

In Docker Lab 12, we used `--memory=128m --cpus=0.5` — a single setting that both reserved and capped resources. Kubernetes splits this into two separate concepts for more fine-grained control.

---

## Requests vs Limits

| | Request | Limit |
|---|---|---|
| Purpose | Minimum guaranteed resources | Maximum allowed resources |
| Used by | `kube-scheduler` (placement decisions) | Kernel (runtime enforcement) |
| CPU behavior when exceeded | N/A (can't exceed a request, only a limit) | Throttled (slowed down) |
| Memory behavior when exceeded | N/A | OOMKilled (process killed immediately) |

```
Request → "I need at least this much to run"
           kube-scheduler uses this to find a Node with enough free capacity

Limit   → "I must never use more than this"
           kernel enforces this at runtime — no exceptions
```

A Pod is only scheduled to a Node if the Node has enough **unallocated** capacity to satisfy the Pod's requests. Limits can be higher than requests — the Pod can "burst" above its request up to the limit if the Node has spare capacity.

---

## CPU and Memory units

**CPU:**
- `m` = millicpu (1/1000th of a core)
- `250m` = 0.25 cores (one quarter)
- `500m` = 0.5 cores (one half)
- `1000m` = 1 full core

**Memory:**
- `Mi` = Mebibytes (1Mi = 1,048,576 bytes)
- `Gi` = Gibibytes
- `128Mi` ≈ 134MB

---

## What I did

### Part 1 — Pod with resource requests and limits

```yaml
# lab10-resources.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab10-pod
spec:
  containers:
    - name: app
      image: nginx:alpine
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

Verified via:
```
kubectl describe pod lab10-pod
# Requests: cpu: 250m, memory: 64Mi
# Limits:   cpu: 500m, memory: 128Mi

kubectl get pod lab10-pod -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"cpu":"500m","memory":"128Mi"},"requests":{"cpu":"250m","memory":"64Mi"}}
```

Note: `kubectl top pod` was unavailable in this environment (Docker Desktop K8s requires `metrics-server` to be installed separately, unlike Minikube where it's a built-in addon). The resource definitions are enforced by the kernel regardless of whether metrics are visible.

### Part 2 — OOMKilled: memory limit exceeded

```yaml
# lab10-oom.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab10-oom
spec:
  containers:
    - name: memory-hog
      image: alpine
      command: ["sh", "-c", "cat /dev/zero | head -c 200m | tail"]
      resources:
        limits:
          memory: "50Mi"
```

The container attempted to allocate 200MB but the limit was 50MB.

```
kubectl apply -f lab10-oom.yaml
kubectl get pods --watch
```

Result:
```
lab10-oom   0/1   OOMKilled   1 (2s ago)   3s
```

The Linux kernel's OOM killer terminated the process immediately when it exceeded the 50Mi limit. K8s then restarted it (because no liveness probe was defined to give up) — it would eventually reach `CrashLoopBackOff`. This is the kernel-level enforcement of the same limit we verified with `docker stats` in Docker Lab 12 (`MEM USAGE / LIMIT: 13.15MiB / 128MiB`).

### Part 3 — LimitRange: namespace-level defaults

```yaml
# lab10-limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: lab10-limits
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "128Mi"
      defaultRequest:
        cpu: "250m"
        memory: "64Mi"
      max:
        cpu: "1000m"
        memory: "512Mi"
```

```
kubectl apply -f lab10-limitrange.yaml
kubectl describe limitrange lab10-limits
```

Result:
```
Type       Resource  Max    Default Request  Default Limit
Container  cpu       1      250m             500m
Container  memory    512Mi  64Mi             128Mi
```

With LimitRange in place, any new Pod in the namespace that doesn't define its own resources automatically inherits these defaults. No developer can "forget" to set limits. The `max` field also prevents any single Pod from claiming more than 1 CPU or 512Mi memory.

### Cleanup

```
kubectl delete pod lab10-pod lab10-oom
kubectl delete limitrange lab10-limits
```

---

## Docker Lab 12 → Kubernetes comparison

| Docker (Lab 12) | Kubernetes (Lab 10) |
|---|---|
| `docker run --memory=128m` | `limits.memory: 128Mi` |
| `docker run --cpus=0.5` | `limits.cpu: 500m` |
| `docker stats` (live usage) | `kubectl top pod` (requires metrics-server) |
| Single `--memory` value | Separate `requests` (scheduler) and `limits` (runtime) |
| No namespace defaults | `LimitRange` sets defaults and maximums per namespace |

---

## Key concepts

| Concept | Description |
|---|---|
| **Request** | Minimum resources reserved for scheduling; Node must have this much free |
| **Limit** | Maximum resources the container can use; enforced by the kernel |
| **OOMKilled** | Exit reason when a container exceeds its memory limit |
| **CPU throttling** | When CPU usage hits the limit, the container is slowed — not killed |
| **LimitRange** | Namespace-level policy that sets default requests/limits and enforces maximums |

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Create a Pod with resource constraints |
| `kubectl describe pod <name>` | Show Requests and Limits in the container spec |
| `kubectl get pod <name> -o jsonpath='{.spec.containers[0].resources}'` | Extract resource config as JSON |
| `kubectl top pod <name>` | Show live CPU/memory usage (requires metrics-server) |
| `kubectl describe limitrange <name>` | Show namespace resource policy |
| `kubectl delete limitrange <name>` | Remove namespace resource policy |

## Notes

In production on EKS, every Pod should have both requests and limits defined. Requests that are too low cause Pods to be scheduled on overloaded nodes; limits that are too low cause unexpected OOMKills. A LimitRange in each namespace ensures that Pods without explicit resource definitions still get sensible constraints rather than running unbounded.
