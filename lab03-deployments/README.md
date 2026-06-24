# Lab 03 — Deployments

## Goal

Understand why Deployments are the preferred way to run Pods in Kubernetes, and demonstrate the four core capabilities they add over naked Pods: self-healing, scaling, rolling updates, and rollbacks.

---

## Why Deployments, not naked Pods?

In Lab 01 and Lab 02, we ran "naked Pods" — Pods with no controller managing them. When a naked Pod is deleted, it's gone. In production, this is never acceptable.

A **Deployment** wraps Pods in a controller that:
1. Always maintains the desired number of replicas
2. Replaces failed or deleted Pods automatically
3. Updates Pods with zero downtime (rolling update)
4. Reverts to a previous version instantly (rollback)

```
Naked Pod:   You → Pod         (no manager, no recovery)
Deployment:  You → Deployment → ReplicaSet → Pod(s)
                      ↑                        ↓
                      └────────── watches ──────┘
                         (replaces missing Pods)
```

The **ReplicaSet** is an intermediate layer — Deployment creates and manages ReplicaSets, which in turn manage Pods. You rarely interact with ReplicaSets directly, but their ID is visible in Pod names:

```
lab03-nginx  -  54fc99c8d  -  7p8nk
     │               │           │
 Deployment      ReplicaSet    Pod
   name            hash        hash
```

---

## What I did

### Part 1 — Creating a Deployment

```yaml
# lab03-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab03-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

Key fields:
- `replicas: 3` — always keep 3 Pods running
- `selector.matchLabels` — how the Deployment identifies "its" Pods (must match `template.metadata.labels`)
- `template` — the Pod blueprint, identical in structure to a standalone Pod manifest

```
kubectl apply -f lab03-deployment.yaml
kubectl get pods
kubectl get deployment
```

Result: 3 Pods running, Deployment showing `3/3 READY`.

### Part 2 — Self-healing

Deleted one Pod manually:
```
kubectl delete pod lab03-nginx-54fc99c8d-7p8nk
kubectl get pods
```

Result: replacement Pod appeared within 1 second (`AGE: 1s`). The Deployment controller detected the desired replica count (3) no longer matched reality (2) and immediately created a new Pod.

Contrast with Lab 01: deleting a naked Pod returned "No resources found."

### Part 3 — Scaling

**Imperative** (instant, but doesn't update the YAML file):
```
kubectl scale deployment lab03-nginx --replicas=5
kubectl get pods
# → 5 pods running
```

**Declarative** (correct for production — update the file, then apply):
Changed `replicas: 2` in `lab03-deployment.yaml`, then:
```
kubectl apply -f lab03-deployment.yaml
kubectl get pods
# → 3 pods Terminating, 2 remaining Running
```

Important: if you scale imperatively and then `kubectl apply` from the original YAML, `apply` wins — the file's `replicas` value overrides the live state. This is why the YAML file should always reflect the true desired state.

### Part 4 — Rolling update

Updated the container image from `nginx:alpine` to `nginx:latest`:
```
kubectl set image deployment/lab03-nginx nginx=nginx:latest
kubectl rollout status deployment/lab03-nginx
kubectl get pods
```

Result: Pod names changed from `54fc99c8d-xxxxx` to `7d584c4448-xxxxx` — a new ReplicaSet was created for the new image. At no point were 0 Pods running — K8s brings up new Pods before taking down old ones.

Rolling update sequence (with 2 replicas):
```
Start:  [Pod1-alpine] [Pod2-alpine]
Step 1: [Pod1-alpine] [Pod2-alpine] [Pod3-latest]  ← new pod added
Step 2: [Pod2-alpine] [Pod3-latest]                ← old pod removed
Step 3: [Pod3-latest] [Pod4-latest]                ← second new pod added
Step 4: [Pod3-latest] [Pod4-latest]                ← second old pod removed
```
Zero downtime throughout.

### Part 5 — Rollback

```
kubectl rollout undo deployment/lab03-nginx
kubectl rollout status deployment/lab03-nginx
kubectl get pods
```

Pod names reverted to `54fc99c8d-xxxxx` — the old ReplicaSet (nginx:alpine) was reactivated. K8s never deletes old ReplicaSets immediately after a rolling update, precisely to enable instant rollback.

View rollout history:
```
kubectl rollout history deployment/lab03-nginx
```

Roll back to a specific revision:
```
kubectl rollout undo deployment/lab03-nginx --to-revision=1
```

### Cleanup

```
kubectl delete deployment lab03-nginx
kubectl get pods
# → No resources found in default namespace
```
Deleting the Deployment automatically deletes all its Pods and ReplicaSets. Never delete individual Pods to "stop" a Deployment — the controller will just recreate them.

---

## Key concepts

| Concept | Description |
|---|---|
| **Desired state** | The replica count you declare; K8s continuously works to maintain it |
| **Self-healing** | Automatically replaces failed or deleted Pods |
| **ReplicaSet** | The intermediate controller that manages a specific set of Pod replicas; Deployment manages ReplicaSets |
| **Rolling update** | Replaces Pods incrementally — new ones come up before old ones go down, ensuring zero downtime |
| **Rollback** | Reverts to the previous ReplicaSet; instant because old ReplicaSets are kept |
| **`kubectl scale`** | Imperative scaling — useful for testing but doesn't update the manifest |
| **`kubectl set image`** | Imperative image update — triggers a rolling update |
| **`kubectl rollout`** | Manage and inspect the rollout state of a Deployment |

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Create or update a Deployment |
| `kubectl get deployment` | List Deployments with replica status |
| `kubectl scale deployment <name> --replicas=<n>` | Imperatively scale a Deployment |
| `kubectl set image deployment/<name> <container>=<image>` | Trigger a rolling update with a new image |
| `kubectl rollout status deployment/<name>` | Watch rollout progress in real time |
| `kubectl rollout history deployment/<name>` | Show revision history |
| `kubectl rollout undo deployment/<name>` | Roll back to the previous revision |
| `kubectl rollout undo deployment/<name> --to-revision=<n>` | Roll back to a specific revision |
| `kubectl delete deployment <name>` | Delete a Deployment and all its Pods |

## Notes

Always delete the **Deployment**, not individual Pods, when you want to stop a workload. Deleting a Pod that belongs to a Deployment just causes the controller to replace it — the Deployment is the actual unit of ownership.
