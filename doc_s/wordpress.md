# Creating Fargate Profile for Wordpress
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region ap-south-1 \
  --name wordpress-profile \
  --namespace wordpress \
  --profile abhinav

# Apply the Wordpress Config

kubectl apply -f kube_files/wordpress_full.yaml

