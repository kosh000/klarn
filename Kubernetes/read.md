# Master Server Setup Script

* Since we cannot have a "Virtual IP" on AWS - we will require an NLB and this NLB is required before setting up Master Servers.
* 

```bash
#!/bin/bash
# CloudWatch Installation
sudo dnf install -y https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/linux_amd64/amazon-ssm-agent.rpm

# Kubernets PreReqs
# # Disable SWAP
swapoff -a
sed -i '/swap/d' /etc/fstab
# # Load Kernel Modules Required for container networking.
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# # Configure Networking Parameters Ensure packets traverse the bridge correctly.
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

# # Configure SELinux Set SELinux to permissive mode to allow Kubernetes components to access the filesystem required.
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

```

* Kube Install Script

```Bash
# Firewall Disable
systemctl stop firewalld
systemctl disable firewalld

# Define the Version Variables
KUBERNETES_VERSION=v1.29
CRIO_VERSION=v1.29

# Add CRI-O Repo
cat <<EOF | tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=CRI-O
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/rpm/repodata/repomd.xml.key
EOF

# Add Kubernetes Repo
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/rpm/repodata/repomd.xml.key
EOF

dnf install -y cri-o kubelet kubeadm kubectl

systemctl enable --now crio
systemctl enable kubelet
```