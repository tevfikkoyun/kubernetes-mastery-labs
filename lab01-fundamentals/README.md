# Lab 01 — Kubernetes Fundamentals

## Goal

Understand Kubernetes' core architecture, inspect the internal components running in a real cluster, run a first Pod, and observe how K8s differs from plain Docker — both in concept and in what happens when a Pod is deleted.

---

## Kubernetes Architecture — Deep Dive

Before running any commands, it's worth understanding what Kubernetes actually is and how its pieces fit together. This section covers the concepts that everything else builds on.

### What is a Cluster?

The outermost boundary — everything lives inside it. A cluster contains both the "manager" and the "workers." You talk to the cluster via `kubectl`, and K8s handles the rest.

```
┌─────────────────────────────────────────────────────┐
│                    CLUSTER                          │
│                                                     │
│  ┌──────────────────┐    ┌──────────────────────┐  │
│  │  CONTROL PLANE   │    │    WORKER NODE(S)    │  │
│  │   (the brain)    │    │    (the hands)       │  │
│  └──────────────────┘    └──────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

In Minikube, both live on the same machine (single node). In a real EKS cluster, the Control Plane is managed by AWS on separate infrastructure, and the Worker Nodes are your EC2 instances.

---

### Control Plane — "The Brain"

The cluster's decision-making layer. Your application does **not** run here — this is purely for coordination, scheduling, and state tracking. Every `kubectl` command you run reaches the Control Plane first.

```
You: kubectl apply -f deployment.yaml
         │
         ▼
┌─────────────────────────────────────────┐
│           CONTROL PLANE                 │
│                                         │
│  ┌─────────────────┐                    │
│  │  kube-apiserver │ ← all commands     │
│  │                 │   come here first  │
│  └────────┬────────┘                    │
│           │ "new deployment received"   │
│           ▼                             │
│  ┌─────────────────┐  ┌──────────────┐ │
│  │ kube-scheduler  │  │     etcd     │ │
│  │                 │  │              │ │
│  │ "which node     │  │  cluster's   │ │
│  │  should run     │  │  database    │ │
│  │  this Pod?"     │  │  (all state) │ │
│  └────────┬────────┘  └──────────────┘ │
│           │                             │
│  ┌────────▼────────────────────┐        │
│  │ kube-controller-manager     │        │
│  │                             │        │
│  │ "3 pods desired, only 2     │        │
│  │  running → create 1 more"   │        │
│  └─────────────────────────────┘        │
└─────────────────────────────────────────┘
```

| Component | Purpose |
|---|---|
| `kube-apiserver` | Central gateway — every `kubectl` command goes through here. The only component that reads/writes to etcd directly. |
| `etcd` | K8s database — stores the entire cluster state (what's running, where, what's desired). Think of it as the source of truth. |
| `kube-scheduler` | Watches for newly created Pods with no assigned node, then picks the best node based on resource availability. |
| `kube-controller-manager` | Runs a set of controllers that continuously compare desired state vs actual state — if a Pod dies and wasn't supposed to, it creates a new one. |

The most important concept here: **desired state**. You don't tell K8s "run this container." You tell it "I want 3 replicas of this Pod always running," and the Control Plane continuously works to make reality match that declaration — forever, without you having to intervene.

---

### Worker Node — "The Hands"

Where your application actually runs. Each Worker Node has three core components:

```
┌──────────────────────────────────────────┐
│              WORKER NODE                 │
│                                          │
│  ┌─────────────────────────────────┐     │
│  │           kubelet               │     │
│  │                                 │     │
│  │  Receives instructions from     │     │
│  │  Control Plane, starts/stops    │     │
│  │  Pods on this node, reports     │     │
│  │  health back up                 │     │
│  └─────────────────────────────────┘     │
│                                          │
│  ┌─────────────────────────────────┐     │
│  │          kube-proxy             │     │
│  │                                 │     │
│  │  Manages network rules,         │     │
│  │  routes Service traffic to      │     │
│  │  the correct Pod                │     │
│  └─────────────────────────────────┘     │
│                                          │
│  ┌──────────┐  ┌──────────┐             │
│  │  Pod 1   │  │  Pod 2   │  ...        │
│  │ (nginx)  │  │  (app)   │             │
│  └──────────┘  └──────────┘             │
└──────────────────────────────────────────┘
```

`kubelet` is the Control Plane's field representative — the Control Plane says "run this Pod on this node," and kubelet makes it happen, then keeps checking "is the Pod still healthy?"

---

### What is a Pod?

The smallest deployable unit in Kubernetes. If Docker's atomic unit is a container, K8s's atomic unit is a Pod. A Pod wraps one or more containers that share network and storage:

```
┌─────────────────────────────┐
│            POD              │
│                             │
│  ┌──────────┐ ┌──────────┐ │
│  │container │ │container │ │  ← usually 1 container,
│  │  (nginx) │ │ (helper) │ │    sometimes 2-3 together
│  └──────────┘ └──────────┘ │
│                             │
│  Shared IP: 10.244.0.3     │
│  Shared volumes             │
│  Shared network namespace   │
└─────────────────────────────┘
```

All containers in a Pod share the same IP address (they reach each other via `localhost`), the same storage mounts, and the same network namespace. They are born together and die together. In practice, most Pods contain a single container. A second container is typically a "sidecar" (log collector, proxy, etc.) — covered in later labs.

---

### Full picture — From `kubectl apply` to a running Pod

```
You                 Control Plane              Worker Node
 │                       │                         │
 │   kubectl apply       │                         │
 │──────────────────────▶│  kube-apiserver         │
 │                       │  "deployment saved"     │
 │                       │         │               │
 │                       │  write to etcd          │
 │                       │         │               │
 │                       │         ▼               │
 │                       │  kube-scheduler         │
 │                       │  "assign to: minikube"  │
 │                       │         │               │
 │                       │         ▼               │
 │                       │  controller-manager     │
 │                       │  "create the Pod"       │
 │                       │                │        │
 │                       │                ▼        │
 │                       │           kubelet       │
 │                       │           "pull image,  │
 │                       │            start        │
 │                       │            container"   │
 │                       │                │        │
 │   kubectl get pods    │                ▼        │
 │──────────────────────▶│           Pod: Running  │
