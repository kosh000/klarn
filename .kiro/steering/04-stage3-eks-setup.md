---
inclusion: manual
description: "Stage 3: EKS Cluster Setup — eksctl, managed nodes, first workload deployment"
---

# Stage 3: EKS Cluster Setup (EC2 Managed Nodes)

## Goal
Create a real EKS cluster on AWS, deploy a workload, and understand everything that was created under the hood.

## Cluster Lifecycle Strategy (Learning)

**Decisions made:**
- **No NAT Gateway** — nodes in public subnets only (saves ~$33/month)
- **Scale nodes to 0 when idle** — only control plane charged (~$73/month)
- **Public subnets for nodes** — less secure but fine for learning, ingress works the same
- **One cluster, kept alive** — don't destroy/recreate. Scale nodes up/down as needed.

**Cost model:**
- Active learning: ~$133/month (control plane + 2 nodes)
- Idle (nodes=0): ~$73/month (control plane only)
- Destroyed: $0

**Commands:**
```bash
# Stop learning for today (keep cluster, remove nodes)
eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=0 --nodes-min=0 --profile eks-learning

# Resume learning (bring nodes back)
eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=2 --profile eks-learning
```

**When we have Terraform + ArgoCD (Stages 10-11):** destroy/recreate becomes trivial — one command rebuilds everything from code.

## What is EKS?

Amazon Elastic Kubernetes Service. AWS manages the control plane (API server, etcd, scheduler, controller manager) for you. You manage (or let AWS manage) the worker nodes.

**What AWS handles:**
- Control plane availability (multi-AZ)
- etcd backups
- Kubernetes version upgrades (control plane)
- API server scaling
- Security patches for control plane

**What you handle:**
- Worker nodes (EC2 instances or Fargate)
- Your applications
- Networking configuration
- IAM permissions
- Add-ons (CoreDNS, kube-proxy, VPC CNI, etc.)

## Node Types on EKS

| Type | What it is | When to use |
|------|-----------|-------------|
| **Managed Node Groups** | EC2 instances managed by AWS (AMI updates, scaling) | Default choice. Full control + AWS automation. |
| **Self-Managed Nodes** | EC2 instances you fully manage | Custom AMIs, special requirements |
| **Fargate** | Serverless — no nodes to manage | Simple workloads, no DaemonSets needed |
| **EKS Auto Mode** | Fully managed compute (Karpenter under hood) | Maximum automation, less control |

**We start with Managed Node Groups** — you get real EC2 instances you can SSH into and inspect.

## Prerequisites

```bash
# Install eksctl
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install kubectl (if not already)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify AWS credentials
aws sts get-caller-identity
```

## Create Your First Cluster

### Quick Way (eksctl defaults)
```bash
eksctl create cluster \
  --name eks-learning \
  --region ap-south-1 \
  --version 1.35 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 0 \
  --nodes-max 4 \
  --managed \
  --profile eks-learning
```

This takes ~15 minutes. It creates:
- VPC with public and private subnets across 2-3 AZs
- Internet Gateway + NAT Gateways
- EKS control plane
- IAM roles (cluster role + node role)
- Managed node group with 2 x t3.medium instances
- Security groups
- OIDC provider (for Pod Identity/IRSA)

### Config File Way (recommended — version controlled)
```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eks-learning
  region: ap-south-1
  version: "1.35"

vpc:
  nat:
    gateway: Disable  # No NAT — saves ~$33/month

managedNodeGroups:
  - name: workers
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 0          # Allows scaling to 0 when idle
    maxSize: 4
    volumeSize: 20
    privateNetworking: false  # Nodes in public subnets
    ssh:
      allow: true  # Enable SSH for learning (disable in production)
    iam:
      withAddonPolicies:
        ebs: true
        efs: true
        albIngress: true
        cloudWatch: true
```

```bash
eksctl create cluster -f cluster-config.yaml --profile eks-learning
```

## After Cluster Creation

```bash
# Verify cluster
kubectl get nodes
kubectl get pods -n kube-system

# See what's running in kube-system
kubectl get pods -n kube-system -o wide

# You should see:
# - coredns (DNS for service discovery)
# - kube-proxy (network rules on each node)
# - aws-node (VPC CNI - pod networking)
```

## Understand What Was Created

### VPC and Networking
```bash
# See the VPC
aws ec2 describe-vpcs --filters "Name=tag:eksctl.cluster.k8s.io/v1alpha1/cluster-name,Values=eks-learning"

# See subnets
aws ec2 describe-subnets --filters "Name=tag:eksctl.cluster.k8s.io/v1alpha1/cluster-name,Values=eks-learning" --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,Public:MapPublicIpOnLaunch}'
```

