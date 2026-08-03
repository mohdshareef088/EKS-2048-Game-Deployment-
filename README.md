

# 📘 **Amazon EKS 2048 Game Deployment using AWS Load Balancer Controller**

This project demonstrates how to deploy a containerized web application (the classic **2048 game**) on **Amazon EKS**, expose it using **Kubernetes Ingress**, and automatically provision an **AWS Application Load Balancer (ALB)** using the **AWS Load Balancer Controller**.
---

## 🏗️ **Architecture Overview**

**Components:**

- **Amazon EKS Cluster**  
- **AWS Load Balancer Controller** (Helm)  
- **IAM Role + Policy** for ALB controller  
- **Fargate Profile** 
- **2048 Deployment** (5 replicas)  
- **Kubernetes Service** (NodePort)  
- **Ingress** with `alb` ingress class  
- **AWS Application Load Balancer** (internet-facing)


## 🛠️ **Prerequisites**

Before deploying, ensure you have:

- AWS CLI
sudo curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure

- kubectl
From <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/> 
sudo rm -f /usr/local/bin/kubectl
curl -LO "https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
- eksctl
PLATFORM=$(uname -s)_amd64
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt"
grep ${PLATFORM}.tar.gz eksctl_checksums.txt | sha256sum --check
tar -xzf eksctl_${PLATFORM}.tar.gz
sudo mv eksctl /usr/local/bin/
eksctl version

- IAM role + policy  
- Helm
- Application Load Balancer    

---

## 🚀 **1. Deploy the 2048 Game**

- Active EKS cluster
  eksctl create cluster --name demo-testing-cluster --region ap-south-1 --fargate
- fargate profile & namespace
eksctl create fargateprofile \
    --cluster demo-testing-cluster \
    --region ap-south-1 \
    --name alb-sample-app \
    --namespace game-2048

- manifest
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
- config file
  
aws eks update-kubeconfig --name demo-testing-cluster --region ap-south-1 

## 🔍 **2. Verify Deployment**

### Pods
<img width="1338" height="586" alt="pods" src="https://github.com/user-attachments/assets/4b8264a8-a707-4c8d-8b48-18d507c12f08" />

```bash
kubectl get pods -n game-2048
```
### Service

```bash
kubectl get svc -n game-2048
```
### Ingress

```bash
kubectl get ingress -n game-2048
```

Expected:

```
NAME           CLASS   HOSTS   ADDRESS   PORTS   AGE
ingress-2048   alb     *                 80      2m
```
<img width="542" height="47" alt="ingress" src="https://github.com/user-attachments/assets/2da72f17-69b3-4416-8d7c-9933fbe876a2" />

---
## 🔧 **4. AWS Load Balancer Controller Requirements**

### ✔ IAM Policy

- Attach the official AWS policy:

IAM OIDC provider
eksctl utils associate-iam-oidc-provider --cluster demo-testing-cluster --approve


- Download IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

- Create IAM Policy - to access the alb-controller thru pods
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

- Create IAM Role
eksctl create iamserviceaccount   --cluster=demo-testing-cluster   --namespace=kube-system   --name=aws-load-balancer-controller   --role-name AmazonEKSLoadBalancerControllerRole   --attach-policy-arn=arn:aws:iam::140447104913:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve

<img width="1244" height="735" alt="sa" src="https://github.com/user-attachments/assets/100d2237-902b-4b38-87fc-b42873e710f1" />

- Deploy ALB controller
Install Helm to create ALB Controller
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh

- Add helm repo
helm repo add eks https://aws.github.io/eks-charts
Update the repo
helm repo update eks

- Helm Install
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-testing-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region= ap-south-1 \
  --set vpcId=vpc-012779105586dbdd3
<img width="389" height="241" alt="namespaces" src="https://github.com/user-attachments/assets/2516b5af-6565-47d3-95ee-3f79a2fa1c2d" />


### ✔ Subnet Tags

Public subnets:

```
kubernetes.io/role/elb = 1
kubernetes.io/cluster/<cluster-name> = shared
```

Private subnets:

```
kubernetes.io/role/internal-elb = 1
kubernetes.io/cluster/<cluster-name> = shared
```

### ✔ Controller Health

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```
READY   UP-TO-DATE   AVAILABLE
2/2     2            2
```

## 🌐 **3. Access the Application**

Once the ALB is provisioned Active, the Ingress will show:
<img width="994" height="330" alt="alb" src="https://github.com/user-attachments/assets/c7eff8f5-77ac-498b-992e-46ab7ffa3692" />

Open it:

```
http://k8s-ingress-xxxx.ap-south-1.elb.amazonaws.com
```

You should see the 2048 game UI.
<img width="1179" height="955" alt="2048" src="https://github.com/user-attachments/assets/a7ae1c01-811b-4f9b-a8f1-5e54f7542e8c" />

---

## 🧹 **Cleanup**

Delete the EKS cluster:

```bash
eksctl delete cluster --name demo-testing-cluster --region ap-south-1
```

