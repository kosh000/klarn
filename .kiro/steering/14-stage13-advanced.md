---
inclusion: manual
description: "Stage 13: Advanced & Serverless — Fargate, EKS Auto Mode, service mesh, custom operators"
---

# Stage 13: Advanced & Serverless

## Goal
Master the highest-level abstractions (Fargate, Auto Mode), service mesh, hybrid deployments, and custom operators. You now understand what these abstract away.

## EKS Fargate

### What is Fargate?
Serverless compute for EKS. No nodes to manage — AWS runs each pod on its own isolated micro-VM.

**What you get:**
- No EC2 instances to manage, patch, or scale
- Per-pod billing (pay only for what pods use)
- Each pod runs in its own kernel (stronger isolation than shared nodes)

**What you lose:**
- No DaemonSets (no nodes to run them on)
- No privileged containers
- No hostNetwork, hostPort, hostPath
- No GPU workloads
- Limited to 4 vCPU / 30 GB memory per pod
- Slower pod startup (~30-60s vs ~5-10s on EC2)
- No SSH into "nodes"

### Fargate Profile
```yaml
# eksctl config
fargateProfiles:
  - name: default
    selectors:
      - namespace: fargate-workloads
        labels:
          compute: fargate
      - namespace: kube-system
        labels:
          k8s-app: coredns
```

```bash
# Create Fargate profile
eksctl create fargateprofile \
  --cluster eks-learning \
  --name my-fargate \
  --namespace fargate-workloads

# Deploy to Fargate (just deploy to the matching namespace)
kubectl create namespace fargate-workloads
kubectl apply -f deployment.yaml -n fargate-workloads

# Pods automatically run on Fargate (no node visible)
kubectl get pods -n fargate-workloads -o wide
# NODE column shows "fargate-ip-xxx"
```

### When to Use Fargate
- Batch jobs that run occasionally
- Dev/test environments (simpler, no node management)
- Workloads with unpredictable scaling patterns
- When you want maximum isolation between pods
- When you don't need DaemonSets or host-level access

### When NOT to Use Fargate
- Need DaemonSets (monitoring agents, log collectors)
- Need GPU
- Need high pod density (Fargate overhead per pod)
- Need fast startup times
- Need persistent local storage
- Cost-sensitive high-throughput workloads (EC2 + spot is cheaper at scale)

## EKS Auto Mode

### What is Auto Mode?
AWS fully manages compute, storage, networking, and add-ons. Uses Karpenter under the hood with Bottlerocket AMIs.

**What AWS manages:**
- Node provisioning (Karpenter-based, automatic instance selection)
- Node patching (Bottlerocket auto-updates)
- Core add-ons (VPC CNI, CoreDNS, kube-proxy, EBS CSI)
- Node scaling and consolidation

**What you manage:**
- Your applications
- Kubernetes RBAC
- Application-level networking (Ingress, Services)

```bash
# Create cluster with Auto Mode
eksctl create cluster \
  --name auto-mode-cluster \
  --region us-east-1 \
  --version 1.35 \
  --auto-mode

# Or enable on existing cluster
aws eks update-cluster-config \
  --name eks-learning \
  --compute-config enabled=true
```

### Auto Mode vs Manual (Managed Node Groups + Karpenter)

| Aspect | Auto Mode | Manual |
|--------|-----------|--------|
| Node provisioning | Automatic | You configure Karpenter NodePools |
| AMI management | AWS handles (Bottlerocket) | You choose AMI |
| Add-on management | AWS handles versions | You manage versions |
| Customization | Limited | Full control |
| Cost | Slightly higher (management fee) | Lower (you manage) |
| Learning value | Low (everything hidden) | High (you see everything) |

### When to Use Auto Mode
- Teams that want to focus purely on applications
- When you don't need custom AMIs or node configurations
- Smaller teams without dedicated platform engineers
- When operational simplicity > cost optimization

## EKS Hybrid Nodes

### What is it?
Run on-premises or edge servers as worker nodes in your EKS cluster. Control plane stays in AWS, data plane runs on your hardware.