**VPC layout:**
- Public subnets: have route to Internet Gateway (for load balancers)
- Private subnets: route through NAT Gateway (for nodes — they can reach internet but aren't directly exposed)
- Nodes run in private subnets (security best practice)

### IAM Roles
```bash
# Cluster role (allows EKS to manage AWS resources)
aws iam list-roles --query 'Roles[?contains(RoleName, `eks-learning`)].[RoleName,Arn]'
```

Two key roles:
1. **Cluster Role**: allows EKS control plane to manage AWS resources
2. **Node Role**: allows worker nodes to pull images from ECR, register with cluster, etc.

### Node Group
```bash
# See node group details
aws eks describe-nodegroup --cluster-name eks-learning --nodegroup-name workers

# See actual EC2 instances
aws ec2 describe-instances --filters "Name=tag:eks:cluster-name,Values=eks-learning" --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,AZ:Placement.AvailabilityZone,IP:PrivateIpAddress,State:State.Name}'
```

### SSH into a Node (for learning)
```bash
# Find node public IP (if SSH enabled)
kubectl get nodes -o wide

# SSH in
ssh -i ~/.ssh/your-key.pem ec2-user@<node-ip>

# Once inside, explore:
systemctl status kubelet          # The node agent
journalctl -u kubelet -f          # kubelet logs
crictl ps                         # Running containers (via containerd)
ls /var/log/pods/                 # Pod logs on disk
cat /etc/kubernetes/kubelet/kubelet-config.json  # kubelet config
```

## Deploy Your First Workload on EKS

```yaml
# my-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-eks
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-eks
  template:
    metadata:
      labels:
        app: hello-eks
    spec:
      containers:
        - name: hello
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: hello-eks
spec:
  selector:
    app: hello-eks
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer  # Creates an AWS Classic/NLB load balancer
```

```bash
kubectl apply -f my-app.yaml

# Wait for LoadBalancer to get external IP
kubectl get svc hello-eks -w

# Once EXTERNAL-IP appears, curl it
curl http://<EXTERNAL-IP>
```

## EKS Add-ons

Pre-installed components that EKS manages:

| Add-on | Purpose |
|--------|---------|
| **VPC CNI** (aws-node) | Assigns real VPC IPs to pods |
| **CoreDNS** | DNS for service discovery inside cluster |
| **kube-proxy** | Network rules for Service routing |
| **EBS CSI Driver** | Allows pods to use EBS volumes |

```bash
# List installed add-ons
aws eks list-addons --cluster-name eks-learning

# See add-on details
aws eks describe-addon --cluster-name eks-learning --addon-name vpc-cni
```

## Cost Awareness

**EKS pricing (2026):**
- Control plane: $0.10/hour (~$73/month) per cluster
- Worker nodes: standard EC2 pricing (t3.medium ~$0.0416/hour)
- Data transfer, load balancers, EBS volumes: additional costs

**For learning — keep costs down:**
- Use t3.medium or t3.small nodes
- Scale to 0 nodes when not using: `eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=0`
- Delete cluster when done for the day: `eksctl delete cluster --name=eks-learning`
- Use one cluster, multiple namespaces (not multiple clusters)

## Labs

### Lab 3.1: Create and Explore
1. Create an EKS cluster with eksctl
2. Deploy nginx with 3 replicas
3. Expose with LoadBalancer service
4. Access it from your browser
5. SSH into a node, run `crictl ps` to see containers

### Lab 3.2: Understand the Infrastructure
1. Find the VPC, subnets, and security groups created
2. Identify which subnets are public vs private
3. Find the NAT Gateway and understand why it exists
4. Find the IAM roles and understand their policies
5. Check the OIDC provider (used for Pod Identity later)

### Lab 3.3: Node Operations
1. Cordon a node: `kubectl cordon <node-name>` (no new pods scheduled)
2. Drain a node: `kubectl drain <node-name> --ignore-daemonsets` (evict pods)
3. Watch pods get rescheduled to the other node
4. Uncordon: `kubectl uncordon <node-name>`

### Lab 3.4: Scale the Node Group
```bash
# Scale up
eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=3

# Watch new node join
kubectl get nodes -w

# Scale down
eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=1

# Watch pods get rescheduled
kubectl get pods -o wide -w
```

### Lab 3.5: Clean Up (IMPORTANT for cost)
```bash
# Delete all services with LoadBalancers first (otherwise LB lingers)
kubectl delete svc --all

# Delete cluster
eksctl delete cluster --name=eks-learning

# Verify no lingering resources
aws ec2 describe-vpcs --filters "Name=tag:eksctl.cluster.k8s.io/v1alpha1/cluster-name,Values=eks-learning"
```

## Self-Test Questions

1. What does EKS manage for you vs what do YOU manage?
2. You run `eksctl create cluster`. Name 5 AWS resources it creates behind the scenes.
3. Why do worker nodes run in private subnets? What would happen if they were in public subnets?
4. What's the difference between a managed node group and a self-managed node group?
5. You SSH into a node and run `crictl ps`. What are you seeing? Why not `docker ps`?
6. A pod on your EKS cluster has IP `10.0.2.47`. Is this a "fake" cluster IP or a real VPC IP? Why does this matter?
7. You have 2 nodes, each t3.medium. Roughly how many pods can each node run? What limits this?
8. You delete your EKS cluster but notice an AWS load balancer still exists and is costing money. Why? How do you prevent this?
9. What's the OIDC provider that eksctl creates? What is it used for?
10. You run `kubectl cordon node-1`. What happens to existing pods on that node? What about new pods?
11. What's the difference between `cordon` and `drain`?
12. Your cluster has 2 nodes and you deploy 10 replicas each requesting 1 CPU. t3.medium has 2 vCPUs. What happens?
13. What are EKS add-ons? Name 3 and what they do.
14. You want to give a pod access to read from S3. What's the modern (2026) way to do this on EKS?
15. EKS costs $0.10/hour for the control plane. If you have 3 clusters running 24/7, what's your monthly control plane cost?

## Checklist Before Moving On

- [ ] Can create an EKS cluster with eksctl
- [ ] Can deploy workloads and expose them with LoadBalancer
- [ ] Understand what eksctl created (VPC, subnets, IAM, node groups)
- [ ] Can SSH into a node and inspect kubelet, containers
- [ ] Understand public vs private subnets and why nodes go in private
- [ ] Know how to scale node groups up/down
- [ ] Know how to cordon/drain nodes
- [ ] Can clean up clusters to avoid costs
- [ ] Understand EKS add-ons (VPC CNI, CoreDNS, kube-proxy)
