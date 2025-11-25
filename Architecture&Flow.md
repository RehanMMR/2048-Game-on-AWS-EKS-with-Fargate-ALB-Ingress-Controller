## 📌 Architecture Diagram

<img width="1630" height="1047" alt="Screenshot 2025-11-25 164621" src="https://github.com/user-attachments/assets/4f06d77c-ee1b-4e2c-9c0c-07b9b6befd21" />


## COMPLETE ARCHITECTURE BREAKDOWN

Let me walk you through EXACTLY what happens, step-by-step, matching your actual commands.

## 🔍 PART 1: The Components (What Each Box Means)

1. User/Internet (Blue Circle - Top Left)

What it is: Anyone browsing the internet
What they do: Type the ALB DNS URL in their browser
Real example: http://k8s-game2048-ingress2-abc123.us-east-1.elb.amazonaws.com

2. VPC (Virtual Private Cloud)

What it is: Your isolated network in AWS
Created by: eksctl create cluster automatically creates this
Contains: All your AWS resources (subnets, EKS, etc.)

3. Public Subnet (Green Box)

What it is: Network accessible from the internet
Contains: The Application Load Balancer (ALB)
Why public? So users can reach your app from anywhere

4. Application Load Balancer - ALB (Orange Box)

What it is: AWS service that receives internet traffic
Created by: ALB Controller (NOT manually by you!)
Job:

Accept HTTP requests from users
Distribute traffic across multiple pods
Health check pods
Provide a single DNS endpoint



5. Private Subnet (Orange/Brown Box)

What it is: Network NOT accessible from internet (secure)
Contains: Your EKS pods running on Fargate
Why private? Security best practice - apps don't need direct internet access

6. EKS Control Plane (Blue Box - "EKS Control Plane")

What it is: The brain of Kubernetes (managed by AWS)
Created by: eksctl create cluster
Contains:

API Server (receives kubectl commands)
etcd (database storing cluster state)
Scheduler (decides which pod goes where)


YOU DON'T MANAGE THIS - AWS handles it

7. ALB Controller Pod (Dark Blue - "Helm AWS LB Controller")

What it is: A Kubernetes pod running in kube-system namespace
Installed by: helm install aws-load-balancer-controller
Job:

Watch for Ingress resources
Call AWS APIs to create ALB
Create Target Groups
Register pods with ALB
Update ALB when pods change



8. Fargate Profile (Orange Dashed Box)

What it is: Configuration telling EKS to run pods on Fargate
Created by: eksctl create fargateprofile --namespace game-2048
Means: All pods in "game-2048" namespace run serverless (no EC2)

9. Pods 1, 2, 3 (Green Boxes - "2048 Container")

What it is: Your actual 2048 game application running in containers
Created by: kubectl apply -f 2048_full.yaml
Runs on: AWS Fargate (serverless - no servers to manage)
Contains:

Docker container with 2048 game
Listens on port 80
Each pod = separate Fargate task



10. Service (Cyan Box - "ClusterIP")

What it is: Kubernetes load balancer INSIDE the cluster
Created by: The 2048_full.yaml manifest
Job:

Provides a stable internal IP
Load balances traffic across all 3 pods
If a pod dies, routes traffic to healthy pods



11. Ingress (Purple Box)

What it is: Kubernetes resource with routing rules
Created by: The 2048_full.yaml manifest
Contains rules like:

yaml  path: /
  backend:
    serviceName: game-2048
    servicePort: 80

- **Does NOT create ALB itself** - just defines the rules!

---

## 🚦 **PART 2: The Traffic Flow (What Actually Happens)**

### **FLOW 1: User Request → Your App** (Left to Right)
```
👤 User Types URL
    ↓
🌐 DNS resolves to ALB IP
    ↓
📡 ALB (Public Subnet) receives HTTP request
    ↓
🎯 ALB checks Target Groups (which pods are healthy?)
    ↓
📦 ALB forwards request to one of the 3 pods (Private Subnet)
    ↓
🎮 Pod processes request and returns 2048 game HTML
    ↓
📡 Response goes back through ALB
    ↓
👤 User sees the game!
```

**Real Example:**
1. User types: `k8s-game2048-abc.elb.amazonaws.com`
2. ALB receives it at port 80
3. ALB sends to Pod 2 (because it's healthy and least loaded)
4. Pod 2 serves the game
5. User plays 2048!

---

### **FLOW 2: How ALB Gets Created** (The Magic Part!)
```
1️⃣ You run: kubectl apply -f 2048_full.yaml
    ↓
2️⃣ Kubernetes creates:
   - Deployment (3 pods)
   - Service (internal load balancer)
   - Ingress (routing rules)
    ↓
3️⃣ ALB Controller (running in kube-system) WATCHES Ingress
    ↓
4️⃣ Controller says: "Hey! New Ingress detected!"
    ↓
5️⃣ Controller calls AWS APIs:
   - Create Application Load Balancer
   - Create Target Group
   - Create Listeners (HTTP on port 80)
   - Register pods as targets
    ↓
6️⃣ ALB appears in AWS Console (you didn't create it manually!)
    ↓
7️⃣ ALB gets a DNS name
    ↓
8️⃣ Ingress shows the DNS: kubectl get ingress -n game-2048
```

**This is shown by the DASHED ORANGE LINE from Controller to ALB**

---

### **FLOW 3: How Pods Talk to Service** (Internal Kubernetes)
```
Service sits in front of pods like a receptionist

Pod 1 ←→ Service ←→ Ingress
Pod 2 ←→ Service ←→ Ingress  
Pod 3 ←→ Service ←→ Ingress
```
Service has a stable IP: 10.100.200.50 (example)
Even if pods die and restart, Service IP stays the same


---

## 🔐 **PART 3: Security Flow (Why Public + Private Subnets?)**

### **Public Subnet:**
- ✅ ALB has public IP
- ✅ Internet can reach it
- ✅ No application code runs here

### **Private Subnet:**
- ✅ Pods have NO public IP
- ✅ Internet CANNOT directly reach pods
- ✅ Only ALB can talk to pods
- ✅ More secure (pods can't be attacked directly)

**Traffic MUST go through ALB** (acts as a security gatekeeper)

---

## 🎯 **PART 4: What Makes This "Serverless"?**

### **Traditional (With EC2):**
```
You create EC2 instances
You install Docker
You manage scaling
You patch servers
You worry about capacity
```

### **Your Setup (Fargate):**
```
No EC2 instances
No server management
AWS runs your containers
Auto-scales automatically
You only pay for running pods
```
## **That's why it's called "Serverless Kubernetes"!**
