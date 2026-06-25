# Lab 04 — Services

## Goal

Understand why Pods need a Service in front of them, and learn the three Service types — ClusterIP, NodePort, and LoadBalancer — and when to use each.

---

## The Problem Services Solve

After Lab 03, we had Pods running — but with two fundamental problems:

**Problem 1: Pod IPs are temporary.**
Every Pod gets a cluster-internal IP (`10.244.x.x`). When a Pod is deleted and recreated (by a Deployment), it gets a *new* IP. You can't hardcode Pod IPs anywhere — they change constantly.

**Problem 2: Pods are not reachable from outside the cluster.**
Pod IPs are only visible inside the cluster. You can't open a browser and connect to `10.244.0.3`.

**Service solves both:**

```
Outside world → Service (stable IP + DNS name) → Pod-1
                                                 → Pod-2
                                                 → Pod-3
```

- The Service IP never changes, even as Pods come and go
- The Service finds the right Pods using **label selectors**
- Depending on the Service type, it can expose Pods externally

---

## How a Service finds its Pods — label selectors

The Service doesn't track Pod IPs directly. Instead, it uses labels:

```yaml
# In the Service:
selector:
  app: lab04-nginx     ← "send traffic to any Pod with this label"

# In the Deployment template:
labels:
  app: lab04-nginx     ← "I have this label"
```

This is the key connection between a Service and a Deployment. If the labels don't match, the Service sends traffic nowhere.

---

## Service types

| Type | Reachable from | Use case |
|---|---|---|
| `ClusterIP` | Inside the cluster only | Internal services (databases, backend APIs) |
| `NodePort` | Host machine via a specific port (30000-32767) | Local development, testing |
| `LoadBalancer` | Public internet via cloud load balancer | Production on AWS/GCP/Azure |

---

## What I did

### Part 1 — NodePort Service

```yaml
# lab04-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab04-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lab04-nginx
  template:
    metadata:
      labels:
        app: lab04-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - containerPort: 80
```

```yaml
# lab04-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: lab04-nginx-service
spec:
  type: NodePort
  selector:
    app: lab04-nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

Port mapping explained:
```
Host machine :30080 → Service :80 → Pod :80
```

- `nodePort: 30080` — the port opened on the host machine
- `port: 80` — the Service's own port inside the cluster
- `targetPort: 80` — the port the container is actually listening on

```
kubectl apply -f lab04-deployment.yaml
kubectl apply -f lab04-service.yaml
kubectl get pods
kubectl get service
```

Result:
```
lab04-nginx-service   NodePort   10.96.218.66   <none>   80:30080/TCP
```

### Part 2 — Accessing the Service

Used `kubectl port-forward` to access the Service locally:
```
kubectl port-forward service/lab04-nginx-service 8080:80
```
Then visited `http://localhost:8080` → nginx welcome page confirmed.

### Part 3 — port-forward vs real load balancing

Wrote unique content to each Pod:
```
kubectl exec <pod-1> -- sh -c "echo 'Pod-1' > /usr/share/nginx/html/index.html"
kubectl exec <pod-2> -- sh -c "echo 'Pod-2' > /usr/share/nginx/html/index.html"
kubectl exec <pod-3> -- sh -c "echo 'Pod-3' > /usr/share/nginx/html/index.html"
```

Tested repeatedly via `port-forward` → always returned `Pod-1`.

Then port-forwarded directly to the second Pod on a different port:
```
kubectl port-forward pod/<pod-2> 8081:80
```
→ always returned `Pod-2`.

**Key finding:** `kubectl port-forward` opens a fixed tunnel to one specific Pod or Service endpoint — it does not perform load balancing. Real load balancing requires `minikube tunnel` locally, or a real cloud cluster with a LoadBalancer Service. `port-forward` is a debugging tool, not a traffic management tool.

### Part 4 — ClusterIP comparison

```
kubectl expose deployment lab04-nginx --name=lab04-clusterip --port=80 --type=ClusterIP
kubectl get service
```

Result:
```
lab04-clusterip       ClusterIP   10.96.77.241   <none>   80/TCP
lab04-nginx-service   NodePort    10.96.218.66   <none>   80:30080/TCP
```

ClusterIP has no external port mapping and no external IP — only reachable from inside the cluster. This is the right type for internal services (like a database) that should never be exposed outside the cluster.

### Cleanup

```
kubectl delete deployment lab04-nginx
kubectl delete service lab04-nginx-service lab04-clusterip
```

---

## Key concepts

| Concept | Description |
|---|---|
| **Service** | A stable network endpoint in front of a dynamic set of Pods |
| **Label selector** | How a Service identifies which Pods to route traffic to |
| **ClusterIP** | Internal-only Service — no external access |
| **NodePort** | Opens a port on the host machine, routes to the Service |
| **LoadBalancer** | Creates a cloud load balancer (ELB on AWS); seen in the EKS final project |
| **`port-forward`** | A kubectl debugging tool — opens a local tunnel to a Pod or Service, not real load balancing |

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Create or update a Service from YAML |
| `kubectl get service` | List all Services in the namespace |
| `kubectl expose deployment <name> --port=<n> --type=<type>` | Create a Service imperatively |
| `kubectl port-forward service/<name> <local>:<remote>` | Open a local tunnel to a Service (debug only) |
| `kubectl port-forward pod/<name> <local>:<remote>` | Open a local tunnel to a specific Pod |
| `kubectl exec <pod> -- <command>` | Run a command inside a Pod |
| `kubectl delete service <name>` | Delete a Service |

## Notes

In production on AWS (EKS), using `type: LoadBalancer` automatically provisions an Elastic Load Balancer and writes its DNS address into the `EXTERNAL-IP` field. This is what makes a K8s service publicly accessible without any extra configuration. We'll see this in the EKS final project.
