# 🗳️ MultiStack Voting App — EKS Deployment

A cloud-native, microservices-based voting application deployed on Amazon EKS, with a full CI/CD pipeline via GitHub Actions, NGINX Ingress routing, Kubernetes Secrets, and HTTPS via cert-manager + Let's Encrypt.

---

## 📐 Architecture

```
                          ┌─────────────────────────────────────────────────┐
                          │                  AWS Cloud                       │
                          │                                                  │
   Browser ──────────────▶│  Route 53 DNS                                   │
                          │  vote.joao.and.irene.ironlabs.online             │
                          │  result.joao.and.irene.ironlabs.online           │
                          │         │                                        │
                          │         ▼                                        │
                          │  ┌─────────────┐                                │
                          │  │  AWS NLB    │  (Network Load Balancer)        │
                          │  └──────┬──────┘                                │
                          │         │                                        │
                          │  ┌──────▼──────────────────────────────────┐   │
                          │  │           Amazon EKS Cluster             │   │
                          │  │           eks-cluster-irene-and-joao     │   │
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
                          │       → kubectl set image → EKS updated          │
                          └──────────────────────────────────────────────────┘
```

---

## 🧩 Services

| Service | Language | Role | Internal Port |
|---------|----------|------|---------------|
| vote | Python (Flask) | Frontend — cast your vote | 80 |
| result | Node.js + Socket.io | Frontend — live results | 80 |
| worker | .NET | Reads Redis, writes to Postgres | — |
| redis | Redis | Vote queue (in-memory) | 6379 |
| db | PostgreSQL | Persistent vote storage | 5432 |

---

## 🚀 Prerequisites

- `kubectl`
- `helm`
- `eksctl`
- AWS CLI configured with valid credentials
- A domain managed in Route 53
- Docker Hub account

---

## 📦 Setup

### 1. Configure AWS credentials

```bash
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
export AWS_REGION="us-east-1"
```

### 2. Update kubeconfig

```bash
aws eks update-kubeconfig --name eks-cluster-irene-and-joao --region us-east-1
```

### 3. Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update ingress-nginx

helm upgrade --install joao-and-irene ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"=internet-facing \
  --set controller.allowSnippetAnnotations=true \
  --wait --timeout=5m
```

### 4. Install EBS CSI Driver

Required for the Postgres PersistentVolumeClaim:

```bash
aws eks create-addon \
  --cluster-name eks-cluster-irene-and-joao \
  --addon-name aws-ebs-csi-driver \
  --region us-east-1
```

### 5. Create Kubernetes Secrets

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

### 6. Deploy all services

```bash
kubectl apply -f k8s/
```

### 7. Verify everything is running

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
| `vote.joao.and.irene.ironlabs.online` | vote-service:80 |
| `result.joao.and.irene.ironlabs.online` | result-service:80 |

The Ingress resource lives in `k8s/ingress/ingress.yaml`.

> **Note:** The result service uses Socket.io (WebSockets) for live vote updates. The Ingress includes WebSocket timeout annotations to handle persistent connections.

---

## 🔒 HTTPS with cert-manager

TLS certificates are issued automatically via Let's Encrypt using cert-manager and a ClusterIssuer.

### Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update jetstack

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \
  --set crds.enabled=true \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::686699774218:role/CertManagerRole-eks-cluster-irene-and-joao" \
  --wait --timeout=5m
```

### Create ClusterIssuers

```bash
# Staging (for testing — no rate limits)
kubectl apply -f setup/cluster-issuer.sh

# Or manually:
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: joaoribeiro9595@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - dns01:
          route53:
            region: us-east-1
            hostedZoneID: Z042372728MB5VI4H04IG
EOF
```

Certificates are renewed automatically before expiry.

---

## ⚙️ CI/CD Pipeline (GitHub Actions)

The pipeline triggers on every push to `main`:

1. Builds Docker images for `vote`, `result`, and `worker`
2. Pushes images to Docker Hub
3. Authenticates to AWS and updates kubeconfig
4. Updates deployments with `kubectl set image`
5. Waits for rollout with `kubectl rollout status`

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `AWS_ACCESS_KEY_ID` | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret key |
| `AWS_SESSION_TOKEN` | AWS session token (required for Academy credentials) |

---

## 🗂️ Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── ci-cd-pipeline.yaml
├── images/
│   ├── result/
│   │   ├── views/
│   │   ├── .dockerignore
│   │   ├── Dockerfile
│   │   ├── package-lock.json
│   │   ├── package.json
│   │   └── server.js
│   ├── vote/
│   │   ├── static/
│   │   ├── templates/
│   │   ├── app.py
│   │   ├── Dockerfile
│   │   └── requirements.txt
│   └── worker/
│       ├── obj/
│       ├── .dockerignore
│       ├── Dockerfile
│       ├── Program.cs
│       └── Worker.csproj
├── k8s/
│   ├── ingress/
│   │   └── voting-app-ingress.yaml
│   ├── ingress-tls/
│   │   └── ingress-tls.yaml
│   ├── postgres/
│   │   ├── postgres-deployment.yaml
│   │   ├── postgres-pvc.yaml
│   │   └── postgres-service.yaml
│   ├── redis/
│   │   ├── redis-deployment.yaml
│   │   └── redis-service.yaml
│   ├── result/
│   │   ├── result-deployment.yaml
│   │   └── result-service.yaml
│   ├── vote/
│   │   ├── vote-deployment.yaml
│   │   └── vote-service.yaml
│   └── worker/
│       ├── worker-deployment.yaml
│       └── worker-service.yaml
│   └── ingress.yaml
├── Setup/
│   ├── install-cert-manager.sh
│   ├── install-clusterissuers.sh
│   ├── install-external-dns.sh
│   ├── install-nginx-controller.sh
│   └── install-oidc-iam.sh
└── README.md
```

---

## ✅ Validation

Once deployed, cast a vote and watch it flow through the system:

```
vote app → Redis → worker → Postgres → result app (live via WebSocket)
```

Access:

- 🗳️ **https://vote.joao.and.irene.ironlabs.online**
- 📊 **https://result.joao.and.irene.ironlabs.online**

---

## 👥 Authors

| Author | Links |
|--------|-------|
| João Ribeiro | [GitHub](https://github.com/joaodmorgadoribeiro-del) |
| Irene Romero | [GitHub](https://github.com/ireneromero95) |

**Ironhack Cloud & DevOps Bootcamp — Capstone Project 2026**
