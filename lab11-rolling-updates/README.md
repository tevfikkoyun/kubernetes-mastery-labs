# Lab 11 — Rolling Updates and Rollbacks

## Goal

Understand Kubernetes update strategies in depth — `RollingUpdate` vs `Recreate` — observe how `maxSurge` and `maxUnavailable` control update behavior, and practice rollback to a specific revision.

---

## Update Strategies

### RollingUpdate (default)

Replaces Pods incrementally — new ones come up before (or alongside) old ones going down. Guarantees a minimum number of Pods are always available during the update.

```
[v1][v1][v1][v1]  → initial state
[v1][v1][v1][v2]  → 1 new added (maxSurge)
[v1][v1][v2][v2]  → 1 old removed (maxUnavailable)
[v1][v2][v2][v2]  → continues...
[v2][v2][v2][v2]  → complete
Downtime: NONE
```

### Recreate

Kills all existing Pods first, then creates new ones. There's a gap where zero Pods are running.

```
[v1][v1][v1]  → all Terminating simultaneously
[  ][  ][  ]  → gap (downtime)
[v2][v2][v2]  → all ContainerCreating simultaneously
Downtime: YES (brief, but real)
```

**When to use Recreate:** Database migrations or schema changes where running old and new versions simultaneously could corrupt data. The gap ensures only one version ever runs at a time.

---

## maxSurge and maxUnavailable

These two fields control the pace and safety of a `RollingUpdate`:

| Field | Default | Meaning |
|---|---|---|
| `maxSurge` | 25% | Max Pods above `replicas` during update |
| `maxUnavailable` | 25% | Max Pods below `replicas` during update |

With `replicas: 4`, `maxSurge: 1`, `maxUnavailable: 1`:
- At most 5 Pods run at any time (4 + 1)
- At least 3 Pods serve traffic at any time (4 - 1)

**Common production configurations:**

```yaml
# Zero downtime, slower update:
maxSurge: 1
maxUnavailable: 0
# → new Pod must be Running before any old Pod is removed

# Faster update, brief reduction in capacity:
maxSurge: 1
maxUnavailable: 1
# → new Pod starts while old Pod is simultaneously removed
```

---

## What I did

### Part 1 — Deployment with explicit update strategy

```yaml
# lab11-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab11-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: lab11-app
  template:
    metadata:
      labels:
        app: lab11-app
    spec:
      containers:
        - name: app
          image: nginx:1.25-alpine
          resources:
            requests:
              memory: "32Mi"
              cpu: "100m"
            limits:
              memory: "64Mi"
              cpu: "200m"
```

### Part 2 — Rolling update with annotation

```
kubectl set image deployment/lab11-app app=nginx:1.27-alpine
kubectl annotate deployment/lab11-app kubernetes.io/change-cause="Update nginx from 1.25 to 1.27"
```

Observed via `kubectl get pods --watch`:
- Old Pods (`5b5b995898-xxx`) terminated one by one
- New Pods (`5fb67d46d-xxx`) started in parallel
- At no point were fewer than 3 Pods running

### Part 3 — Rollout history

```
kubectl rollout history deployment/lab11-app
```

```
REVISION  CHANGE-CAUSE
1         <none>
2         Update nginx from 1.25 to 1.27
```

The `CHANGE-CAUSE` field is populated by the `kubectl annotate` command — without it, all revisions show `<none>`. In production, annotating every rollout creates a clear audit trail.

### Part 4 — Rollback to specific revision

```
kubectl rollout undo deployment/lab11-app --to-revision=1
kubectl rollout history deployment/lab11-app
```

Result:
```
REVISION  CHANGE-CAUSE
2         Update nginx from 1.25 to 1.27
3         <none>
```

**Key insight:** A rollback creates a new revision — it doesn't restore the old one. `REVISION 1` became `REVISION 3`. This means every state change (forward or backward) is tracked and reversible. `REVISION 2` is still in history and can be rolled back to at any time.

### Part 5 — Recreate strategy comparison

```yaml
# lab11-recreate.yaml
strategy:
  type: Recreate
```

Observed via `kubectl get pods --watch` after `kubectl set image`:
```
# All old Pods terminated simultaneously:
lab11-recreate-5889f5fb94-45kp6   Terminating
lab11-recreate-5889f5fb94-pzgqv   Terminating
lab11-recreate-5889f5fb94-qfp62   Terminating

# Gap — zero Pods running (downtime)

# All new Pods created simultaneously:
lab11-recreate-88cc5cc56-5zq5k    ContainerCreating
lab11-recreate-88cc5cc56-wtgbk    ContainerCreating
lab11-recreate-88cc5cc56-b28bx    ContainerCreating
```

Contrast with RollingUpdate where old and new Pods overlapped — no gap, no downtime.

---

## Strategy comparison

| | RollingUpdate | Recreate |
|---|---|---|
| Downtime | None | Brief (gap between old and new) |
| Old + new running simultaneously | Yes | No |
| Update speed | Gradual | Instant (all at once) |
| Use case | Stateless services, APIs | Database migrations, single-version requirements |
| `maxSurge` / `maxUnavailable` | Configurable | Not applicable |

---

## Rollout revision behavior

```
Initial deploy  → REVISION 1
Update image    → REVISION 2 (REVISION 1 kept in history)
Rollback to 1   → REVISION 3 (same config as REVISION 1, new number)
```

K8s never deletes rollout history immediately — old ReplicaSets are retained (scaled to 0) so rollbacks are instant. The number of retained revisions is controlled by `spec.revisionHistoryLimit` (default: 10).

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl set image deployment/<name> <container>=<image>` | Trigger a rolling update |
| `kubectl annotate deployment/<name> kubernetes.io/change-cause="..."` | Add a description to the current revision |
| `kubectl rollout status deployment/<name>` | Watch rollout progress |
| `kubectl rollout history deployment/<name>` | Show revision history with change causes |
| `kubectl rollout undo deployment/<name>` | Roll back to the previous revision |
| `kubectl rollout undo deployment/<name> --to-revision=<n>` | Roll back to a specific revision |
| `kubectl get pods --watch` | Watch Pod state changes in real time |

## Notes

In production CI/CD pipelines, the `annotate` step is typically automated — the pipeline stamps each deployment with the git commit SHA, the deployer's identity, and a timestamp. Combined with `rollout history`, this creates a complete audit trail of every change to the cluster.
