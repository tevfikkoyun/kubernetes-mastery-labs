# Lab 06 — Namespaces

## Goal

Understand how Kubernetes Namespaces provide logical isolation between workloads, environments, and teams — and how cross-namespace DNS resolution works.

---

## What is a Namespace?

A Namespace is a virtual cluster inside a Kubernetes cluster. It groups related resources together and isolates them from resources in other Namespaces.

```
┌─────────────────────────────────────────────────────────┐
│                     CLUSTER                             │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────┐  │
│  │     dev     │  │   staging   │  │  kube-system  │  │
│  │             │  │             │  │               │  │
│  │  nginx (1)  │  │  nginx (2)  │  │  apiserver    │  │
│  │  Service    │  │  Service    │  │  etcd         │  │
│  └─────────────┘  └─────────────┘  └───────────────┘  │
│                                                         │
│  Same resource names, completely isolated               │
└─────────────────────────────────────────────────────────┘
```

---

## Default Namespaces

| Namespace | Purpose |
|---|---|
| `default` | Where resources land when no namespace is specified |
| `kube-system` | K8s internal components (apiserver, etcd, coredns, kube-proxy) |
| `kube-public` | Publicly readable resources, rarely used directly |
| `kube-node-lease` | Node heartbeat objects for health tracking |
| `local-path-storage` | Minikube's local storage provisioner (Lab 07) |

---

## What I did

### Part 1 — Listing namespaces

```
kubectl get namespaces
```

Showed 5 default namespaces. All resources created so far landed in `default`.

### Part 2 — Creating namespaces

```yaml
# lab06-namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

The `---` separator allows multiple resources in a single YAML file.

```
kubectl apply -f lab06-namespaces.yaml
kubectl get namespaces
```

### Part 3 — Deploying to specific namespaces

Deployed the same application (`nginx`) to both namespaces with different replica counts:

```yaml
# dev: 1 replica, ENV_NAME=development
metadata:
  name: nginx
  namespace: dev

# staging: 2 replicas, ENV_NAME=staging
metadata:
  name: nginx
  namespace: staging
```

```
kubectl apply -f lab06-dev-deployment.yaml
kubectl apply -f lab06-staging-deployment.yaml

kubectl get pods          # → No resources (default namespace is empty)
kubectl get pods -n dev   # → 1 Pod
kubectl get pods -n staging  # → 2 Pods
```

Same Deployment name (`nginx`) in both — valid because names only need to be unique *within* a namespace.

### Part 4 — Cross-namespace DNS resolution

Added a Service to each namespace:
```
kubectl apply -f lab06-services.yaml
```

K8s DNS format for cross-namespace access:
```
<service-name>.<namespace>.svc.cluster.local
```

Tested from a Pod in `dev`:
```
# Same namespace (short name):
kubectl exec -n dev <pod> -- wget -qO- http://nginx
# → nginx welcome page ✅ (reached dev's own nginx Service)

# Different namespace (full DNS name):
kubectl exec -n dev <pod> -- wget -qO- http://nginx.staging.svc.cluster.local
# → nginx welcome page ✅ (reached staging's nginx Service)
```

### Part 5 — Viewing all namespaces at once

```
kubectl get all --all-namespaces
# or shorthand:
kubectl get all -A
```

Shows resources across every namespace in one output, with a `NAMESPACE` column added. Without `-A` or `-n <name>`, `kubectl get` always defaults to the `default` namespace.

### Cleanup

```
kubectl delete namespace dev staging
```
Deleting a namespace automatically deletes **all resources inside it** — Deployments, Pods, Services, ConfigMaps, Secrets. The cleanest way to tear down an entire environment.

---

## Key concepts

| Concept | Description |
|---|---|
| **Namespace** | A logical boundary grouping related K8s resources |
| **Name uniqueness** | Resource names must be unique within a namespace, but the same name can exist in different namespaces |
| **`-n <namespace>`** | Flag to target a specific namespace in any `kubectl` command |
| **`-A` / `--all-namespaces`** | Show resources across all namespaces |
| **DNS within namespace** | `http://<service-name>` — short form, resolves within the same namespace |
| **DNS across namespaces** | `http://<service>.<namespace>.svc.cluster.local` — full form, works cluster-wide |

---

## Namespace isolation — an important nuance

Namespaces provide **logical** isolation (resource grouping, name scoping, RBAC access control), but **not network isolation** by default. As demonstrated: a Pod in `dev` could still reach a Service in `staging` using the full DNS name.

For true network isolation (blocking cross-namespace traffic), **NetworkPolicy** resources are used — an advanced K8s topic beyond this lab's scope.

Common real-world uses for namespaces:
- **Environment separation**: `dev`, `staging`, `production`
- **Team separation**: `team-frontend`, `team-backend`, `team-data`
- **Resource quotas**: limit CPU/memory per namespace so one team can't starve another

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl get namespaces` | List all namespaces |
| `kubectl apply -f <file>` | Create a namespace (or any resource) from YAML |
| `kubectl get pods -n <namespace>` | List Pods in a specific namespace |
| `kubectl get all -A` | List all resources across all namespaces |
| `kubectl exec -n <namespace> <pod> -- <cmd>` | Run a command in a Pod in a specific namespace |
| `kubectl delete namespace <name>` | Delete a namespace and all its resources |
