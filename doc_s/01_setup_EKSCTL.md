## Create a Cluster using EKSCTL

Configure AWS CLI and use below variables

```bash
export EKS_CLUSTER_NAME=eks-workshop
export AWS_REGION=ap-south-1
```

Run the EKS CTL to deploy using yml.

```bash
curl -fsSL https://raw.githubusercontent.com/aws-samples/eks-workshop-v2/main/cluster/eksctl/cluster.yaml | \
envsubst | eksctl create cluster -f -
```
