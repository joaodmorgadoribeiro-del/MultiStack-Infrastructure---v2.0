# ☸️ Kubernetes Manifests

This folder contains all Kubernetes manifests to deploy the MultiStack Voting App on your EKS cluster.

---

## 📁 Structure

```
k8s/
├── ingress/
│   └── voting-app-ingress.yaml     # NGINX Ingress (HTTP, host-based routing)
├── ingress-tls/
│   └── ingress-tls.yaml            # NGINX Ingress with TLS (HTTPS)
├── postgres/
│   ├── postgres-deployment.yaml
│   ├── postgres-pvc.yaml
│   └── postgres-service.yaml
├── redis/
│   ├── redis-deployment.yaml
│   └── redis-service.yaml
├── result/
│   ├── result-deployment.yaml
│   └── result-service.yaml
├── vote/
│   ├── vote-deployment.yaml
│   └── vote-service.yaml
└── worker/
    ├── worker-deployment.yaml
    └── worker-service.yaml
```

---

## ✅ Prerequisites

Before applying these manifests, make sure you have:

- `kubectl` connected to your EKS cluster
- EBS CSI Driver installed (required for Postgres PVC)
- NGINX Ingress Controller installed via Helm

Verify your connection:

```bash
kubectl get nodes
```

---

## 🚀 Deploy All Services

### 1. Deploy Redis

```bash
kubectl apply -f k8s/redis/
kubectl get pods -l app=redis
```

### 2. Deploy Postgres

```bash
kubectl apply -f k8s/postgres/
kubectl get pods -l app=postgres
kubectl get pvc postgres-pvc
```

> **Note:** The PVC requires the EBS CSI Driver. If it stays `Pending`, run:
> ```bash
> aws eks create-addon \
>   --cluster-name eks-cluster-irene-and-joao \
>   --addon-name aws-ebs-csi-driver \
>   --region us-east-1
> ```

### 3. Deploy Vote, Result and Worker

```bash
kubectl apply -f k8s/vote/
kubectl apply -f k8s/result/
kubectl apply -f k8s/worker/
```

### 4. Apply Ingress

For HTTP only:
```bash
kubectl apply -f k8s/ingress/
```

For HTTPS (after completing the Setup/HTTPS steps):
```bash
kubectl apply -f k8s/ingress-tls/
```

### 5. Verify everything is running

```bash
kubectl get pods
kubectl get services
kubectl get ingress
kubectl get pvc
```

All pods should show `Running` and the Ingress should have an `ADDRESS`.

---

## 🔁 Deploy All at Once

```bash
kubectl apply -f k8s/redis/ \
              -f k8s/postgres/ \
              -f k8s/vote/ \
              -f k8s/result/ \
              -f k8s/worker/ \
              -f k8s/ingress/
```

---

## ⚙️ Environment Variables

Each deployment requires specific env vars to connect the services. These are already set in each `*-deployment.yaml`:

| Service | Key Env Vars |
|---------|-------------|
| vote | `REDIS_HOST=redis-service` · `REDIS_PORT=6379` · `FLASK_ENV=development` |
| result | `PG_HOST=db` · `PG_USER=postgres` · `PG_PASSWORD=postgres` · `PG_DATABASE=postgres` · `PORT=80` |
| worker | `REDIS_HOST=redis-service` · `PG_HOST=db` · `PG_USER` · `PG_PASSWORD` · `PG_DATABASE` |
| postgres | `POSTGRES_USER=postgres` · `POSTGRES_PASSWORD=postgres` · `POSTGRES_DB=postgres` |

> ⚠️ The Postgres service must be named `db` — the result and worker apps look for this hostname.

---

## 🐛 Troubleshooting

**Pods in CrashLoopBackOff:**
```bash
kubectl logs deployment/<name>-deployment
```

**PVC stuck in Pending:**
```bash
kubectl describe pvc postgres-pvc
# Check if EBS CSI Driver is active:
kubectl get pods -n kube-system | grep ebs
```

**502/504 from NGINX:**
```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=30
kubectl get endpoints vote-service result-service
```

**Services not finding each other:**
```bash
# Check env vars are correct
kubectl exec -it deployment/vote-deployment -- env | grep REDIS
kubectl exec -it deployment/result-deployment -- env | grep PG
```

---

## 🧹 Delete Everything

```bash
kubectl delete -f k8s/vote/ \
               -f k8s/result/ \
               -f k8s/worker/ \
               -f k8s/redis/ \
               -f k8s/postgres/ \
               -f k8s/ingress/ \
               -f k8s/ingress-tls/
```

> ⚠️ Deleting the postgres PVC will destroy all vote data permanently.