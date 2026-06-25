# Lab 09 — Liveness and Readiness Probes

## Goal

Understand the difference between Liveness and Readiness Probes, observe their behavior in practice, and connect them to the health check patterns established in Docker Lab 12.

---

## The Problem

A running container is not necessarily a healthy container. Two distinct failure modes exist:

1. **The container is stuck** — the process is running but deadlocked, hanging, or in an unrecoverable state. The only fix is a restart.
2. **The container is not ready** — the process started but isn't ready to serve traffic yet (still initializing, waiting for a database connection, warming up a cache).

Kubernetes handles these separately with two different probe types.

---

## Liveness vs Readiness

| | Liveness Probe | Readiness Probe |
|---|---|---|
| Question | "Is the container still alive?" | "Is the container ready to serve traffic?" |
| Failure action | **Restart the container** | **Remove Pod from Service endpoints** (no restart) |
| Use case | Detect deadlocks, hung processes | Detect slow startup, temporary unavailability |
| Docker equivalent | `HEALTHCHECK` in Dockerfile | No direct equivalent (K8s-specific) |

```
Liveness fails  →  container RESTART
Readiness fails →  removed from Service load balancer (stays Running, just gets no traffic)
```

Both can run simultaneously — and should, for production workloads.

---

## Probe types

All three work for both Liveness and Readiness:

| Type | How it works |
|---|---|
| `httpGet` | Makes an HTTP GET request; success = 200-399 response |
| `tcpSocket` | Opens a TCP connection; success = connection established |
| `exec` | Runs a command inside the container; success = exit code 0 |

---

## What I did

### Part 1 — Liveness Probe

```yaml
# lab09-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab09-liveness
spec:
  containers:
    - name: app
      image: nginx:alpine
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
        failureThreshold: 3
```

`kubectl describe pod lab09-liveness` confirmed the probe was registered:
```
Liveness: http-get http://:80/ delay=5s timeout=1s period=10s #success=1 #failure=3
```

**Forcing a liveness failure:**
```
kubectl exec lab09-liveness -- nginx -s stop
kubectl get pods --watch
```

Result: `RESTARTS: 1 (7s ago)` — K8s detected the container failure and restarted it within 7 seconds. The `--watch` flag only prints new lines when state changes, so a healthy Pod produces no output.

### Part 2 — Readiness Probe (intentionally failing)

```yaml
# lab09-readiness.yaml
readinessProbe:
  httpGet:
    path: /ready     ← nginx doesn't serve this path by default
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

```
kubectl apply -f lab09-readiness.yaml
kubectl get pods
```

Result:
```
lab09-liveness    1/1   Running   ← Ready, receives traffic
lab09-readiness   0/1   Running   ← Running but NOT ready, excluded from Service
```

Both Pods are `Running` — neither is restarted. But `lab09-readiness` shows `0/1` in the READY column, meaning if a Service pointed at it, no traffic would be sent. The Pod stays alive, K8s just waits for it to become ready.

### Part 3 — Both probes together

```yaml
# lab09-both-probes.yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 3
  periodSeconds: 5
  failureThreshold: 3
```

`kubectl describe pod -l app=lab09-app` confirmed both probes on each Pod:
```
Liveness:  http-get http://:80/ delay=5s timeout=1s period=10s #success=1 #failure=3
Readiness: http-get http://:80/ delay=3s timeout=1s period=5s  #success=1 #failure=3
```

Readiness checks more frequently (`period=5s` vs `period=10s`) and starts sooner (`delay=3s` vs `delay=5s`) — it's more time-sensitive because it controls live traffic routing.

---

## Connection to Docker Lab 12

In Docker Lab 12, a health check failure was traced to a bug: the `/health` endpoint didn't exist, so `wget` requests hung indefinitely — making the container appear unhealthy when the application was actually running. The fix was:

1. A dedicated `/health` endpoint that never touches the database
2. A `dbReady` flag so the main route fails fast instead of hanging

The same pattern applies directly to K8s probes:

```
Liveness → /health endpoint (lightweight, no DB dependency)
            "Is the server process itself alive?"

Readiness → /ready endpoint (checks DB connection, cache warm-up, etc.)
             "Is the application ready to handle real requests?"
```

Separating these two concerns is a production best practice — a Pod that's alive but not ready should stay in the cluster (so K8s can wait for it to recover) but should not receive user traffic.

---

## Probe configuration fields

| Field | Default | Description |
|---|---|---|
| `initialDelaySeconds` | 0 | Wait this long after container start before first probe |
| `periodSeconds` | 10 | How often to run the probe |
| `timeoutSeconds` | 1 | Probe times out after this many seconds |
| `successThreshold` | 1 | Consecutive successes needed to mark healthy |
| `failureThreshold` | 3 | Consecutive failures before taking action |

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Create a Pod/Deployment with probes |
| `kubectl describe pod <name>` | Show probe configuration and health events |
| `kubectl get pods --watch` | Watch Pod state changes in real time |
| `kubectl exec <pod> -- <cmd>` | Run a command inside a Pod (used to trigger failures) |

## Notes

In the EKS final project, probes will be added to the backend Deployment — a liveness probe on `/health` and a readiness probe that also verifies the database connection is established before the Pod starts receiving traffic from the Service. This mirrors the `dbReady` flag pattern from Docker Lab 12.