**Use cases:**
- Low-latency requirements (data must stay on-prem)
- Regulatory compliance (data residency)
- Edge computing (retail stores, factories)
- Migration path (gradually move to cloud)

```bash
# Create cluster with hybrid node support
eksctl create cluster --name hybrid-cluster --region us-east-1

# Register on-prem node
# On the on-prem server:
sudo /usr/local/bin/nodeadm init \
  --cluster-name hybrid-cluster \
  --region us-east-1
```

### Hybrid Nodes Gateway (2026)
Simplifies networking between VPC and on-premises pods. Automatically manages pod-to-pod traffic across environments using VXLAN tunnels.

### Exposing Apps on Hybrid Nodes (Ingress Traffic Path)

When running workloads on on-prem Hybrid Nodes, external traffic can reach them via two patterns:

**Pattern 1: Cloud Ingress → On-Prem Pods**
```
Internet → ALB (in AWS) → Gateway/Ingress → routes to pods on Hybrid Nodes
```
The ALB lives in AWS. The AWS LB Controller routes traffic to pod IPs on Hybrid Nodes via the Hybrid Nodes Gateway VXLAN tunnel. This works because the Gateway makes on-prem pod IPs routable from the VPC.

**Pattern 2: On-Prem Ingress (fully local)**
```
Local network → MetalLB VIP → Envoy Gateway (on-prem node) → pods on Hybrid Nodes
```
Install MetalLB + Envoy Gateway on the on-prem nodes. Traffic never touches AWS. Use this for low-latency or data-residency requirements.

**Pattern 3: Split Ingress (hybrid)**
```
Internet traffic → ALB (AWS) → cloud pods
Local traffic → MetalLB (on-prem) → on-prem pods
```
Route53 or your DNS splits traffic by source. Cloud users hit the ALB, on-prem users hit the local VIP.

**Key consideration:** If your Hybrid Nodes lose connectivity to the EKS control plane, existing pods keep running but no new scheduling happens. Your on-prem ingress (MetalLB + Envoy Gateway) continues serving traffic independently — it doesn't need the control plane for data-path operations.

## Service Mesh

### What is a Service Mesh?
A dedicated infrastructure layer for service-to-service communication. Handles:
- mTLS (mutual TLS — encrypted communication between all services)
- Traffic management (canary, retries, timeouts, circuit breaking)
- Observability (automatic metrics, traces for every request)
- Access control (which service can talk to which)

### Istio (most popular)
```bash
# Install Istio
curl -L https://istio.io/downloadIstio | sh -
istioctl install --set profile=demo

# Enable sidecar injection for a namespace
kubectl label namespace default istio-injection=enabled

# Now every pod in that namespace gets an Envoy sidecar proxy
```

### How it Works
```
[Your App Container] ←→ [Envoy Sidecar Proxy] ←→ Network ←→ [Envoy Sidecar Proxy] ←→ [Other App Container]
```

Every request goes through the sidecar proxy. The proxy handles TLS, retries, metrics, etc. Your app code doesn't change.

### Traffic Management with Istio
```yaml
# Canary: 90% to v1, 10% to v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
    - my-app
  http:
    - route:
        - destination:
            host: my-app
            subset: v1
          weight: 90
        - destination:
            host: my-app
            subset: v2
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### When to Use a Service Mesh
- Many microservices (10+) that need mTLS
- Need fine-grained traffic control (canary per-service)
- Need automatic observability without code changes
- Compliance requires encrypted service-to-service communication

### When NOT to Use
- Simple architectures (< 5 services)
- Performance-critical paths (sidecar adds ~1-3ms latency)
- Small teams (operational overhead of managing the mesh)

## Custom Controllers and Operators

### What is an Operator?
A custom controller that extends Kubernetes to manage complex applications. It encodes operational knowledge (how to deploy, scale, backup, upgrade) into code.

**Example:** PostgreSQL Operator manages PostgreSQL clusters — handles replication, failover, backups, upgrades automatically.

## Custom Schedulers and Extenders

The default kube-scheduler works for most cases, but you can extend or replace it.

### Scheduler Extenders (Webhooks)
Add custom logic to the default scheduler without replacing it:
```yaml
# Scheduler configuration with extender
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
extenders:
  - urlPrefix: "http://my-scheduler-extender:8080"
    filterVerb: "filter"
    prioritizeVerb: "prioritize"
    weight: 5
    enableHTTPS: false
