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
eksctl create fargateprofile -f kube_files/1_wordpress_eksctl_namespaces.yaml --profile abhinav
```
# Apply the Wordpress Config

```bash
kubectl apply -f kube_files/2_wordpress_full.yaml
```

# Create Secrets for the Databse

kubectl apply -f kube_files/3_mysql_secret.yaml

# Created Mysql Deployment

kubectl apply -f kube_files/4_mysql_deployment.yaml

# Created ROLLING UPDATE FILE

* This updates the existing deployment of wordpress with the env variables
  * Host as `Service Name`
  * Username and Password for the wordpress from `Secret`.
kubectl apply -f kube_files/5_wordpress_update.yaml

# Creating Storage for Apps

EFS is choosen here.

* Creating SG
```bash
# Create the Security Group
export VPC_ID="vpc-0150c3faaf19af0f8"
export GROUP_ID=$(aws ec2 create-security-group \
  --description "EFS Security Group for EKS" \
  --group-name "eks-efs-sg" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text --profile abhinav)

# Verify it was created
echo $GROUP_ID
```
* Creating Ingress
```bash
aws ec2 authorize-security-group-ingress \
  --group-id $GROUP_ID \
  --protocol tcp \
  --port 2049 \
  --cidr $VPC_CIDR \
  --profile abhinav
```
* Creating EFS
```bash
export FILE_SYSTEM_ID=$(aws efs create-file-system \
  --region ap-south-1 \
  --performance-mode generalPurpose \
  --query 'FileSystemId' --output text --profile abhinav)

echo $FILE_SYSTEM_ID
```

* Adding PV and PVC
kubectl apply -f kube_files/6_efs_storage.yaml

# Updated App with PV and PVC

kubectl apply -f kube_files/7_mysql_stateful.yaml