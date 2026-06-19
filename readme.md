# EKS Application Deployment via GitOps

## Project Overview

This project demonstrates a complete GitOps-based deployment pipeline for a microservice called **payment-service** on **Amazon EKS** using modern DevOps practices.

The goal was not only to deploy an application, but to build a production-style workflow including:

- GitHub Actions (CI)
- Amazon ECR
- Amazon EKS
- Helm
- ArgoCD (GitOps)
- AWS Secrets Manager
- IRSA (IAM Roles for Service Accounts)
- AWS Load Balancer Controller
- ALB Ingress
- Prometheus
- Grafana

---

# Problem Statement

A new microservice called `payment-service` must be deployed to EKS.

Required Architecture:

GitHub
↓
GitHub Actions
↓
Amazon ECR
↓
ArgoCD
↓
Amazon EKS

Requirements:

- Deploy using Helm Chart
- Use ArgoCD Auto Sync
- Store secrets in AWS Secrets Manager
- Use IRSA
- Expose through ALB Ingress
- Metrics visible in Grafana

---

# Why This Project?

Traditional Kubernetes deployments often involve:

- Manual deployments
- Manual image updates
- Hardcoded secrets
- Poor observability
- Configuration drift

This project solves those problems by implementing:

### GitOps

Everything is stored in Git and automatically synchronized using ArgoCD.

### Automated CI

GitHub Actions automatically builds and pushes Docker images to ECR.

### Secure Secret Management

Secrets are stored in AWS Secrets Manager instead of Kubernetes manifests.

### Fine-Grained IAM

IRSA allows pods to securely access AWS services without storing AWS credentials.

### Scalable Ingress

Application traffic is routed through an AWS Application Load Balancer.

### Monitoring

Prometheus collects metrics and Grafana visualizes cluster health.

---

# Final Architecture

```text
GitHub
   ↓
GitHub Actions
   ↓
Amazon ECR
   ↓
ArgoCD
   ↓
Amazon EKS
   ↓
Helm Chart
   ↓
Payment Service
   ↓
ALB Ingress

AWS Secrets Manager
          ↑
        IRSA

Prometheus
     ↓
Grafana
```

---

# Technologies Used

- Amazon EKS
- Amazon ECR
- IAM
- OIDC
- IRSA
- AWS Secrets Manager
- AWS Load Balancer Controller
- Docker
- Kubernetes
- Helm
- ArgoCD
- GitHub Actions
- Prometheus
- Grafana
- Flask

---

# Repository Structure

```text
payment-service-github/
│
├── app/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── payment-service/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│
└── .github/
    └── workflows/
        └── ecr.yml
```

---

# Step 1 – Create EKS Cluster

```bash
eksctl create cluster \
  --name payment-cluster \
  --region ap-south-1 \
  --nodes 2 \
  --node-type t3.medium
```

Verify:

```bash
kubectl get nodes
```

---

# Step 2 – Build Application

Example Flask App:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Payment Service Running"

app.run(host="0.0.0.0", port=8080)
```

---

# Step 3 – Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name payment-service \
  --region ap-south-1
```

Login:

```bash
aws ecr get-login-password \
| docker login \
--username AWS \
--password-stdin <account>.dkr.ecr.ap-south-1.amazonaws.com
```

Build:

```bash
docker build -t payment-service .
```

Tag:

```bash
docker tag payment-service:latest \
<account>.dkr.ecr.ap-south-1.amazonaws.com/payment-service:v1
```

Push:

```bash
docker push \
<account>.dkr.ecr.ap-south-1.amazonaws.com/payment-service:v1
```

---

# Step 4 – Create Helm Chart

```bash
helm create payment-service
```

Customized:

- Deployment
- Service
- Image Repository
- Replica Count
- Service Type
- Container Port

Render:

```bash
helm template payment-service .
```

Validate:

```bash
helm lint .
```

Deploy:

```bash
helm install payment-service .
```

---

# Step 5 – Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd

