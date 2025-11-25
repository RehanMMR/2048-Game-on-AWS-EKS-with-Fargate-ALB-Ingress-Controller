# 2048-Game-on-AWS-EKS-with-Fargate-ALB-Ingress-Controller
🎮 Project: Deploying 2048 Game on Amazon EKS (Fargate) Using AWS Load Balancer Controller

This project demonstrates how to deploy a 2048 game application on an Amazon EKS cluster running on AWS Fargate, and expose it to the internet using an AWS Application Load Balancer (ALB) via the AWS Load Balancer Controller.
📌 Architecture Diagram

🚀 Overview
This project includes:

Creating an EKS cluster with Fargate

Deploying the 2048 application

Setting up AWS Load Balancer Controller (ALB Controller) using Helm

Automatically creating:

AWS ALB

Target groups

Listener rules

Pod registration

Public DNS endpoint

This setup makes your Fargate-hosted app publicly accessible.

🧰 1. Tools Used
kubectl- Interact with Kubernetes cluster
aws cli- Interact with AWS services
eksctl- Create and manage EKS cluster
Helm- Kubernetes package manager to deploy ALB controller
AWS Load Balancer Controller- Creates ALB based on Ingress rules

🏗 2. Create EKS Cluster (With Fargate)
This creates:
Control plane
Fargate profiles for default namespaces

Private and public subnets
(Pods run in private)

🗂 3. Create Fargate Profile for Application
All pods in namespace game-2048 now run on Fargate.

🎮 4. Deploy the 2048 Game Application
This deploys:
Deployment
Service (ClusterIP)
Ingress

BUT Ingress will not work yet — ALB Controller is missing.

⚠️ Why?
Because:

✔ Ingress = Just rules
✔ ALB Controller = Creates the actual ALB in AWS
Without the controller → NO ALB is created.

🟦 5. Install AWS Load Balancer Controller (MOST IMPORTANT)
Before that use:
🔐Associate IAM OIDC Provider

EKS needs this to let pods assume IAM roles.
5A. Download IAM Policy
sh
Copy code
curl -O https://raw.githubusercontent.com/.../iam_policy.json
Create the IAM policy:

sh
Copy code
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
5B. Create IAM Service Account

eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
📦 6. Install ALB Controller using Helm

helm repo add eks https://aws.github.io/eks-charts

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
Verify installation:


kubectl get deployment -n kube-system aws-load-balancer-controller
🔥 8. ALB Is Created Automatically
Once the controller sees your Ingress, it automatically creates:

✔ AWS ALB
✔ Target Groups
✔ Listeners (HTTP/HTTPS)
✔ Pod registration
✔ Public DNS

Get DNS:

kubectl get ingress -n game-2048
Open the URL → The 2048 game loads.

💡 ALB vs ALB Controller (Simple Explanation)
ALB (AWS Service)
A real load balancer in AWS that receives internet traffic.

ALB Controller (K8s pod)
A controller that:

Watches Ingress

Calls AWS APIs

Creates & configures the ALB

ALB Controller CREATES the ALB.
You never create the ALB manually.

🎓 What is Helm? Why is it used?
✔ Helm = Kubernetes package manager
(Like apt, yum, pip, npm)

✔ Why required here?
AWS provides the ALB controller only as a Helm chart.

✔ What Helm does?
Deploys 20–25 YAMLs in one go:

Deployment

RBAC

CRDs

Controller logic

📘 Final Short Summary (Interview Answer)
“I deployed a 2048 game on EKS using Fargate.
I created the cluster, added a Fargate profile, and deployed the app.
I then installed the AWS Load Balancer Controller using Helm, which watches Ingress resources and automatically creates an ALB, target groups, listeners, and registers pods.
The ALB DNS exposes the game publicly.”
