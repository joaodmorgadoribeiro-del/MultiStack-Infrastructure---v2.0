# 🔒 HTTPS Setup — cert-manager + Let's Encrypt + Route 53

This guide configures automatic TLS certificates for:
- `https://vote.joao.and.irene.ironlabs.online`
- `https://result.joao.and.irene.ironlabs.online`

---

## 📋 Overview

| Step | What it does |
|------|-------------|
| 1. OIDC Provider | Allows EKS pods to assume IAM roles without stored credentials |
| 2. IAM Roles | Grants cert-manager and external-dns permission to manage Route 53 |
| 3. NGINX Ingress | Installs the ingress controller with a public LoadBalancer |
| 4. cert-manager | Manages TLS certificates automatically |
| 5. ClusterIssuers | Configures Let's Encrypt (staging + production) |
| 6. Ingress TLS | Applies the Ingress with TLS — certs issued automatically |

---

## ✅ Prerequisites

- `kubectl` connected to your EKS cluster
- `helm` installed
- `eksctl` installed
- `aws` CLI configured with valid credentials
- Domain managed in Route 53 (`ironlabs.online`)

---

## 📦 Setup Scripts

All scripts are in the `Setup/` folder. Run them in order.

---

## Step 1 — OIDC Provider + IAM Roles

```bash
chmod +x Setup/install-oidc-iam.sh
bash Setup/install-oidc-iam.sh
```

This script:
- Registers the EKS OIDC provider
- Creates `CertManagerRole` with Route 53 permissions
- Creates `ExternalDNSRole` with Route 53 permissions

Verify the roles were created:
```bash
aws iam get-role --role-name CertManagerRole-eks-cluster-irene-and-joao
aws iam get-role --role-name ExternalDNSRole-eks-cluster-irene-and-joao
```

---

## Step 2 — NGINX Ingress Controller

```bash
chmod +x Setup/install-nginx-controller.sh
bash Setup/install-nginx-controller.sh
```

Verify it's running and get the LoadBalancer hostname:
```bash
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx
```

Copy the `EXTERNAL-IP` — you'll need it for Route 53 CNAME records.

### Route 53 DNS Records

In the AWS Console → Route 53 → `ironlabs.online` hosted zone, create two CNAME records:

| Name | Type | Value |
|------|------|-------|
| `vote.joao.and.irene.ironlabs.online` | CNAME | `<EXTERNAL-IP from above>` |
| `result.joao.and.irene.ironlabs.online` | CNAME | `<EXTERNAL-IP from above>` |

TTL: 300

Verify DNS propagation:
```bash
nslookup vote.joao.and.irene.ironlabs.online
```

---

## Step 3 — cert-manager

```bash
chmod +x Setup/install-cert-manager.sh
bash Setup/install-cert-manager.sh
```

Verify it's running:
```bash
kubectl get pods -n cert-manager
```

All 3 pods (`cert-manager`, `cert-manager-cainjector`, `cert-manager-webhook`) should be `Running`.

---

## Step 4 — ClusterIssuers

```bash
chmod +x Setup/install-clusterissuers.sh
bash Setup/install-clusterissuers.sh
```

This creates two ClusterIssuers:
- `letsencrypt-staging` — for testing (untrusted certs, no rate limits)
- `letsencrypt-prod` — for real browser-trusted certificates

Verify they are `Ready`:
```bash
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```

Both should show `Status: True` and `Ready`.

---

## Step 5 — Apply Ingress with TLS

```bash
kubectl apply -f k8s/ingress-tls/ingress-tls.yaml
```

Check the certificate is being issued:
```bash
kubectl get certificate
kubectl describe certificate joao-and-irene-tls-secret
```

It will show `Issuing` for a few minutes, then `Ready: True`. This means the certificate was issued successfully.

---

## Step 6 — Verify HTTPS

```bash
curl -v https://vote.joao.and.irene.ironlabs.online
curl -v https://result.joao.and.irene.ironlabs.online
```

Or open in the browser — you should see the 🔒 padlock.

---

## 🔄 Certificate Renewal

cert-manager renews certificates automatically before expiry (Let's Encrypt certs last 90 days, renewal happens at ~60 days). No manual action needed.

---

## 🐛 Troubleshooting

**ClusterIssuer not Ready:**
```bash
kubectl describe clusterissuer letsencrypt-prod
# Check cert-manager logs:
kubectl logs -n cert-manager -l app=cert-manager --tail=50
```

**Certificate stuck in Pending/Issuing:**
```bash
kubectl describe certificaterequest -n default
kubectl logs -n cert-manager -l app=cert-manager --tail=50
```

**Browser shows "Not Secure" after applying ingress-tls:**

The certificate may still be issuing. Wait 2-3 minutes and check:
```bash
kubectl get certificate -w
```

**DNS not resolving:**
```bash
nslookup vote.joao.and.irene.ironlabs.online
# Should return the EXTERNAL-IP of the NGINX LoadBalancer
```

---

## 📁 Files Reference

| File | Purpose |
|------|---------|
| `Setup/install-oidc-iam.sh` | OIDC provider + IAM roles for cert-manager and external-dns |
| `Setup/install-nginx-controller.sh` | NGINX Ingress Controller via Helm |
| `Setup/install-cert-manager.sh` | cert-manager via Helm with IRSA |
| `Setup/install-clusterissuers.sh` | Let's Encrypt staging + prod ClusterIssuers |
| `k8s/ingress-tls/ingress-tls.yaml` | Ingress with TLS annotation and cert-manager integration |