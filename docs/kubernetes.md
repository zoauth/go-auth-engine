# Kubernetes Deployment

ZoAuth can be deployed on any Kubernetes cluster (GKE, EKS, AKS, k3s, etc.).

---

## Prerequisites

- Kubernetes 1.25+
- `kubectl` configured
- A PostgreSQL database (external or in-cluster)
- A Secret with ZoAuth credentials

---

## Quick Deploy (kubectl)

### 1. Create namespace and secret

```bash
kubectl create namespace zoauth

kubectl create secret generic zoauth-secrets \
  --namespace zoauth \
  --from-literal=JWK_ENCRYPTION_PASSWORD="$(openssl rand -base64 32)" \
  --from-literal=JWK_ENCRYPTION_SALT="$(openssl rand -hex 8)" \
  --from-literal=DATABASE_URL="postgres://zoauth:yourpassword@postgres-service:5432/zoauth?sslmode=require"
```

### 2. Apply the deployment

```bash
kubectl apply -f https://github.com/zoauth/go-auth-engine/releases/latest/download/kubernetes-manifests.yaml
```

Or use the example manifests in [examples/kubernetes/](../examples/kubernetes/).

---

## Helm Chart

```bash
helm install zoauth oci://ghcr.io/zoauth/charts/zoauth \
  --namespace zoauth \
  --create-namespace \
  --set config.jwtIssuer=https://auth.example.com \
  --set config.springEnabled=false \
  --set secrets.existingSecret=zoauth-secrets \
  --set ingress.enabled=true \
  --set ingress.host=auth.example.com \
  --set replicaCount=2
```

### Key Helm values

See [examples/kubernetes/helm-values.yaml](../examples/kubernetes/helm-values.yaml) for a production-ready values file.

| Value | Default | Description |
|---|---|---|
| `image.repository` | `ghcr.io/zoauth/go-auth-engine` | Container image |
| `image.tag` | `latest` | Image tag |
| `replicaCount` | `1` | Number of replicas |
| `config.jwtIssuer` | — | OIDC issuer URL (required) |
| `config.springEnabled` | `false` | Standalone mode |
| `config.logFormat` | `json` | Log format |
| `config.logLevel` | `info` | Log level |
| `secrets.existingSecret` | — | Name of existing K8s secret |
| `ingress.enabled` | `false` | Enable Kubernetes Ingress |
| `ingress.host` | — | Hostname for Ingress |
| `ingress.tls` | `[]` | TLS config for Ingress |
| `resources.requests.memory` | `128Mi` | Memory request |
| `resources.limits.memory` | `512Mi` | Memory limit |
| `postgresql.enabled` | `false` | Deploy bundled PostgreSQL (dev only) |
| `redis.enabled` | `false` | Deploy bundled Redis |

---

## Production Checklist

- [ ] `replicaCount` ≥ 2 for high availability
- [ ] All replicas point to the **same** PostgreSQL instance
- [ ] If using Redis, all replicas point to the **same** Redis
- [ ] `JWK_ENCRYPTION_PASSWORD` is identical across all replicas
- [ ] Liveness probe: `GET /health/live` (starts failing after 5s)
- [ ] Readiness probe: `GET /health/ready` (fails if DB disconnected)
- [ ] `PodDisruptionBudget` set to `minAvailable: 1`
- [ ] Resource limits set to prevent OOM kills
- [ ] `topologySpreadConstraints` for zone spreading

---

## Example Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zoauth
  namespace: zoauth
spec:
  replicas: 2
  selector:
    matchLabels:
      app: zoauth
  template:
    metadata:
      labels:
        app: zoauth
    spec:
      containers:
        - name: zoauth
          image: ghcr.io/zoauth/go-auth-engine:latest
          ports:
            - containerPort: 8080
            - containerPort: 9090  # metrics
          env:
            - name: JWT_ISSUER
              value: "https://auth.example.com"
            - name: SPRING_ENABLED
              value: "false"
            - name: LOG_FORMAT
              value: "json"
            - name: JWK_ENCRYPTION_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: zoauth-secrets
                  key: JWK_ENCRYPTION_PASSWORD
            - name: JWK_ENCRYPTION_SALT
              valueFrom:
                secretKeyRef:
                  name: zoauth-secrets
                  key: JWK_ENCRYPTION_SALT
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: zoauth-secrets
                  key: DATABASE_URL
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```
