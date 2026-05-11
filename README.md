# 🗳️ MultiStack Voting App — EKS Deployment

A cloud-native, microservices-based voting application deployed on **Amazon EKS**, with a full CI/CD pipeline via **GitHub Actions**, NGINX Ingress routing, Kubernetes Secrets, and HTTPS via **cert-manager + Let's Encrypt**.

---

## 📐 Architecture

```
                          ┌─────────────────────────────────────────────────┐
                          │                  AWS Cloud                       │
                          │                                                  │
   Browser ──────────────▶│  Route 53 DNS                                   │
                          │  (vote.*.ironlabs.online)                        │
                          │  (result.*.ironlabs.online)                      │
                          │         │                                        │
                          │         ▼                                        │
                          │  ┌─────────────┐                                │
                          │  │  AWS NLB    │  (Network Load Balancer)        │
                          │  └──────┬──────┘                                │
                          │         │                                        │
                          │  ┌──────▼──────────────────────────────────┐   │
                          │  │           Amazon EKS Cluster             │   │
                          │  │                                          │   │
                          │  │  ┌────────────────────────────────────┐ │   │
                          │  │  │     ingress-nginx (Namespace)      │ │   │
                          │  │  │   NGINX Ingress Controller Pod     │ │   │
                          │  │  └──────────────┬─────────────────────┘ │   │
                          │  │                 │                        │   │
                          │  │    ┌────────────┴────────────┐          │   │
                          │  │    │                         │          │   │
                          │  │    ▼                         ▼          │   │
                          │  │ ┌──────────┐         ┌─────────────┐   │   │
                          │  │ │   vote   │         │   result    │   │   │
                          │  │ │  (Flask) │         │  (Node.js + │   │   │
                          │  │ │  :80     │         │  Socket.io) │   │   │
                          │  │ └────┬─────┘         └──────┬──────┘   │   │
                          │  │      │                       │          │   │
                          │  │      ▼                       ▼          │   │
                          │  │ ┌──────────┐         ┌─────────────┐   │   │
                          │  │ │  Redis   │         │  Postgres   │   │   │
                          │  │ │  :6379   │         │   :5432     │   │   │
                          │  │ └────┬─────┘         └──────┬──────┘   │   │
                          │  │      │                       ▲          │   │
                          │  │      ▼                       │          │   │
                          │  │ ┌──────────────────────────────────┐   │   │
                          │  │ │          worker (.NET)           │   │   │
                          │  │ │  reads Redis → writes Postgres   │   │   │
                          │  │ └──────────────────────────────────┘   │   │
                          │  └──────────────────────────────────────── ┘   │
                          └─────────────────────────────────────────────────┘
                                              │
                          ┌───────────────────▼──────────────────────────────┐
                          │              GitHub Actions CI/CD                 │
                          │  push → build images → push to Docker Hub        │
                          │       → kubectl apply → EKS updated              │
                          └──────────────────────────────────────────────────┘
```

---

## 🧩 Services

| Service | Language | Role | Internal Port |
|---------|----------|------|---------------|
| `vote` | Python (Flask) | Frontend — cast your vote | 80 |
| `result` | Node.js + Socket.io | Frontend — live results | 80 |
| `worker` | .NET | Reads Redis, writes to Postgres | — |
| `redis` | Redis | Vote queue (in-memory) | 6379 |
| `db` | PostgreSQL | Persistent vote storage | 5432 |

---

## 🚀 Prerequisites

- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [eksctl](https://eksctl.io/)
- [AWS CLI](https://aws.amazon.com/cli/) configured with your credentials
- A domain managed in **Route 53**
- Docker Hub account

---

## 📦 Setup

### 1. Configure environment variables

```bash
export NAME=yourname          # lowercase, no spaces
export CLUSTER_NAME=$(aws eks list-clusters \
  --query "clusters[?contains(@, '${STUDENT_NAME}')] | [0]" \
  --output text)
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

### 2. Update kubeconfig

```bash
aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION
```

### 3. Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update ingress-nginx

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"=internet-facing \
  --set controller.allowSnippetAnnotations=true \
  --wait --timeout=5m
```

### 4. Create Kubernetes Secrets

Store database credentials securely — never in plain manifests:

```bash
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=yourSecurePassword
```

Verify:

```bash
kubectl get secrets
kubectl describe secret db-credentials
```

### 5. Deploy all services

```bash
kubectl apply -f k8s/
```

### 6. Verify everything is running

```bash
kubectl get pods
kubectl get services
kubectl get ingress
```

---

## 🌐 Ingress Configuration

Traffic is routed by hostname:

| Host | Service |
|------|---------|
| `vote.<name>.ironlabs.online` | `vote-service:80` |
| `result.<name>.ironlabs.online` | `result-service:80` |

The Ingress resource lives in `k8s/ingress.yaml`.

> **Note:** The result service uses Socket.io (WebSockets) for live vote updates. Make sure the Ingress Controller has WebSocket annotations if you experience connection issues.

---

## 🔒 HTTPS with cert-manager

TLS certificates are issued automatically via **Let's Encrypt** using cert-manager and a `ClusterIssuer`.

### Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

### Create a ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Certificates are renewed automatically before expiry.

---

## ⚙️ CI/CD Pipeline (GitHub Actions)

The pipeline triggers on every push to `main`:

1. Builds Docker images for `vote`, `result`, and `worker`
2. Pushes images to Docker Hub
3. Authenticates to AWS and updates kubeconfig
4. Applies all Kubernetes manifests via `kubectl apply -f k8s/`

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `AWS_ACCESS_KEY_ID` | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret key |

---

## 🗂️ Repository Structure

```
.
├── k8s/
│   ├── vote-deployment.yaml
│   ├── result-deployment.yaml
│   ├── worker-deployment.yaml
│   ├── redis-deployment.yaml
│   ├── postgres-deployment.yaml
│   └── ingress.yaml
├── vote/
│   ├── Dockerfile
│   └── app.py
├── result/
│   ├── Dockerfile
│   └── server.js
├── worker/
│   └── Dockerfile
└── .github/
    └── workflows/
        └── ci-cd-pipeline.yml
```

---

## ✅ Validation

Once deployed, cast a vote and watch it flow through the system:

```
vote app → Redis → worker → Postgres → result app (live via WebSocket)
```

Access:
- 🗳️ `https://vote.<name>.ironlabs.online`
- 📊 `https://result.<name>.ironlabs.online`

---

## Authors

| Author | Links |
|---|---|
| **João Ribeiro** | [GitHub](https://github.com/joaodmorgadoribeiro-del) · [LinkedIn](https://www.linkedin.com/in/joaoribeiro9595) |
| **Irene Romero** | [GitHub](https://github.com/ireneromero95) · [LinkedIn](http://linkedin.com/in/irene-romero-mart%C3%ADnez-0b6215119/) |

---

*Ironhack Cloud & DevOps Bootcamp — Capstone Project*
