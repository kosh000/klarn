## Create a Cluster using EKSCTL

# lazy to type name again and again then do this
export cluster_name=demo-cluster
export AWS_ClUSTER_REGION=ap-south-1
export AWS_FARGATE_PROFILE="alb-sample-app"

# Create Cluster 
eksctl create cluster --name $cluster_name --region $AWS_ClUSTER_REGION --fargate

# Update KubeCTL config from eks - this will enable you to use the EKS cluster using kubectl.
aws eks update-kubeconfig --name $cluster_name --region $AWS_ClUSTER_REGION

# There are 2 ways to do the deployment
##### pay for EC2 Instances and Manage it Manually
##### Use Fargate <3 This smol containers shit - which is better and is tuned for HA.
# This Example uses Fargate

# Fargate Profile
eksctl create fargateprofile --cluster $cluster_name --region $AWS_ClUSTER_REGION --name $AWS_FARGATE_PROFILE --namespace game-2048

# Create the 2048 App on EKS.
#   # This is going to create several things.
# # # Namespace, Deployment, Service and Ingress.
##### Namespace - scopes names, access, and quotas for namespaced objects like Deployments and Services.
##### Deployment - manages ReplicaSets to create/update Pods and roll out/roll back changes.
##### Service - stable virtual IP/DNS that groups Pods (usually via label selectors) and exposes them on a network.
##### Ingress HTTP/HTTPS routing resource that requires an Ingress controller to expose Services; it is not a Service type.
# Running below will deploy the right config as mentioned above for the game.
kubectl apply -f kube_files/2048_full.yaml

# KubeCTL Commands
kubectl get endpoints -n game-2048
kubectl get pods -n game-2048
kubectl get deployments.apps -n game-2048
kubectl get svc -n game-2048
kubectl get ingress -n game-2048
kubectl get namespaces