helm install argocd argo/argo-cd \
-n argocd
```

Get Password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d
```

---

# Step 6 – Configure GitOps

Push Helm Chart to GitHub.

Create ArgoCD Application:

- Repository URL
- Path: payment-service
- Sync Policy: Automatic
- Self Heal: Enabled
- Prune: Enabled

Verification:

```bash
kubectl get applications -n argocd
```

---

# Step 7 – GitHub Actions

Workflow:

```text
Git Push
   ↓
GitHub Actions
   ↓
Docker Build
   ↓
Push Image To ECR
```

Required Secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

Workflow File:

```text
.github/workflows/ecr.yml
```

---

# Step 8 – Configure OIDC

Verify:

```bash
aws eks describe-cluster \
--name payment-cluster \
--region ap-south-1 \
--query "cluster.identity.oidc.issuer"
```

Associate:

```bash
eksctl utils associate-iam-oidc-provider \
--cluster payment-cluster \
--region ap-south-1 \
--approve
```

---

# Step 9 – AWS Secrets Manager

Create Secret:

```bash
aws secretsmanager create-secret \
--name payment-service-secret \
--secret-string '{"db_password":"supersecret123"}'
```

Verify:

```bash
aws secretsmanager get-secret-value \
--secret-id payment-service-secret
```

---

# Step 10 – IRSA

Create IAM Policy:

```bash
aws iam create-policy \
--policy-name PaymentServiceSecretsPolicy \
--policy-document file://secrets-policy.json
```

Create Service Account:

```bash
eksctl create iamserviceaccount \
--name payment-service \
--namespace default \
--cluster payment-cluster \
--attach-policy-arn <policy-arn> \
--approve
```

Verify:

```bash
kubectl get sa payment-service -o yaml
```

Expected:

```yaml
eks.amazonaws.com/role-arn: <role-arn>
```

---

# Step 11 – Install AWS Load Balancer Controller

Create IAM Policy:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

Install Controller:

```bash
helm install aws-load-balancer-controller \
eks/aws-load-balancer-controller \
-n kube-system
```

Verify:

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

# Step 12 – ALB Ingress

Service Type:

```yaml
service:
  type: ClusterIP
```

Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-service-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
```

Verify:

```bash
kubectl get ingress
```

Expected:

```text
k8s-default-payments-xxxxx.ap-south-1.elb.amazonaws.com
```

---

# Step 13 – Monitoring

Create Namespace:

```bash
kubectl create namespace monitoring
```

Install:

```bash
helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts

helm repo update

helm install monitoring \
prometheus-community/kube-prometheus-stack \
-n monitoring
```

Grafana Password:

```bash
kubectl get secret monitoring-grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" \
| base64 -d
```

Port Forward:

```bash
kubectl port-forward svc/monitoring-grafana \
-n monitoring 3000:80
```

Access:

```text
http://localhost:3000
```

---

# Verification Checklist

- [x] Deploy using Helm Chart
- [x] Use ArgoCD Auto-Sync
- [x] Store secrets in AWS Secrets Manager
- [x] Use IRSA
- [x] Expose through ALB Ingress
- [x] Metrics visible in Grafana

---

# Screenshots to Include

1. EKS Cluster
2. ECR Repository
3. GitHub Actions Success
4. ArgoCD Healthy + Synced
5. ServiceAccount IRSA Annotation
6. Secrets Manager Secret
7. ALB Ingress
8. Payment Service Running
9. Grafana Dashboard

---

# Key Learnings

- Kubernetes Fundamentals
- Helm Package Management
- GitOps with ArgoCD
- CI/CD using GitHub Actions
- Secure AWS Authentication using IRSA
- Secret Management with AWS Secrets Manager
- ALB-based Application Exposure
- Monitoring using Prometheus and Grafana

---

# Author

Dharani Kumar

Project: EKS Application Deployment via GitOps

End-to-End Cloud Native DevOps Implementation on AWS.
