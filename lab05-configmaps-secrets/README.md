# Lab 05 — ConfigMaps and Secrets

## Goal

Learn how Kubernetes separates configuration from application code using ConfigMaps (for non-sensitive data) and Secrets (for sensitive data), and how to inject them into Pods as environment variables or mounted files.

---

## The Problem

Hardcoding configuration values (database names, ports, passwords, API keys) inside a container image is bad practice:
- The same image can't be reused across environments (dev, staging, prod)
- Sensitive values end up in version control
- Changing a value requires rebuilding the image

In Docker (Lab 08), we solved this with `.env` files. In Kubernetes, the solution is **ConfigMaps** and **Secrets**.

---

## ConfigMap vs Secret

| | ConfigMap | Secret |
|---|---|---|
| Purpose | Non-sensitive configuration | Sensitive data (passwords, tokens, keys) |
| Storage | Plain text in etcd | Base64-encoded in etcd |
| Use cases | App settings, ports, log levels, config files | DB passwords, API keys, TLS certificates |
| YAML `data` values | Plain text | Base64-encoded |

**Important:** Base64 is *encoding*, not *encryption*. Secrets are not truly secure by default — they're just base64 in etcd. For real security, enable etcd encryption at rest or use an external secrets manager (AWS Secrets Manager, HashiCorp Vault). In K8s, Secrets exist primarily to separate sensitive data from regular config and to restrict access via RBAC.

---

## Two ways to inject ConfigMaps/Secrets into Pods

### Method 1 — Environment variables

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: lab05-config    # ConfigMap name
        key: APP_ENV          # Key inside the ConfigMap
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: lab05-secret    # Secret name
        key: DB_PASSWORD      # Key inside the Secret
```

The Pod sees these as regular environment variables — identical to how Docker's `.env` file injects values.

### Method 2 — Volume mount (as files)

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /config        # Directory inside the container
volumes:
  - name: config-volume
    configMap:
      name: lab05-config      # Each key becomes a file at /config/<key>
```

Every key in the ConfigMap becomes a separate file under the mount path:
```
/config/APP_ENV
/config/APP_PORT
/config/APP_LOG_LEVEL
/config/welcome_message
```

Use this method when your application reads configuration from files rather than environment variables (e.g. nginx config, application.properties, etc.).

---

## What I did

### Part 1 — Creating a ConfigMap

```yaml
# lab05-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lab05-config
data:
  APP_ENV: production
  APP_PORT: "3000"
  APP_LOG_LEVEL: info
  welcome_message: "Hello from ConfigMap!"
```

### Part 2 — Creating a Secret

```yaml
# lab05-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: lab05-secret
type: Opaque
data:
  DB_PASSWORD: bXlzZWNyZXRwYXNzd29yZA==
  API_KEY: c3VwZXItc2VjcmV0LWFwaS1rZXk=
```

Values are base64-encoded. To encode a value in PowerShell:
```powershell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("mysecretpassword"))
```

### Part 3 — Injecting as environment variables

```yaml
# lab05-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab05-pod
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "env && sleep 3600"]
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: lab05-config
              key: APP_ENV
        - name: APP_PORT
          valueFrom:
            configMapKeyRef:
              name: lab05-config
              key: APP_PORT
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: lab05-secret
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: lab05-secret
              key: API_KEY
```

```
kubectl apply -f lab05-pod.yaml
kubectl logs lab05-pod
```

Result in logs (selected):
```
APP_ENV=production
APP_PORT=3000
DB_PASSWORD=mysecretpassword
API_KEY=super-secret-api-key
```

Secret values were automatically base64-decoded by K8s before injection — the Pod sees the plain text value, not the encoded one.

### Part 4 — Injecting as a volume

```yaml
# lab05-volume-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab05-volume-pod
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "cat /config/welcome_message && sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /config
  volumes:
    - name: config-volume
      configMap:
        name: lab05-config
```

```
kubectl apply -f lab05-volume-pod.yaml
kubectl exec lab05-volume-pod -- ls /config
kubectl exec lab05-volume-pod -- cat /config/welcome_message
```

Result:
```
# ls /config:
APP_ENV  APP_LOG_LEVEL  APP_PORT  welcome_message

# cat /config/welcome_message:
Hello from ConfigMap!
```

### Cleanup

```
kubectl delete pod lab05-pod lab05-volume-pod
kubectl delete configmap lab05-config
kubectl delete secret lab05-secret
```

---

## Docker Lab 08 → Kubernetes comparison

| Docker (Lab 08) | Kubernetes (Lab 05) |
|---|---|
| `.env` file | ConfigMap + Secret |
| `environment:` in Compose | `env.valueFrom.configMapKeyRef` / `secretKeyRef` |
| `.env` excluded from git | Secret stored in etcd (base64), not in the repo |
| `DB_PASSWORD=mysecretpassword` | Secret `data.DB_PASSWORD: bXlzZWNyZXRwYXNzd29yZA==` |

---

## Commands reference

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>` | Create a ConfigMap or Secret from YAML |
| `kubectl get configmap` | List ConfigMaps |
| `kubectl get secret` | List Secrets |
| `kubectl describe configmap <name>` | Show ConfigMap keys and values |
| `kubectl describe secret <name>` | Show Secret keys (values are hidden) |
| `kubectl exec <pod> -- env` | List environment variables inside a Pod |
| `kubectl exec <pod> -- ls /config` | List files in a mounted ConfigMap volume |
| `kubectl delete configmap <name>` | Delete a ConfigMap |
| `kubectl delete secret <name>` | Delete a Secret |

## Notes

Secrets in Kubernetes are base64-encoded, not encrypted. Anyone with access to the cluster can decode them. For production workloads with strict security requirements, use:
- **etcd encryption at rest** (K8s built-in)
- **AWS Secrets Manager** with the Secrets Store CSI Driver (relevant for the EKS final project)
- **HashiCorp Vault**

This is the same principle as the `.env` file pattern from Docker Lab 08 — separate secrets from code — but at the cluster level with more access control options.
