# Stage 3: EKS Cluster Setup — Command Log

## Tool Installation

```bash
bash Installations/eksctl.bash     # Install eksctl CLI (EKS cluster management)
bash Installations/kubectl.bash    # Install kubectl CLI (Kubernetes API client)
bash Installations/heml.bash       # Install helm CLI (K8s package manager)
```

## Tool Verification

```bash
eksctl version                     # Check eksctl is installed → 0.227.0
kubectl version --client           # Check kubectl is installed → v1.36.1
helm version                       # Check helm is installed → v4.2.0
```

## Cluster Creation

```bash
# Create EKS cluster from config file (takes ~15 min)
# Creates: VPC, subnets, IAM roles, control plane, node group, security groups
eksctl create cluster -f exec_01_cluster/cluster-config.yaml --profile eks-learning
```

## Tagging Existing Resources

```bash
# Find the CloudFormation stack names eksctl created
aws cloudformation describe-stacks \
  --query 'Stacks[?contains(StackName, `eks-learning`)].StackName' \
  --output text --profile eks-learning

# Tag the cluster stack — propagates to VPC, subnets, IGW, security groups, IAM roles
aws cloudformation update-stack \
  --stack-name eksctl-eks-learning-cluster \
  --use-previous-template \                    # Don't change the template, just add tags
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \  # Required when stack has IAM resources
  --tags Key=Environment,Value=dev Key=team,Value=backend Key=Project,Value=ecommerce Key=CostCenter,Value=engineering Key=Owner,Value=rohan Key=Schedule,Value=office-hours \
  --profile eks-learning

# Tag the nodegroup stack — propagates to ASG, launch template, EC2 instances
aws cloudformation update-stack \
  --stack-name eksctl-eks-learning-nodegroup-workers \
  --use-previous-template \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --tags Key=Environment,Value=dev Key=team,Value=backend Key=Project,Value=ecommerce Key=CostCenter,Value=engineering Key=Owner,Value=rohan Key=Schedule,Value=office-hours \
  --profile eks-learning

# Tag the EKS cluster resource directly (CF tags don't always reach EKS itself)
aws eks tag-resource \
  --resource-arn arn:aws:eks:ap-south-1:851060550361:cluster/eks-learning \
  --tags Environment=dev,team=backend,Project=ecommerce,CostCenter=engineering,Owner=rohan,Schedule=office-hours \
  --profile eks-learning
```

## Cluster Exploration

```bash
kubectl get nodes                  # List all worker nodes (name, status, version)
kubectl get nodes -o wide          # Extended info: IPs, OS, container runtime, AZ
kubectl get pods -n kube-system    # List system pods (CoreDNS, kube-proxy, aws-node, metrics-server)
```

## First Deployment

```bash
# Create Deployment (3 nginx pods) + LoadBalancer Service (provisions AWS ELB)
kubectl apply -f exec_01_cluster/hello-eks.yaml

kubectl get pods                   # List pods in default namespace
kubectl get pods -o wide           # See which node each pod is on + pod IPs
kubectl get svc hello-eks          # Get service details — EXTERNAL-IP is the ELB hostname

# Hit the ELB from internet — traffic flows: internet → ELB → Service → Pod → nginx
curl a49af655aafdc4d378624712618d94b5-83708046.ap-south-1.elb.amazonaws.com
```

## Self-Healing & Scaling

```bash
# Delete a pod — controller sees actual<desired, creates replacement immediately
kubectl delete pod hello-eks-64f7c68ff5-2hjc7

kubectl get pods                   # New pod appears with a new name + new IP

# Scale up: change desired replicas from 3 to 5
kubectl scale deployment hello-eks --replicas=5

# Scale down: change desired replicas from 5 to 3 (K8s terminates 2 pods)
kubectl scale deployment hello-eks --replicas=3

kubectl get pods -o wide           # Confirm 3 pods distributed across nodes
```

## Cordon / Drain / Uncordon

```bash
# Cordon — mark node as unschedulable (existing pods stay, no new ones land here)
kubectl cordon ip-192-168-85-219.ap-south-1.compute.internal

kubectl get nodes                  # Shows "Ready,SchedulingDisabled" for cordoned node

# Scale to 6 — all 3 new pods land on the OTHER node (scheduler skips cordoned one)
kubectl scale deployment hello-eks --replicas=6
kubectl get pods -o wide           # Confirm: new pods only on the non-cordoned node

# Delete the last pod on cordoned node — its replacement also avoids that node
kubectl delete pod hello-eks-64f7c68ff5-9gfzp
kubectl get pods -o wide           # All 6 pods now on the healthy node

# Drain — cordon + evict all non-DaemonSet pods off the node
# --ignore-daemonsets: don't try to evict aws-node/kube-proxy (they must stay per-node)
# --delete-emptydir-data: allow evicting pods that use emptyDir volumes (data is lost)
kubectl drain ip-192-168-85-219.ap-south-1.compute.internal --ignore-daemonsets --delete-emptydir-data

# Uncordon — make node schedulable again (new pods CAN land here now)
kubectl uncordon ip-192-168-85-219.ap-south-1.compute.internal

# Scale back to 3 (existing pods don't rebalance — only new ones go to freed node)
kubectl scale deployment hello-eks --replicas=3
```

## Cleanup (cost savings)

```bash
# Delete LoadBalancer service FIRST — removes the AWS ELB (stops ~$0.025/hr billing)
kubectl delete svc hello-eks

# Delete all resources defined in the manifest (deployment + service)
kubectl delete -f exec_01_cluster/hello-eks.yaml

# Scale nodes to 0 — stops EC2 billing, keeps control plane ($0.10/hr)
eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=0 --nodes-min=0 --profile eks-learning

# Resume learning — bring nodes back (takes ~2 min for nodes to become Ready)
eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=2 --nodes-min=1 --profile eks-learning
```
