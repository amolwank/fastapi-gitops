Importance Notice:


This needs to run every 12 hours as 
kubectl create secret docker-registry ecr-secret \
  --docker-server=774305591014.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  --namespace=fastapi-namespace \
  --dry-run=client -o yaml | kubectl apply -f -

🚀 FastAPI on AWS EKS with GitOps, ALB & Progressive Delivery

This project deploys a FastAPI application on AWS EKS using:

Terraform for infrastructure

Private VPC with VPC Endpoints

ECR for container registry

GitHub Actions for CI

ArgoCD + Helm for GitOps

AWS ALB for traffic

Argo Rollouts (Blue/Green & Canary)

🧱 Architecture
Internet
   ↓
AWS ALB
   ↓
Kubernetes Ingress
   ↓
Argo Rollouts
   ↓
FastAPI Pods (Private Subnets)
   ↓
ECR via VPC Endpoints
   ↓
S3 via Gateway Endpoint

🛠 1. Infrastructure Setup (Terraform)
Folder structure
platform/
├── vpc/
└── eks/

1️⃣ Deploy VPC
cd platform/vpc
terraform init
terraform apply


This creates:

VPC

Public & Private subnets

NAT Gateway

ECR & S3 VPC Endpoints

2️⃣ Deploy EKS
cd ../eks
terraform init
terraform apply


This creates:

EKS cluster

Private node group

IAM roles

3️⃣ Configure kubectl
aws eks update-kubeconfig \
  --region us-east-1 \
  --name my-eks-cluster

kubectl get nodes

📦 2. Install AWS Load Balancer Controller
eksctl utils associate-iam-oidc-provider \
  --cluster my-eks-cluster \
  --region us-east-1 \
  --approve


Create IAM policy:

curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json


Create service account:

eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn <POLICY_ARN> \
  --approve


Install controller:

helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

🚀 3. Install ArgoCD
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


Access UI:

kubectl port-forward svc/argocd-server -n argocd 8080:443


Get password:

kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath="{.data.password}" | base64 -d

📦 4. GitOps Deployment

ArgoCD watches the GitOps repo which contains:

Helm charts

Rollout (Blue/Green or Canary)

Services

Ingress

Create application:

argocd app create fastapi \
  --repo https://github.com/<your-user>/fastapi-gitops.git \
  --path charts/fastapi \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace fastapi-namespace \
  --sync-policy automated

🌍 5. AWS ALB Ingress

Ingress must point to stable service:

backend:
  service:
    name: fastapi-app-stable
    port:
      number: 8000


Get ALB URL:

kubectl get ingress -n fastapi-namespace

🔁 6. Progressive Delivery (Argo Rollouts)
Canary

Traffic shifts gradually:

20% → 50% → 100%

Blue/Green
Users → Stable
Test → Preview
Promote → Switch traffic


Check rollout:

kubectl argo rollouts get rollout fastapi-app -n fastapi-namespace


Promote Blue/Green:

kubectl argo rollouts promote fastapi-app -n fastapi-namespace


Rollback:

kubectl argo rollouts undo fastapi-app -n fastapi-namespace

🧠 Why ArgoCD shows "Health: Suspended"

Argo Rollouts manages pod health instead of ArgoCD.
This is normal for progressive delivery.

🏆 What this platform provides

Zero-downtime deployments

Private Kubernetes nodes

Secure ECR image pulls

GitOps-driven releases

Canary & Blue/Green deployments

AWS-native ALB routing  