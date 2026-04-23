# 🚀 Robot Shop Deployment on AWS EKS

This guide walks you through deploying the Robot Shop application on AWS EKS using Helm, ALB Ingress, and IRSA.

---

## 🖥️ Step 1: Create EC2 Instance

- Instance Type: `m7i-flex.large`
- Storage: `20 GB`

---

## 📦 Prerequisites

### 1. Install kubectl
https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html

### 2. Install eksctl
https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html

### 3. Configure AWS Credentials
```bash
aws configure
```

Provide:
- AWS Access Key
- AWS Secret Access Key
- Region (e.g., us-east-1)
- Output format: json

---

## ☸️ Step 2: Create EKS Cluster

```bash
eksctl create cluster \
  --name robotshop-cluster \
  --region us-east-1 \
  --nodegroup-name robotshop-nodegroup \
  --node-type m7i-flex.large \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3
```

---

## 🔐 Step 3: Configure IAM OIDC Provider

### Why?
Allows Kubernetes pods to securely access AWS services using IRSA.

### Commands
```bash
export cluster_name=robotshop-cluster

oidc_id=$(aws eks describe-cluster \
  --name $cluster_name \
  --query "cluster.identity.oidc.issuer" \
  --output text | cut -d '/' -f 5)
```

```bash
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

```bash
export AWS_DEFAULT_REGION=us-east-1

eksctl utils associate-iam-oidc-provider \
  --cluster $cluster_name \
  --approve
```

---

## 🌐 Step 4: Setup ALB Controller IAM

### Download IAM Policy
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
```

### Create Policy
```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### Create IAM Role & Service Account
```bash
eksctl create iamserviceaccount \
  --cluster robotshop-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

## ⚙️ Step 5: Install AWS Load Balancer Controller

### Install Helm
```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Add Repo
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

### Install Controller
```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=robotshop-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<YOUR-VPC-ID>
```

### Verify
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 💾 Step 6: Configure EBS CSI Driver
👉 Amazon EBS CSI Driver
It allows Kubernetes to:
Automatically create and attach EBS volumes to pods

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster robotshop-cluster \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster robotshop-cluster \
  --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

---

## 📦 Step 7: Deploy Robot Shop Application

### Clone Repo
```bash
git clone https://github.com/uniquesreedhar/RobotShop-Project.git
cd RobotShop-Project/EKS/helm
```

### Deploy
```bash
kubectl create ns robot-shop
helm install robot-shop --namespace robot-shop .
```

### Verify
```bash
kubectl get pods -n robot-shop
```

---

## 🌍 Step 8: Create Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: robot-shop
  name: robot-shop
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
```

### Apply
```bash
kubectl apply -f ingress.yaml
```

---

## 🌐 Access Application

```bash
kubectl get ingress -n robot-shop
```

Copy the **ADDRESS (ALB DNS)** and open in browser:

```
http://<ALB-DNS>
```

---

## 🎯 Summary

- Created EKS Cluster
- Configured IRSA (OIDC)
- Installed ALB Controller
- Configured EBS CSI Driver
- Deployed Robot Shop using Helm
- Exposed app using Ingress (ALB)

---

## 💡 Notes

- Ensure subnets are properly tagged
- Ensure IAM policies are correctly attached
- ALB creation may take 1–3 minutes

---

## 🧹 Cleanup

```bash
eksctl delete cluster --name robotshop-cluster --region us-east-1
```

---

🎉 Happy Learning DevOps!

