## Create a Cluster using EKSCTL

# lazy to type name again and again then do this
export cluster_name=demo-cluster
export AWS_ClUSTER_REGION=ap-south-1
export AWS_FARGATE_PROFILE="alb-sample-app"

# Create Cluster 
eksctl create cluster --name $cluster_name --region ap-south-1 --fargate

# Update KubeCTL config from eks - this will enable you to use the EKS cluster using kubectl.
aws eks update-kubeconfig --name $cluster_name --region ap-south-1

# There are 2 ways to do the deployment
#   # pay for EC2 Instances and Manage it Manually
#   # Use Fargate <3 This smol containers shit - which is better and is tuned for HA.
# This Example uses Fargate

# Fargate Profile
eksctl create fargateprofile --cluster $cluster_name --region $AWS_ClUSTER_REGION --name alb-sample-app --namespace game-2048

