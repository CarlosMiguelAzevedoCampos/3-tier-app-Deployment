# 🚀 Deploy a Three-Tier Application on AWS EKS using Terraform

Welcome to the official repository for deploying a **Three-Tier Todo List Application** on **AWS EKS** using **Terraform** and **Kubernetes**! This project demonstrates how to build, containerize, and deploy a production-style architecture in the cloud using best DevOps practices.

---

## 📁 Project Structure

```bash
3-tier-app-Deployment/
├── backend/              # Node.js backend code
├── frontend/             # React frontend code
├── mongo/                # MongoDB Kubernetes manifests
├── k8s_manifests/        # K8s manifests for frontend, backend, ingress
└── terra-config/         # Terraform files to provision AWS infrastructure
````

---

## ⚙️ Technologies Used

* **Terraform** (Infrastructure as Code)
* **Amazon EKS** (Kubernetes Cluster)
* **Amazon ECR** (Docker image registry)
* **Amazon S3** (Terraform remote state storage)
* **Kubernetes** (App deployment & orchestration)
* **Helm** (Load Balancer controller installation)
* **React + Node.js + MongoDB** (Application stack)

---

## 📦 Prerequisites

Make sure you have the following installed and configured:

* [ ] AWS Account + IAM user with AdministratorAccess
* [ ] AWS CLI
* [ ] Docker
* [ ] Terraform
* [ ] `kubectl`
* [ ] `eksctl`
* [ ] `helm`

---

## 🔧 Setup Instructions

### 1. Clone the Repo

```bash
git clone https://github.com/Pravesh-Sudha/3-tier-app-Deployment.git
cd 3-tier-app-Deployment/
```

### 2. Set Up AWS CLI

Create an IAM user → Generate access key → Run:

```bash
aws configure
```

Use region `us-east-1` and output `json`.

---

### 3. Create an S3 Bucket for Terraform State

```bash
aws s3api create-bucket \
  --bucket pravesh-terra-state-bucket \
  --region us-east-1 \
  --create-bucket-configuration LocationConstraint=us-east-1

aws s3api put-bucket-versioning \
  --bucket pravesh-terra-state-bucket \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket pravesh-terra-state-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

---

### 4. Provision AWS Infrastructure with Terraform

```bash
cd terra-config/
terraform init
terraform apply --auto-approve
```

---

### 5. Push Docker Images to ECR

Follow the "View push commands" from the ECR console for both:

* `three-tier-frontend`
* `three-tier-backend`

Update image URIs in:

* `k8s_manifests/frontend_deployment.yml`
* `k8s_manifests/backend_deployment.yml`

---

### 6. Configure and Deploy to EKS

```bash
aws eks update-kubeconfig --region us-east-1 --name Three-tier-cloud
kubectl create namespace workshop
kubectl config set-context --current --namespace workshop

# Apply app deployments
kubectl apply -f k8s_manifests/frontend-deployment.yaml -f k8s_manifests/frontend-service.yaml
kubectl apply -f k8s_manifests/backend-deployment.yaml -f k8s_manifests/backend-service.yaml
kubectl apply -f mongo/
```

---

### 7. Set Up Application Load Balancer (ALB) & Ingress

#### a. IAM Policy and OIDC

```bash
cd k8s_manifests/
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
```

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=Three-tier-cloud --approve
```

```bash
eksctl create iamserviceaccount \
  --cluster=Three-tier-cloud \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<YOUR-AWS-ACCOUNT-ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region=us-east-1
```

#### b. Install Helm and Load Balancer Controller

```bash
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=Three-tier-cloud \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

### 8. Apply Ingress and Access App

```bash
kubectl apply -f k8s_manifests/full_stack_lb.yaml
kubectl get ing -n workshop
```

🎉 Visit the **ADDRESS** shown in the output to access your live application in the browser!

---

## 🧹 Cleanup Instructions

After testing, you can tear everything down to avoid charges:

```bash
# Delete ECR images manually from AWS Console
terraform destroy --auto-approve
aws s3 rm s3://pravesh-terra-state-bucket/eks/terraform.tfstate
# Then empty and delete the bucket via S3 console
```

---

## ✨ Author

**Pravesh Sudha**
AWS Community Builder – Containers | DevOps & Cloud Blogger

* 💻 [Website](https://praveshsudha.com)
* 🧠 [Blog](https://blog.praveshsudha.com)
* 🐦 [Twitter](https://x.com/praveshstwt)
* 💼 [LinkedIn](https://www.linkedin.com/in/pravesh-sudha/)

---

## ⭐️ Support & Contribution

If you find this project helpful, please consider giving it a ⭐ on GitHub!

Pull requests and suggestions are welcome 🤝

---



## This is needed for the aws loadbalancer to work

# --- Vars ---

# Make sure kubectl points to the right cluster
aws eks update-kubeconfig --name $CLUSTER --region $REGION
kubectl config current-context

# Ensure OIDC provider exists for this cluster (idempotent)
eksctl utils associate-iam-oidc-provider \
  --cluster $CLUSTER \
  --region  $REGION \
  --approve

# Get OIDC issuer -> build provider ARN
OIDC_ISSUER=$(aws eks describe-cluster --name $CLUSTER --region $REGION --query "cluster.identity.oidc.issuer" --output text)
OIDC_HOSTPATH=${OIDC_ISSUER#https://}
OIDC_PROVIDER_ARN=arn:aws:iam::$ACCOUNT_ID:oidc-provider/$OIDC_HOSTPATH
echo "OIDC_PROVIDER_ARN = $OIDC_PROVIDER_ARN"

# Create IAM trust policy for the K8s SA kube-system/aws-load-balancer-controller
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "$OIDC_PROVIDER_ARN" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$OIDC_HOSTPATH:aud": "sts.amazonaws.com",
          "$OIDC_HOSTPATH:sub": "system:serviceaccount:kube-system:aws-load-balancer-controller"
        }
      }
    }
  ]
}
EOF

# Create the IAM role & attach the LB controller policy
aws iam create-role --role-name $ROLE_NAME --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name $ROLE_NAME --policy-arn $POLICY_ARN

# Get the Role ARN
ROLE_ARN=$(aws iam get-role --role-name $ROLE_NAME --query Role.Arn --output text)
echo "ROLE_ARN = $ROLE_ARN"

# Get the VPC ID from its Name tag
VPC_ID=$(aws ec2 describe-vpcs --region $REGION \
  --filters "Name=tag:Name,Values=${VPC_NAME}" \
  --query "Vpcs[0].VpcId" --output text)
echo "VPC_ID = $VPC_ID"

# (Re)install the AWS Load Balancer Controller
# Let Helm CREATE the ServiceAccount and annotate it with your Role ARN
helm -n kube-system upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=$CLUSTER \
  --set region=$REGION \
  --set vpcId=$VPC_ID \
  --set installCRDs=true \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="${ROLE_ARN}"

# Verify
kubectl -n kube-system get sa aws-load-balancer-controller -o yaml | sed -n '/annotations:/,/labels:/p'
kubectl -n kube-system get deploy aws-load-balancer-controller
kubectl -n kube-system logs deploy/aws-load-balancer-controller
