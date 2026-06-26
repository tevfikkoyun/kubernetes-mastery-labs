# Lab 12 — Horizontal Pod Autoscaler (HPA)

## Goal

Understand how Kubernetes automatically scales Pod count based on CPU utilization using the Horizontal Pod Autoscaler, observe scale-up and scale-down behavior in real time, and configure cooldown periods.

---

## The Problem

Manual scaling (`kubectl scale --replicas=N`) requires human intervention and can't react to sudden traffic spikes in real time. In production, traffic is unpredictable — a viral post, a scheduled batch job, or a flash sale can multiply load in seconds.

**HPA** solves this by continuously monitoring resource metrics and automatically adjusting replica count within defined bounds:

```
Low traffic  → CPU 5%   → 1 Pod  (minReplicas)
High traffic → CPU 88%  → HPA triggers → 2, 3... Pods
Load drops   → CPU 0%   → HPA waits (cooldown) → back to 1 Pod
```

---

## How HPA works

```
                    ┌─────────────────┐
                    │  metrics-server  │
                    │  (collects CPU,  │
                    │   memory, etc.)  │
                    └────────┬────────┘
                             │ every 15s
                             ▼
┌──────────────────────────────────────────┐
│              HPA controller               │
│                                          │
│  current CPU > target? → scale UP        │
│  current CPU < target? → wait cooldown   │
│                          → scale DOWN    │
└──────────────┬───────────────────────────┘
               │ adjusts
               ▼
         Deployment replicas
```

**Prerequisites:**
- `metrics-server` must be installed (provides CPU/memory data)
- Pods must have `resources.requests.cpu` defined (HPA calculates utilization as `actual / request`)

---

## Scale-up vs Scale-down asymmetry

| | Scale-up | Scale-down |
|---|---|---|
| Speed | Fast (immediate) | Slow (cooldown period) |
| Default wait | 0s | 5 minutes |
| Reason | Users shouldn't wait | Prevent thrashing (rapid open/close cycles) |

**Thrashing** = repeatedly scaling up and down due to short traffic spikes. If traffic spikes for 20 seconds every minute, HPA without a cooldown would open and close Pods constantly — wasteful and slow.

---

## What I did

### Part 1 — Installing metrics-server

`kubectl top pod` failed in Lab 10 (Docker Desktop K8s doesn't include metrics-server by default). Installed and patched for local TLS:

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

Verified:
```
kubectl top nodes
# NAME                    CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
# desktop-control-plane   60m          0%       725Mi           4%
```

### Part 2 — Deployment with resource requests

```yaml
# lab12-deployment.yaml
containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
```

`requests.cpu` is required — HPA calculates utilization as `(actual CPU) / (requested CPU)`. Without it, HPA can't compute a percentage.

### Part 3 — HPA with custom cooldown

```yaml
# lab12-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: lab12-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: lab12-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
```

`stabilizationWindowSeconds: 15` on scale-down (reduced from the 5-minute default) for faster observation. In production, the default 5 minutes is usually appropriate.

### Part 4 — Load generation and HPA in action

```
kubectl run load-generator --image=busybox --restart=Never -- \
  sh -c "while true; do wget -q -O- http://lab12-service; done"
```

Observed via `kubectl get hpa --watch`:

```
cpu: 0%/50%    REPLICAS: 1  ← initial state, no load
cpu: 88%/50%   REPLICAS: 1  ← load generator started, CPU spiked
cpu: 60%/50%   REPLICAS: 2  ← HPA scaled up to 2 Pods
cpu: 38%/50%   REPLICAS: 2  ← load distributed, CPU stabilized
cpu: 29%/50%   REPLICAS: 2  ← 2 Pods sufficient, no further scale-up
```

2 Pods was enough to keep average CPU below 50% — HPA found the equilibrium and stopped.

### Part 5 — Scale-down after load removal

```
kubectl delete pod load-generator
kubectl get hpa --watch
```

```
cpu: 33%/50%   REPLICAS: 2  ← load generator deleted
cpu: 12%/50%   REPLICAS: 2  ← CPU dropping, 15s stabilization window starts
cpu: 0%/50%    REPLICAS: 1  ← 15s elapsed, scaled back down to 1
```

With the default 5-minute cooldown this would have taken ~300 seconds. With `stabilizationWindowSeconds: 15` it took ~15 seconds — demonstrating the cooldown's direct effect.

### Cleanup

```
kubectl delete deployment lab12-app
kubectl delete service lab12-service
kubectl delete hpa lab12-hpa
```

---

## HPA configuration reference

| Field | Description |
|---|---|
| `minReplicas` | Minimum Pods — never scales below this |
| `maxReplicas` | Maximum Pods — never scales above this |
| `averageUtilization` | Target CPU % across all Pods (e.g. 50 = keep average at 50%) |
| `stabilizationWindowSeconds` (scaleDown) | How long to wait after metric drops before scaling down |
| `stabilizationWindowSeconds` (scaleUp) | How long to wait after metric rises before scaling up (default 0) |

---

## HPA scaling formula

```
desiredReplicas = ceil(currentReplicas × (currentMetric / desiredMetric))
```

Example: 2 Pods, current CPU 88%, target 50%:
```
ceil(2 × (88 / 50)) = ceil(3.52) = 4 Pods
```

HPA would want 4 Pods — but since CPU stabilized at ~35% with 2 Pods before the next evaluation, it stayed at 2.

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Create or update an HPA |
| `kubectl get hpa` | List HPAs with current metrics and replica count |
| `kubectl get hpa --watch` | Watch HPA scaling decisions in real time |
| `kubectl describe hpa <name>` | Full HPA details including scaling events |
| `kubectl top pod` | Show live CPU/memory per Pod (requires metrics-server) |
| `kubectl top nodes` | Show live CPU/memory per Node |

## Notes

In production on EKS, HPA works identically but with a properly configured metrics-server (or KEDA for event-driven scaling). Combined with Cluster Autoscaler (which adds/removes EC2 nodes when all nodes are full), HPA + Cluster Autoscaler creates a fully elastic system: Pods scale within existing nodes, and nodes scale when Pods can't be scheduled. This will be relevant in the EKS final project.
