---
inclusion: auto
description: "Stage 0: Prerequisites — Linux, networking, YAML, AWS basics (IAM, VPC, EC2)"
---

# Stage 0: Prerequisites

## Goal
Build the foundation skills needed before touching Kubernetes. Without these, everything else will feel like magic you can't debug.

## What You Need to Know

### Linux Fundamentals
- Process model: what's a process, PID, parent/child, signals (SIGTERM, SIGKILL)
- Filesystem: paths, permissions (chmod, chown), /proc, /sys
- systemd: services, units, journalctl for logs
- Package management: apt/yum basics
- Users and groups
- Environment variables
- stdin/stdout/stderr, pipes, redirection

### Networking
- IP addresses (IPv4, CIDR notation like 10.0.0.0/16)
- TCP vs UDP, ports, sockets
- DNS: how name resolution works, /etc/resolv.conf
- HTTP/HTTPS basics
- Subnets, routing tables, NAT, gateways
- Firewalls: iptables basics (you'll see these in Kubernetes)
- curl, dig, nslookup, netstat/ss

### YAML
- Indentation (spaces, not tabs)
- Maps (key: value)
- Lists (- item)
- Multi-line strings (| and >)
- Anchors and aliases (&anchor, *alias)
- You will write HUNDREDS of YAML files in Kubernetes

### AWS Basics
- AWS account setup, IAM users, MFA
- IAM: users, groups, roles, policies (JSON policy documents)
- VPC: what it is, subnets (public vs private), internet gateway, NAT gateway
- EC2: instances, security groups, key pairs
- S3: buckets, objects (used for Terraform state later)
- AWS CLI: configure, basic commands
- Regions and Availability Zones

## Labs

### Lab 0.1: Linux Process Exploration
```bash
# See all processes
ps aux

# See process tree
pstree -p

# Run something in background
sleep 300 &

# Find it
ps aux | grep sleep

# Send signals
kill -SIGTERM <PID>
kill -SIGKILL <PID>

# Check what's listening on ports
ss -tlnp
```

### Lab 0.2: Networking Basics
```bash
# Check your IP
ip addr show

# DNS lookup
dig google.com
nslookup google.com

# Check routing table
ip route show

# Test connectivity
curl -v https://httpbin.org/get

# Check what's listening
ss -tlnp
```

### Lab 0.3: YAML Practice
Create a file `practice.yaml`:
```yaml
# This is a comment
application:
  name: my-app
  version: "1.0.0"
  replicas: 3
  ports:
    - containerPort: 8080
      protocol: TCP
    - containerPort: 9090
      protocol: TCP
  environment:
    - name: DATABASE_URL
      value: "postgres://localhost:5432/mydb"
    - name: LOG_LEVEL
      value: "info"
  labels:
    app: my-app
    tier: backend
    environment: production
```
Validate it: `python3 -c "import yaml; yaml.safe_load(open('practice.yaml'))"`

### Lab 0.4: AWS Setup
```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials
aws configure
# Enter: Access Key ID, Secret Access Key, Region (us-east-1), Output (json)

# Verify
aws sts get-caller-identity

# List VPCs
aws ec2 describe-vpcs

# List EC2 instances
aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,Type:InstanceType}'
```

## Self-Test Questions

Answer these yourself. If you're unsure, research it. Then ask me to check your answers.

### Linux
1. What happens when you run `kill -9 <PID>` vs `kill -15 <PID>`? Why does it matter for containers?
2. What is PID 1 in a Linux system? What happens if PID 1 dies?
3. If a process is using 100% CPU, how do you find it? What command shows real-time resource usage?
4. What's the difference between a process and a thread?
5. What does `chmod 755` mean? Break down each digit.

### Networking
6. You have a subnet `10.0.1.0/24`. How many usable IP addresses does it have? Why not 256?
7. What's the difference between TCP and UDP? Give one real-world use case for each.
8. A service is running on port 8080 but you can't reach it from another machine. List 3 possible reasons.
9. What does NAT do? Why do private subnets need a NAT Gateway to reach the internet?
10. What's the difference between a security group and a NACL in AWS?

### YAML
11. What's wrong with this YAML?
```yaml
name: my-app
  version: 1.0
ports:
- 8080
- 9090
```
12. What's the difference between `|` and `>` in YAML multi-line strings?

### AWS
13. What's the difference between an IAM User and an IAM Role? When would you use each?
14. Can an EC2 instance in a private subnet reach the internet? How?
15. What's the difference between an Internet Gateway and a NAT Gateway?

## Checklist Before Moving On

- [ ] Can navigate Linux filesystem, manage processes, read logs with journalctl
- [ ] Understand IP addresses, subnets, DNS, ports, TCP
- [ ] Can write valid YAML from scratch without syntax errors
- [ ] AWS CLI configured and working
- [ ] Understand IAM roles vs users vs policies
- [ ] Know what a VPC is and why subnets are public vs private
