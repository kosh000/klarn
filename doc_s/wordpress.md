# Creating Namespaces in Fargate Profiles

* Creating Fargate Profile for Wordpress
```bash
eksctl create fargateprofile \
  --cluster demo-cluster \
  --region ap-south-1 \
  --name wordpress-profile \
  --namespace wordpress \
  --profile abhinav
```
* Create Fargate with multiple namespaces

Reference the file please.
```bash
eksctl create fargateprofile -f kube_files/1_wordpress_eksctl_namespaces.yml --profile abhinav
```
# Apply the Wordpress Config

kubectl apply -f kube_files/2_wordpress_full.yaml
