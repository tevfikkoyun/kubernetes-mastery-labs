# Lab 02 — Pods

## Goal

Move from imperative (`kubectl run`) to declarative (YAML manifest) Pod creation, understand the Pod lifecycle, and observe what happens when a Pod fails — including the `CrashLoopBackOff` state and how to debug it.

---

## Imperative vs Declarative

In Lab 01, we used `kubectl run` to create a Pod — this is the **imperative** approach: "do this right now." In real Kubernetes usage, everything is **declarative**: you write a YAML file describing the desired state, and K8s makes it happen.

| Approach | Command | When to use |
|---|---|---|
| Imperative | `kubectl run nginx --image=nginx` | Quick tests, learning |
| Declarative | `kubectl apply -f pod.yaml` | Everything in production |

The declarative approach is more powerful because:
- The YAML file is your source of truth — it can be committed to git
- `kubectl apply` is idempotent — running it twice on the same file updates the resource, not duplicates it
- K8s knows the difference between "create" and "update" based on the resource name in the manifest

---

## YAML Manifest structure

Every K8s manifest has the same four top-level fields:

```yaml
apiVersion: v1        # Which K8s API version handles this resource type
kind: Pod             # What type of resource
metadata:             # Name, labels, annotations
  name: lab02-nginx
  labels:
    app: nginx
    lab: lab02
spec:                 # The actual desired state — what should run
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
```

`apiVersion` varies by resource type:
- `v1` → Pod, Service, ConfigMap, Secret, PersistentVolume
- `apps/v1` → Deployment, ReplicaSet, DaemonSet (Lab 03)

`labels` are key-value pairs attached to resources — they're how Services find their Pods (Lab 04) and how you filter resources in `kubectl get`.

---

## What I did

### Part 1 — Declarative Pod creation

```yaml
# lab02-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab02-nginx
  labels:
    app: nginx
    lab: lab02
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      ports:
        - containerPort: 80
```

```
kubectl apply -f lab02-pod.yaml
kubectl get pods
```
Result: `lab02-nginx` reached `1/1 Running`.

### Part 2 — `kubectl apply` is idempotent

Changed `image: nginx:alpine` to `image: nginx` in the YAML and ran `kubectl apply -f lab02-pod.yaml` again. K8s updated the existing `lab02-nginx` Pod rather than creating a new one — it matched on the `name` field in `metadata`.

This is the core of the declarative model: `apply` means "make reality match this file," not "run this command again."

### Part 3 — Pod lifecycle and CrashLoopBackOff

```yaml
# lab02-failing-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab02-failing
spec:
  containers:
    - name: failing-container
      image: nginx:alpine
      command: ["sh", "-c", "echo 'starting...' && exit 1"]
```

```
kubectl apply -f lab02-failing-pod.yaml
kubectl get pods --watch
```

Observed in real time:
```
lab02-failing   0/1   Error              2 (22s ago)   23s
lab02-failing   0/1   CrashLoopBackOff   2 (14s ago)   32s
```

### Part 4 — Debugging the failing Pod

```
kubectl describe pod lab02-failing
kubectl logs lab02-failing
```

`describe` Events showed:
- Pod restarted **5 times** in 2 minutes (`x5 over 2m6s`)
- K8s applied **exponential backoff** — 9 back-off events (`x9 over 2m5s`), with wait times doubling each retry

`logs` showed: `starting...` — the container ran, printed its message, then exited with code 1. In production, this log output is the first place to look when diagnosing a `CrashLoopBackOff`.

### Cleanup

```
kubectl delete pod lab02-nginx lab02-failing
kubectl get pods
# → No resources found in default namespace
```

---

## Pod lifecycle states

| State | Meaning |
|---|---|
| `Pending` | Pod accepted by K8s but not yet scheduled or image not yet pulled |
| `ContainerCreating` | Scheduled to a node, image being pulled |
| `Running` | At least one container is running |
| `Error` | Container exited with a non-zero exit code |
| `CrashLoopBackOff` | Container keeps crashing; K8s is backing off restarts with increasing delays |
| `Completed` | Container ran to completion successfully (exit code 0) |
| `Terminating` | Pod is being deleted |

---

## CrashLoopBackOff explained

When a container exits with an error, K8s restarts it — but not immediately every time. It uses **exponential backoff**: each restart attempt waits twice as long as the previous one (roughly 10s → 20s → 40s → 80s... up to a maximum of ~5 minutes). This prevents a broken container from hammering the node with constant restarts.

`CrashLoopBackOff` is not a permanent failure — it's K8s saying "this keeps failing, I'll keep trying but with increasing patience."

**Debug checklist when you see CrashLoopBackOff:**
1. `kubectl logs <pod>` — what did the container print before dying?
2. `kubectl describe pod <pod>` — check `Events:` for restart count and back-off messages
3. Check `Exit Code` in `describe` output — `1` means the app errored, `137` means OOM kill, `143` means SIGTERM

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Create or update a resource from a YAML manifest |
| `kubectl get pods` | List Pods in the current namespace |
| `kubectl get pods --watch` | Watch Pod status changes in real time |
| `kubectl describe pod <name>` | Full Pod details including lifecycle events |
| `kubectl logs <name>` | View container stdout/stderr output |
| `kubectl delete pod <name>` | Delete a Pod permanently |
| `kubectl delete -f <file>` | Delete the resource defined in a YAML file |

## Notes

Pod names are **immutable** — once created, you cannot rename a Pod. To "rename" one, you delete it and recreate it with the corrected manifest. This is one of many reasons why Deployments (Lab 03) are preferred over naked Pods in practice: a Deployment lets you update the Pod spec and rolls it out automatically.
