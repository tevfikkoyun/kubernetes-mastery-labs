# kubernetes-mastery-labs

Hands-on Kubernetes labs from fundamentals to production-style autoscaling — covering every core concept needed to deploy and operate containerized workloads on Kubernetes, locally with Minikube/Docker Desktop and on AWS EKS.

Each lab builds directly on the last, and each maps to a real-world pattern encountered in production environments.

## Labs

| Lab | Topic |
|---|---|
| [Lab 01](./lab01-fundamentals) | Kubernetes Fundamentals — cluster architecture, kubectl, first Pod |
| [Lab 02](./lab02-pods) | Pods — YAML manifests, lifecycle, CrashLoopBackOff |
| [Lab 03](./lab03-deployments) | Deployments — self-healing, scaling, rolling update, rollback |
| [Lab 04](./lab04-services) | Services — ClusterIP, NodePort, LoadBalancer |
| [Lab 05](./lab05-configmaps-secrets) | ConfigMaps and Secrets — environment variables and volume mounts |
| [Lab 06](./lab06-namespaces) | Namespaces — resource isolation, cross-namespace DNS |
| [Lab 07](./lab07-persistent-volumes) | Persistent Volumes — PVC, PV, StorageClass, data survival across Pod deletion |
| [Lab 08](./lab08-ingress) | Ingress — path-based routing, Nginx Ingress Controller |
| [Lab 09](./lab09-probes) | Liveness and Readiness Probes — health checks, failure behavior |
| [Lab 10](./lab10-resource-limits) | Resource Limits and Requests — OOMKilled, LimitRange |
| [Lab 11](./lab11-rolling-updates) | Rolling Updates and Rollbacks — RollingUpdate vs Recreate, revision history |
| [Lab 12](./lab12-hpa) | Horizontal Pod Autoscaler — CPU-based autoscaling, cooldown periods |

Each lab folder contains its own `README.md` documenting the goal, all commands run, key concepts, and connections to earlier labs.

## Highlights

- **Self-healing demonstrated end-to-end** — deleted a Pod from a running Deployment and watched K8s replace it within 1 second, compared directly to a "naked Pod" that stayed deleted — [Lab 03](./lab03-deployments)
- **CrashLoopBackOff diagnosed and explained** — deliberately triggered the failure mode, traced it through `kubectl describe` events and `kubectl logs`, and connected it to the Docker Lab 09 debugging methodology — [Lab 02](./lab02-pods)
- **Cross-namespace DNS resolution** — proved that `http://nginx` resolves within a namespace while `http://nginx.staging.svc.cluster.local` reaches across namespace boundaries — [Lab 06](./lab06-namespaces)
- **Data persistence across Pod deletion** — wrote data to a PVC, deleted and recreated the Pod, confirmed the data survived — [Lab 07](./lab07-persistent-volumes)
- **Path-based Ingress routing** — single port 80 entry point routing `/frontend` and `/backend` to separate Services, tested via `Host` header injection without modifying the hosts file — [Lab 08](./lab08-ingress)
- **OOMKilled observed in real time** — container attempted to allocate 200MB against a 50Mi limit and was killed by the kernel immediately — [Lab 10](./lab10-resource-limits)
- **HPA full cycle observed** — load generator pushed CPU to 88% → HPA scaled from 1 to 2 Pods → load removed → 15-second cooldown → scaled back to 1 Pod — [Lab 12](./lab12-hpa)

## Docker → Kubernetes mental map

| Docker / Compose | Kubernetes equivalent |
|---|---|
| `docker run` | `kubectl run` |
| `docker-compose.yml` | YAML manifests (Deployment + Service) |
| Container | Pod |
| `services:` | Deployment |
| `ports:` | Service |
| `networks:` | Namespace + Service DNS |
| `volumes:` | PersistentVolume / PVC |
| `healthcheck:` | Liveness + Readiness Probes |
| `.env` file | ConfigMap + Secret |
| Nginx reverse proxy | Ingress + Ingress Controller |
| `--memory` / `--cpus` | `resources.requests` / `resources.limits` |
| Manual `docker scale` | Horizontal Pod Autoscaler |

## Environment

Labs 01–12 run on a local single-node cluster using **Minikube** (`--driver=docker`) and **Docker Desktop's built-in Kubernetes**, both on Kubernetes `v1.34–1.35`. All `kubectl` commands work identically on AWS EKS — only the infrastructure layer changes.

## What's next

Three Kubernetes projects apply these concepts to real workloads, culminating in a full deployment to **AWS EKS** provisioned with Terraform — bringing together every skill from this series alongside the [docker-mastery-labs](https://github.com/tevfikkoyun/docker-mastery-labs) and [fullstack-aws-deployment](https://github.com/tevfikkoyun/fullstack-aws-deployment) projects.