```

**Use cases for custom scheduling:**
- GPU-aware scheduling (match pod GPU requirements to node GPU types)
- Data locality (schedule pods near their data)
- License-aware scheduling (limit pods per node based on software licenses)
- Cost-aware scheduling (prefer cheaper nodes)

### Scheduling Framework Plugins (K8s 1.19+)
The modern approach — write Go plugins that hook into scheduler phases:
```
QueueSort → PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit → PreBind → Bind → PostBind
```

Each phase is a hook point where you can inject custom logic. This is how Karpenter integrates with scheduling.

### Running Multiple Schedulers
```yaml
# Pod that uses a custom scheduler
spec:
  schedulerName: my-custom-scheduler  # Default is "default-scheduler"
  containers:
    - name: my-app
      image: my-app:v1
```

You can run multiple schedulers simultaneously. Each pod specifies which scheduler should handle it.

## Kubernetes Extensions and APIs

### API Aggregation Layer
Extend the Kubernetes API with your own API servers:
```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1
  groupPriorityMinimum: 100
  versionPriority: 100
```

This is how `kubectl top pods` works — metrics-server registers as an API extension.

### Extension Points Summary

| Extension | What it does | Example |
|-----------|-------------|---------|
| CRDs + Operators | Add new resource types | CloudNativePG, Karpenter NodePool |
| Admission Webhooks | Intercept/modify API requests | Kyverno, Istio sidecar injection |
| Scheduler Extenders | Custom scheduling logic | GPU scheduling, data locality |
| API Aggregation | Extend the K8s API | Metrics Server, custom metrics |
| CSI Drivers | Custom storage backends | EBS CSI, EFS CSI |
| CNI Plugins | Custom networking | VPC CNI, Calico, Cilium |
| Device Plugins | Expose hardware to pods | NVIDIA GPU plugin |

## Self-Managed Clusters (Understanding What EKS Hides)

You're using EKS, which manages the control plane. But understanding what it hides deepens your knowledge.

### What EKS Does For You (That You'd Do Manually)

| Task | EKS handles it | Self-managed (kubeadm) |
|------|---------------|----------------------|
| etcd cluster | Managed, multi-AZ, backed up | You install, configure, backup, restore |
| API server | Managed, auto-scaled, HA | You deploy, configure TLS certs, load balance |
| Certificates | Auto-rotated | You manage CA, issue/rotate certs |
| Scheduler + Controller Manager | Managed | You deploy and configure |
| Upgrades | One API call | You upgrade each component manually |
| etcd encryption | Configurable | You set up KMS encryption |

### kubeadm (How Self-Managed Clusters Work)
```bash
# Initialize control plane (on master node)
kubeadm init --pod-network-cidr=10.244.0.0/16

# Join worker nodes
kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

**Node bootstrapping process:**
1. kubelet starts on the new node
2. kubelet contacts API server with bootstrap token
3. API server issues a client certificate to the node
4. Node is registered and becomes `Ready`
5. Scheduler can now place pods on it

On EKS, this happens automatically via the node group's launch template and the `aws-auth` ConfigMap (legacy) or access entries (modern).

### Why This Matters for EKS Users
- Debugging: when nodes don't join, understanding the bootstrap process helps
- Security: understanding certificate rotation helps with compliance
- Architecture decisions: knowing what EKS hides helps you evaluate alternatives
- Interviews: self-managed cluster knowledge is commonly tested

### Custom Resource Definition (CRD)
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.io
spec:
  group: mycompany.io
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                  enum: ["postgres", "mysql"]
                version:
                  type: string
                storage:
                  type: string
                replicas:
                  type: integer
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
```

### Using a CRD
```yaml
apiVersion: mycompany.io/v1
kind: Database
metadata:
  name: users-db
