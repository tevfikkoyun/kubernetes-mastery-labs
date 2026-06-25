# Lab 07 — Persistent Volumes

## Goal

Understand how Kubernetes handles persistent storage through PersistentVolumes (PV), PersistentVolumeClaims (PVC), and StorageClasses — and prove that data survives Pod deletion.

---

## The Problem

Containers (and Pods) are ephemeral by nature. When a Pod is deleted, everything written to its filesystem is gone. For stateful applications (databases, file storage, caches), this is unacceptable.

In Docker (Lab 04), we solved this with named volumes. In Kubernetes, the solution is more structured and decoupled:

```
Docker:   docker volume create mydata → mount into container (manual, tightly coupled)
K8s:      PVC (claim) → StorageClass (provisioner) → PV (actual storage)
```

---

## The three components

### PersistentVolume (PV)
The actual storage resource in the cluster — a piece of disk. Can be local (like Minikube's hostpath), network-attached (NFS), or cloud (AWS EBS, GCP PD). Created either manually by an admin or automatically by a StorageClass.

### PersistentVolumeClaim (PVC)
A request for storage made by a Pod. The PVC says "I need 1Gi of ReadWriteOnce storage" — K8s finds (or creates) a matching PV and binds them together.

### StorageClass
A template that defines how PVs are dynamically provisioned. When a PVC is created and no matching PV exists, the StorageClass automatically creates one.

```
Pod
 │
 ▼
PVC (claim: "I need 1Gi RWO")
 │
 ▼
StorageClass (provisioner creates PV automatically)
 │
 ▼
PV (actual storage: /data on the node, EBS volume, etc.)
```

---

## Access Modes

| Mode | Abbrev | Description |
|---|---|---|
| ReadWriteOnce | RWO | Read/write by a single node (most common) |
| ReadOnlyMany | ROX | Read-only by many nodes simultaneously |
| ReadWriteMany | RWX | Read/write by many nodes simultaneously (requires NFS or similar) |

---

## What I did

### Part 1 — Inspecting StorageClasses

```
kubectl get storageclass
```

Minikube provides two StorageClasses (`standard` as default, `hostpath`), both using `rancher.io/local-path` as the provisioner. The `(default)` label means PVCs without an explicit StorageClass will use `standard` automatically.

Key columns:
- `RECLAIMPOLICY: Delete` — when the PVC is deleted, the PV and its data are also deleted
- `VOLUMEBINDINGMODE: WaitForFirstConsumer` — the PV is not created until a Pod actually requests it

### Part 2 — Creating a PVC

```yaml
# lab07-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab07-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```
kubectl apply -f lab07-pvc.yaml
kubectl get pvc
kubectl get pv
```

Result:
```
PVC: lab07-pvc   STATUS: Bound   VOLUME: pvc-04bf7d34-...   CAPACITY: 1Gi
PV:  pvc-04bf7d34-...   CAPACITY: 1Gi   STATUS: Bound   CLAIM: default/lab07-pvc
```

The StorageClass automatically created a PV and bound it to the PVC. Both share the same UUID — that's the binding.

### Part 3 — Pod using the PVC

```yaml
# lab07-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab07-pod
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "echo 'data written by pod' > /data/test.txt && cat /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: storage
          mountPath: /data
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: lab07-pvc
```

```
kubectl apply -f lab07-pod.yaml
kubectl logs lab07-pod
# → data written by pod
```

### Part 4 — Data survives Pod deletion

```
kubectl delete pod lab07-pod
kubectl apply -f lab07-pod.yaml
kubectl logs lab07-pod
# → data written by pod ✅
```

The new Pod re-attached to the same PVC and read the file written by the previous Pod. The PVC (and PV) outlived the Pod — exactly like Docker's named volumes outliving containers.

### Part 5 — ReclaimPolicy in action

```
kubectl delete pod lab07-pod
kubectl delete pvc lab07-pvc
kubectl get pv
# → No resources found
```

With `RECLAIMPOLICY: Delete`, deleting the PVC automatically deleted the PV and all its data.

---

## ReclaimPolicy comparison

| Policy | Behavior when PVC is deleted | Use case |
|---|---|---|
| `Delete` | PV and data are deleted automatically | Temporary data, test environments |
| `Retain` | PV and data remain, must be manually cleaned up | Critical data (databases in production) |
| `Recycle` | Data is wiped, PV is made available again | Deprecated, not recommended |

In AWS EKS with EBS volumes, `Retain` is typically preferred for database workloads — accidental PVC deletion should not mean accidental data loss.

---

## Docker Lab 04 → Kubernetes comparison

| Docker (Lab 04) | Kubernetes (Lab 07) |
|---|---|
| `docker volume create mydata` | PVC + StorageClass (auto-provisioned) |
| `-v mydata:/data` in `docker run` | `volumes.persistentVolumeClaim.claimName` |
| Volume outlives container | PVC/PV outlive Pod |
| `docker volume rm mydata` | `kubectl delete pvc` (+ PV deleted if `Delete` policy) |
| No concept of access modes | `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany` |

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl get storageclass` | List available StorageClasses |
| `kubectl apply -f <file>` | Create a PVC from YAML |
| `kubectl get pvc` | List PersistentVolumeClaims and their status |
| `kubectl get pv` | List PersistentVolumes |
| `kubectl describe pvc <name>` | Full PVC details including bound PV |
| `kubectl delete pvc <name>` | Delete a PVC (and PV if ReclaimPolicy is Delete) |

## Notes

In production on AWS EKS, the `ebs.csi.aws.com` provisioner creates EBS volumes dynamically when a PVC is submitted. The StorageClass configuration looks similar to what we used here, but the underlying storage is a real AWS EBS volume — durable, replicated, and independent of any EC2 instance. We'll see this in the EKS final project.
