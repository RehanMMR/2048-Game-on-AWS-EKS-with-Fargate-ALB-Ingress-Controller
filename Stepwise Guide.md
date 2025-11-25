# ✅ 2048 Game on AWS EKS with Fargate & ALB Ingress Controller

## 🎮 Project: Deploying 2048 Game on Amazon EKS (Fargate) Using AWS Load Balancer Controller

This project demonstrates how to deploy a **2048 game application** on an **Amazon EKS cluster running on AWS Fargate**, and expose it to the internet using an **AWS Application Load Balancer (ALB) via the AWS Load Balancer Controller**.

---

<img width="450" height="450" alt="2048_LinkedIn" src="https://github.com/user-attachments/assets/8322a026-62c2-409c-a1f4-9574905700ea" />


## 🚀 Overview

This project includes:

* Creating an **EKS cluster** with **Fargate**
* Deploying the **2048 application**
* Setting up **AWS Load Balancer Controller (ALB Controller)** using **Helm**
* Automatically creating:
  * AWS ALB
  * Target groups
  * Listener rules
  * Pod registration
  * Public DNS endpoint

This setup makes your Fargate-hosted app publicly accessible.

---

## 🧰 1. Tools Used

| Tool                             | Purpose                                             |
| -------------------------------- | --------------------------------------------------- |
| **kubectl**                      | Interact with Kubernetes cluster                    |
| **aws cli**                      | Interact with AWS services                          |
| **eksctl**                       | Create and manage EKS cluster                       |
| **Helm**                         | Kubernetes package manager to deploy ALB controller |
| **AWS Load Balancer Controller** | Creates ALB based on Ingress rules                  |

---

## 🏗 2. Create EKS Cluster (With Fargate)

```sh
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

This creates:

* Control plane
* Fargate profiles for default namespaces
* Private and public subnets (Pods run in private)

---

## 🗂 3. Create Fargate Profile for Application

```sh
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name alb-sample-app \
    --namespace game-2048
```

All pods in namespace `game-2048` now run on Fargate.

---

## 🎮 4. Deploy the 2048 Game Application

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

This deploys:

* Deployment
* Service (ClusterIP)
* Ingress

Use these commands to check the pods:
```sh
kubectl get pods -n game-2048
```
Use these commands to check the Service:
```sh
kubectl get svc -n game-2048
```
Use these commands to check the Ingress:
```sh
kubectl get ingress -n game-2048
```
Note: No address will be visible

**BUT** Ingress will not work yet — ALB Controller is missing.

---

## ⚠️ Why?

Because:

### ✔ Ingress = Just rules

### ✔ ALB Controller = Creates the actual ALB in AWS

Without the controller → NO ALB is created.

---

## 🟦 5. How to setup alb add on

### 🔐 Associate IAM OIDC Provider

EKS needs this to let pods assume IAM roles.

```sh
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

### 5A. Download IAM Policy

```sh
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

Create the IAM policy:

```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

---

### 5B. Create IAM Role

```sh
eksctl create iamserviceaccount \
  --cluster=<your-cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```
* Provide your cluster-name and the account id.

---

## 📦 6. Deploy ALB controller
Add helm repo
```sh
helm repo add eks https://aws.github.io/eks-charts
```

Update the repo:
```sh
helm repo update eks
```
Install:
```sh
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=<your-cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<your-region> \
  --set vpcId=<your-vpc-id>
```
*Provide the vpc id which will be visible under the networking section in you cluster and also provide cluster name and region name by using AWS Console UI.

Verify that the deployments are running:
```sh
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 🔥 7. ALB Is Created Automatically

Once the controller sees your Ingress, it automatically creates:

✔ **AWS ALB**  
✔ **Target Groups**  
✔ **Listeners (HTTP/HTTPS)**  
✔ **Pod registration**  
✔ **Public DNS**

Get DNS:

```sh
kubectl get ingress -n game-2048
```

Open the URL → The 2048 game loads.

---

<img width="1692" height="1071" alt="Screenshot 2025-11-24 200250" src="https://github.com/user-attachments/assets/755b1946-af7b-42d1-afc4-ca9e0592f37f" />

## 💡 ALB vs ALB Controller (Simple Explanation)

### **ALB (AWS Service)**

A real load balancer in AWS that receives internet traffic.

### **ALB Controller (K8s pod)**

A controller that:

* Watches Ingress
* Calls AWS APIs
* Creates & configures the ALB

### **ALB Controller CREATES the ALB.**

You never create the ALB manually.

---

## 🎓 What is Helm? Why is it used?

### ✔ Helm = Kubernetes package manager

(Like apt, yum, pip, npm)

### ✔ Why required here?

AWS provides the ALB controller **only as a Helm chart**.

### ✔ What Helm does?

Deploys 20–25 YAMLs in one go:

* Deployment
* RBAC
* CRDs
* Controller logic

---

## 📘 Final Short Summary (Interview Answer)

> "I deployed a 2048 game on EKS using Fargate. I created the cluster, added a Fargate profile, and deployed the app. I then installed the AWS Load Balancer Controller using Helm, which watches Ingress resources and automatically creates an ALB, target groups, listeners, and registers pods. The ALB DNS exposes the game publicly."

---


## 🤝 Contributing

Contributions, issues, and feature requests are welcome! Feel free to fork the repository.

---

## ⭐ Show your support

Give a ⭐️ if this project helped you!