spec:
  engine: postgres
  version: "16"
  storage: 50Gi
  replicas: 3
```

The operator watches for `Database` resources and creates the necessary StatefulSets, Services, PVCs, ConfigMaps, etc.

### Popular Operators
- **CloudNativePG**: PostgreSQL on Kubernetes
- **Strimzi**: Apache Kafka on Kubernetes
- **Redis Operator**: Redis clusters
- **Prometheus Operator**: (you already used this in Stage 9)
- **Cert-Manager**: Automatic TLS certificate management

```bash
# Install CloudNativePG operator
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.22/releases/cnpg-1.22.0.yaml

# Create a PostgreSQL cluster
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-postgres
spec:
  instances: 3
  storage:
    size: 20Gi
    storageClass: fast-ssd
  postgresql:
    parameters:
      max_connections: "200"
  backup:
    barmanObjectStore:
      destinationPath: s3://my-backups/postgres
```

## Multi-Cluster Management

### Why Multiple Clusters?
- Environment isolation (dev/staging/prod)
- Regional deployment (us-east, eu-west)
- Blast radius reduction (failure in one cluster doesn't affect others)
- Team isolation (each team gets their own cluster)
- Compliance (data residency requirements per region)

### Multi-Cluster Tools

| Tool | What it does | Complexity | Cost |
|------|-------------|-----------|------|
| **ArgoCD** (multi-cluster) | Deploy to multiple clusters from one ArgoCD | Low | Free (OSS) |
| **Kubernetes Federation (KubeFed)** | Sync resources across clusters | High | Free (OSS, deprecated) |
| **Liqo** | Virtual nodes — pods scheduled across clusters transparently | Medium | Free (OSS) |
| **Admiralty** | Multi-cluster scheduling (virtual kubelet approach) | Medium | Free (OSS) |
| **Rancher** | Multi-cluster management UI + lifecycle | Medium | Free (OSS) / Paid (SUSE support) |
| **Rafay** | Managed multi-cluster platform | Low | Paid |
| **AWS EKS Connector** | View external clusters in AWS console | Low | Free |
| **Crossplane** | Manage infrastructure across clouds from K8s | High | Free (OSS) |

### Managing Multiple Clusters with ArgoCD
```bash
# kubectl contexts
kubectl config get-contexts
kubectl config use-context production-eks

# ArgoCD can deploy to multiple clusters
# Register a cluster with ArgoCD
argocd cluster add production-eks
argocd cluster add staging-eks
```

### ApplicationSet (ArgoCD Multi-Cluster Pattern)
```yaml
# Deploy same app to multiple clusters automatically
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-all-clusters
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: 'my-app-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/manifests.git
        path: overlays/production
      destination:
        server: '{{server}}'
        namespace: production
