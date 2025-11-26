## Create a Cluster using EKSCTL

# lazy to type name again and again then do this
```bash
export cluster_name=demo-cluster
export AWS_ClUSTER_REGION=ap-south-1
export AWS_FARGATE_PROFILE="alb-sample-app"
export VPC_ID_EKS="vpc-0150c3faaf19af0f8"
export VPC_CIDR=$(aws ec2 describe-vpcs --vpc-ids $VPC_ID --query "Vpcs[0].CidrBlock" --output text --profile abhinav)
```
# Create Cluster 
```bash
eksctl create cluster --name $cluster_name --region $AWS_ClUSTER_REGION --fargate --profile abhinav
```

# Update KubeCTL config from eks - this will enable you to use the EKS cluster using kubectl.
```bash
aws eks update-kubeconfig --name $cluster_name --region $AWS_ClUSTER_REGION --profile abhinav
```

* There are 2 ways to do the deployment
  * pay for EC2 Instances and Manage it Manually
  * Use Fargate <3 This smol containers shit - which is better and is tuned for HA.
* This Example uses Fargate

# Fargate Profile
```bash
eksctl create fargateprofile --cluster $cluster_name --region $AWS_ClUSTER_REGION --name $AWS_FARGATE_PROFILE --namespace game-2048 --profile abhinav
```

# Create the 2048 App on EKS.
This is going to create several things.
- Namespace, Deployment, Service and Ingress.

Namespace - scopes names, access, and quotas for namespaced objects like Deployments and Services.

Deployment - manages ReplicaSets to create/update Pods and roll out/roll back changes.
Service - stable virtual IP/DNS that groups Pods (usually via label selectors) and exposes them on a network.

Ingress HTTP/HTTPS routing resource that requires an Ingress controller to expose Services; it is not a Service type.

# Running below will deploy the right config as mentioned above for the game.
```bash
kubectl apply -f kube_files/2048_full.yaml
```

# OIDC and ALB

eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve --profile abhinav

Note: AWSLoadBalancerControllerIAMPolicy, has to be this name only

NOTE: Download policy by below - check for updates. just replace the version.
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.1/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json --profile abhinav

eksctl utils associate-iam-oidc-provider --region=ap-south-1 --cluster=demo-cluster --profile abhinav --approve

eksctl create iamserviceaccount \
  --cluster=$cluster_name \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::831869585626:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve --profile abhinav


helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system   --set clusterName=$cluster_name   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller   --set region=$AWS_ClUSTER_REGION   --set vpcId=vpc-0150c3faaf19af0f8

kubectl apply -f kube_files/2048_full.yaml
```
# Creating EKS ALB

```bash
helm repo add eks https://aws.github.io/eks-charts

helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=$cluster_name \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$AWS_ClUSTER_REGION \
  --set vpcId=$VPC_ID_EKS
```
# Metrics Servers will fail on Farge

The Taint on metrics server doesnt let it run so you have to make it available for fargate.

```bash
kubectl patch deployment metrics-server -n kube-system --type=merge --patch='
spec:
  template:
    spec:
      tolerations:
      - key: "eks.amazonaws.com/compute-type"
        operator: "Equal"
        value: "fargate"
        effect: "NoSchedule"
'
```
# KubeCTL Commands
Note: get, describe, edit, exec (pods), -A Flag mostly works on everything for "all".

```bash
kubectl get endpoints -n <namespace>
kubectl get pods -n <namespace>
kubectl get deployments -n <namespace>
kubectl get svc -n <namespace>
kubectl get ingress -n <namespace>
kubectl get namespaces
kubectl get pv -n <namespace>
kubectl get pvc -n <namespace>
kubectl top pods
kubectl rollout restart deployments <podname> -n <namespace>
kubectl logs <podname> -n <namespace>
kubectl apply -f <location of file>/<filename>.yaml
kubectl delete deploy deployment-2048 -n <namespace>
kubectl delete -f <config file name>.yaml # Delete from a file
kubectl get hpa
kubectl describe hpa
kubectl get node
kubectl get serviceaccounts
```
