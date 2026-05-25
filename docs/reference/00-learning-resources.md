# EKS Learning — Reference Resources

## Official Documentation

### Kubernetes
- [Kubernetes Docs](https://kubernetes.io/docs/) — The source of truth
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Blog](https://kubernetes.io/blog/) — Release notes, feature announcements

### Amazon EKS
- [EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [EKS Best Practices Guide](https://docs.aws.amazon.com/eks/latest/best-practices/introduction.html)
- [EKS Blueprints](https://aws-ia.github.io/terraform-aws-eks-blueprints/)
- [AWS Containers Blog](https://aws.amazon.com/blogs/containers/)
- [EKS Workshop](https://www.eksworkshop.com/)

### Tools
- [Helm Docs](https://helm.sh/docs/)
- [Kustomize Docs](https://kustomize.io/)
- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [Karpenter Docs](https://karpenter.sh/docs/)
- [Terraform AWS EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)
- [Prometheus Docs](https://prometheus.io/docs/)
- [Grafana Docs](https://grafana.com/docs/)

## Current Versions (May 2026)

| Component | Version | Notes |
|-----------|---------|-------|
| Kubernetes (upstream) | 1.36 (Haru) | Released April 2026 |
| EKS latest | 1.35 | 1.36 expected soon |
| Helm | 4.x | Major update from 3.x |
| ArgoCD | 2.13+ | CNCF Graduated |
| Karpenter | 1.1+ | Default for EKS |
| Terraform AWS provider | 5.x | |
| Istio | 1.24+ | |
| Fluent Bit | 3.x | |

## Key 2026 Changes to Be Aware Of

1. **Kubernetes 1.36 GA features:**
   - User Namespaces (container root ≠ host root)
   - Fine-Grained Kubelet API Authorization
   - Mutating Admission Policies (CEL-based)
   - SELinux Volume Label Changes
   - Declarative Validation

2. **EKS Auto Mode** — fully managed compute using Karpenter + Bottlerocket

3. **EKS Pod Identity** — replaces IRSA as the recommended approach for pod-level IAM

4. **Karpenter** — default node autoscaler (Cluster Autoscaler is legacy)

5. **Ingress NGINX retirement** — upstream project retiring March 2026. Use AWS Load Balancer Controller instead.

6. **EKS Hybrid Nodes Gateway** — simplifies networking for hybrid (cloud + on-prem) deployments

7. **Helm 4** — native server-side apply patterns

## GitHub Repositories Worth Starring

- [aws/aws-eks-best-practices](https://github.com/aws/aws-eks-best-practices)
- [terraform-aws-modules/terraform-aws-eks](https://github.com/terraform-aws-modules/terraform-aws-eks)
- [kubernetes-sigs/karpenter](https://github.com/kubernetes-sigs/karpenter)
- [argoproj/argo-cd](https://github.com/argoproj/argo-cd)
- [prometheus-community/helm-charts](https://github.com/prometheus-community/helm-charts)
- [external-secrets/external-secrets](https://github.com/external-secrets/external-secrets)
- [cloudnative-pg/cloudnative-pg](https://github.com/cloudnative-pg/cloudnative-pg)
- [kyverno/kyverno](https://github.com/kyverno/kyverno)

## Useful CLI Tools

```bash
# Core
kubectl          # Kubernetes CLI
eksctl           # EKS cluster management
helm             # Package manager
aws              # AWS CLI

# Productivity
kubectx/kubens   # Switch contexts/namespaces quickly
k9s              # Terminal UI for Kubernetes
stern            # Multi-pod log tailing
kubectl-neat     # Clean up kubectl output

# Debugging
kubectl-debug    # Debug running pods
netshoot         # Network troubleshooting container
crictl           # Container runtime CLI (on nodes)

# Security
trivy            # Container image vulnerability scanning
kubeaudit        # Audit cluster security
kube-bench       # CIS benchmark checks
```

## Install All Tools (Linux)

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# eksctl
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# kind (local clusters)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# k9s
curl -sS https://webinstall.dev/k9s | bash

# kubectx + kubens
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens

# stern (multi-pod logs)
curl -Lo stern https://github.com/stern/stern/releases/latest/download/stern_linux_amd64
chmod +x stern && sudo mv stern /usr/local/bin/

# terraform
curl -Lo terraform.zip https://releases.hashicorp.com/terraform/1.9.0/terraform_1.9.0_linux_amd64.zip
unzip terraform.zip && sudo mv terraform /usr/local/bin/

# argocd CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/
```
