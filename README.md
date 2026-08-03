

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

  
- AWS Load Balancer Controller installed  
- IAM role + policy  
- Public/private subnets tagged correctly
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

---

## 🌐 **3. Access the Application**

Once the ALB is provisioned, the Ingress will show:

```
ADDRESS   k8s-ingress-xxxx.ap-south-1.elb.amazonaws.com
```

Open it:

```
http://k8s-ingress-xxxx.ap-south-1.elb.amazonaws.com
```

You should see the 2048 game UI.

---

## 🔧 **4. AWS Load Balancer Controller Requirements**

### ✔ IAM Policy

Attach the official AWS policy:

```bash
aws iam attach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy
```

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

Expected:

```
READY   UP-TO-DATE   AVAILABLE
2/2     2            2
```

---

## 🧩 **5. Common Interview Talking Points**

Use these during interviews:

### **Why ALB instead of NLB?**
ALB supports HTTP/HTTPS, path‑based routing, host‑based routing, and is ideal for web apps.

### **Why do we need AWS Load Balancer Controller?**
Because Kubernetes Ingress alone cannot create AWS ALBs.  
The controller translates Ingress → AWS API calls.

### **Why does Fargate show multiple nodes?**
Each Fargate pod runs in its own isolated environment, represented as a virtual node.

### **Why does Ingress ADDRESS stay empty?**
Missing IAM permissions, wrong service account, or untagged subnets.

### **What did you learn?**
IAM roles, OIDC, Helm charts, Kubernetes networking, ALB provisioning, troubleshooting.

---

## 🛑 **Troubleshooting Guide**

### ❌ Ingress ADDRESS is empty  
Check events:

```bash
kubectl describe ingress ingress-2048 -n game-2048
```

### ❌ ALB not created  
Check controller logs:

```bash
kubectl logs -n kube-system deployment/aws-load-balancer-controller
```

### ❌ Controller pods stuck at 0/2  
Fix service account + IAM role mismatch.

### ❌ Webhook errors  
Ensure the controller is installed correctly via Helm.

---

## 🧹 **Cleanup**

Delete the app:

```bash
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

Delete the ALB:

```bash
aws elbv2 delete-load-balancer --load-balancer-arn <arn>
```

Delete the EKS cluster:

```bash
eksctl delete cluster --name <cluster-name> --region <region>
```

---

## 📚 **References**

- AWS Load Balancer Controller  
- EKS Documentation  
- Kubernetes Ingress  
- 2048 Demo App  

---

Mohammed, if you want, I can also:

- Add a **diagram section**  
- Add **badges** (AWS, Kubernetes, Helm, Terraform)  
- Add a **project summary for your CV**  
- Add a **step-by-step installation guide**  
- Add a **professional architecture diagram**  

Just tell me and I’ll upgrade this README even further.