```

---

### Summary — One line each

- **Cluster** = the entire system
- **Control Plane** = the brain (decides, coordinates, stores state)
- **Worker Node** = the hands (actually runs your applications)
- **Pod** = the smallest unit where your application lives
- **kubelet** = the bridge between Control Plane and Worker Node

---

## Docker → Kubernetes mental map

| Docker / Compose | Kubernetes equivalent |
|---|---|
| `docker run` | `kubectl run` |
| `docker ps` | `kubectl get pods` |
| `docker inspect` | `kubectl describe pod` |
| `docker logs` | `kubectl logs` |
| `docker rm` | `kubectl delete pod` |
| `docker-compose.yml` | YAML manifest files |
| `services:` | Deployment |
| Container | Pod (a Pod wraps one or more containers) |
| `ports:` | Service |
| `networks:` | Namespace + Service |
| `volumes:` | PersistentVolume / PVC |
| `healthcheck:` | Liveness / Readiness Probe |

---

## What I did

### Part 1 — Cluster setup

```
minikube start --driver=docker
minikube status
kubectl cluster-info
kubectl get nodes
```
Result: `minikube` node in `Ready` state, running Kubernetes `v1.35.1`.

### Part 2 — Inspecting K8s internal components

```
kubectl get all -n kube-system
```
Showed all system components in the `kube-system` namespace — `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, `kube-proxy`, `coredns`, `storage-provisioner` — all `Running`.

### Part 3 — First Pod

```
kubectl run lab01-nginx --image=nginx
kubectl get pods
kubectl describe pod lab01-nginx
kubectl logs lab01-nginx
```
Pod reached `1/1 Running` in ~5 seconds. `describe` showed the full scheduling and startup event chain (`Scheduled → Pulling → Pulled → Created → Started`); `logs` showed nginx output — identical to `docker logs` from Docker Lab 01.

### Part 4 — Naked Pod vs controlled Pod (preview)

```
kubectl delete pod lab01-nginx
kubectl get pods
# → No resources found in default namespace
```
The Pod was deleted and did not come back — because it was a "naked pod" with no controller managing it. In Lab 03 (Deployments), a deleted Pod is automatically replaced. That's the real self-healing in action.

---

## Commands reference

| Command | Purpose |
|---|---|
| `minikube start --driver=docker` | Start a local K8s cluster using Docker as the backend |
| `minikube status` | Check cluster component health |
| `kubectl cluster-info` | Show control plane and CoreDNS endpoints |
| `kubectl get nodes` | List nodes and their status |
| `kubectl get all -n kube-system` | List all resources in the kube-system namespace |
| `kubectl run <name> --image=<image>` | Create a naked Pod imperatively |
| `kubectl get pods` | List Pods in the current namespace |
| `kubectl describe pod <name>` | Full Pod details including scheduling events |
| `kubectl logs <name>` | View container output |
| `kubectl delete pod <name>` | Delete a Pod (won't restart if no controller manages it) |

## Notes

The `Events:` section of `kubectl describe` reveals the scheduling chain in action: `kube-scheduler` assigns the Pod to a node → `kubelet` pulls the image → container is created and started. This is the K8s equivalent of `docker pull` + `docker run`, but orchestrated across potentially hundreds of nodes rather than a single machine.