```

### Cross-Cluster Service Discovery
- **AWS Cloud Map** — service registry that works across clusters and even non-K8s services
- **Istio multi-cluster mesh** — transparent service-to-service communication across clusters
- **DNS-based** (Route53 with weighted/latency routing) — simplest approach
- **Skupper** — layer 7 virtual application network across clusters (free, OSS)

### Multi-Cluster Patterns

**Pattern 1: Hub and Spoke**
```
ArgoCD (hub cluster) → deploys to → Cluster A, Cluster B, Cluster C
```
One management cluster, multiple workload clusters.

**Pattern 2: Replicated (Active-Active)**
```
Cluster US-East ←→ Cluster EU-West (both serve traffic, Route53 routes by latency)
```
Same app in multiple regions for low latency and disaster recovery.

**Pattern 3: Specialized Clusters**
```
Cluster "platform" → shared services (monitoring, CI/CD, ArgoCD)
Cluster "team-a"   → team A's workloads
Cluster "team-b"   → team B's workloads
```
Isolation between teams with shared platform services.

## Labs

### Lab 13.1: Fargate
1. Create a Fargate profile for a namespace
2. Deploy a workload — verify it runs on Fargate
3. Try to deploy a DaemonSet — observe it fails
4. Compare startup time: Fargate vs EC2 node
5. Check billing: Fargate vs equivalent EC2

### Lab 13.2: EKS Auto Mode
1. Create a cluster with Auto Mode enabled
2. Deploy workloads — observe automatic node provisioning
3. Scale up — watch nodes appear automatically
4. Scale down — watch consolidation
5. Compare: what would you have configured manually?

### Lab 13.3: Service Mesh (Istio)
1. Install Istio
2. Enable sidecar injection
3. Deploy two services that communicate
4. Verify mTLS is automatic (check with istioctl)
5. Set up traffic splitting (90/10 canary)
6. View automatic metrics in Kiali dashboard

### Lab 13.4: Operators
1. Install CloudNativePG operator
2. Create a 3-replica PostgreSQL cluster using the CRD
3. Connect and write data
4. Kill the primary pod — watch automatic failover
5. Trigger a backup, then restore

### Lab 13.5: Multi-Cluster
1. Create two clusters (us-east-1 and eu-west-1)
2. Configure ArgoCD to manage both
3. Deploy the same app to both clusters
4. Set up Route53 with latency-based routing to both

## Self-Test Questions

1. What's the biggest limitation of Fargate? Name 3 things you CANNOT do on Fargate that you can on EC2 nodes.
2. You deploy a DaemonSet for log collection. It works on EC2 nodes but not on Fargate pods. Why?
3. EKS Auto Mode uses Karpenter under the hood. If you already know how to configure Karpenter, why would you still choose Auto Mode?
4. What's the startup time difference between a pod on EC2 vs Fargate? Why does Fargate take longer?
5. What's a service mesh? In one sentence, what problem does it solve that you can't solve with just Kubernetes Services?
6. Istio adds a sidecar proxy to every pod. What's the performance cost? When is this cost NOT worth it?
7. What's mTLS? Why is it important for microservices? Does Kubernetes provide it by default?
8. What's a CRD (Custom Resource Definition)? How is it different from a built-in resource like a Deployment?
9. You install the CloudNativePG operator and create a `Cluster` resource with 3 replicas. The operator creates StatefulSets, Services, and PVCs for you. What happens if you delete one of those StatefulSets manually?
10. What's the difference between EKS Hybrid Nodes and just running a separate on-prem Kubernetes cluster?
11. You have 50 microservices. Should you use a service mesh? What if you have 3 microservices?
12. Fargate bills per-pod (vCPU-seconds + memory-seconds). EC2 bills per-instance-hour. At what scale does EC2 become cheaper?
13. What's "drift" in the context of EKS Auto Mode? How does it handle node patching differently from managed node groups?
14. You're running a multi-cluster setup with ArgoCD. A developer pushes to git. How do you deploy to cluster-A but NOT cluster-B?
15. Looking back at all 13 stages: if you had to explain to a junior developer why they should learn EC2 nodes before Fargate/Auto Mode, what would you say in 2-3 sentences?

## Checklist — You're Now an EKS Master

- [ ] Understand Fargate: what it abstracts, when to use it, limitations
- [ ] Understand EKS Auto Mode and when it's appropriate
- [ ] Know about EKS Hybrid Nodes for on-prem/edge scenarios
- [ ] Can install and configure a service mesh (Istio)
- [ ] Understand mTLS, traffic management, and observability from mesh
- [ ] Know what CRDs and Operators are
- [ ] Can use operators to manage complex stateful workloads
- [ ] Understand multi-cluster patterns and when to use them
- [ ] Can manage multiple clusters with ArgoCD
- [ ] **Can make informed decisions about WHEN to use each abstraction level**

## Final Reflection

You started from "what is a container?" and now you understand:
- The full Kubernetes architecture (control plane + data plane)
- How to deploy, scale, secure, and observe workloads
- How to automate everything with GitOps
- How to run production-grade infrastructure with Terraform
- How to handle failures, upgrades, and cost optimization
- What Fargate and Auto Mode abstract away (and when that's appropriate)

The key insight: **the more you understand the lower layers, the better decisions you make about which abstractions to use.**
