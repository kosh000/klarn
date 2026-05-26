---
inclusion: manual
---

# Stage 5: Networking & Ingress

## Goal
Understand how networking works in Kubernetes and EKS — from pod-to-pod communication to exposing services to the internet.

## Kubernetes Networking Model

Four networking problems Kubernetes solves:
1. **Container-to-container**: containers in same pod share localhost (network namespace)
2. **Pod-to-pod**: every pod gets a unique IP, can reach any other pod directly (no NAT)
3. **Pod-to-Service**: stable virtual IP (ClusterIP) that load-balances to pod IPs
4. **External-to-Service**: getting traffic from outside the cluster to your pods

### The Flat Network Rule
Every pod can communicate with every other pod without NAT. Each pod gets a real, routable IP address. This is fundamental to Kubernetes networking.

## EKS Networking: VPC CNI

On EKS, the **VPC CNI plugin** (aws-node DaemonSet) assigns real VPC IP addresses to pods.

**How it works:**
- Each node gets secondary IPs from the VPC subnet
- Each pod gets one of these real VPC IPs
- Pods are directly routable within the VPC (no overlay network)
- This means: security groups, NACLs, VPC flow logs all work on pod traffic

**Implication:** You need enough IPs in your subnets. A t3.medium can hold ~17 pods (limited by ENI/IP capacity).

```bash
# See pod IPs (they're real VPC IPs)
kubectl get pods -o wide

# See how many IPs a node can support
kubectl describe node <node-name> | grep -i allocatable -A5
```

## Services Deep Dive

### ClusterIP (default)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80          # Service port (what clients connect to)
      targetPort: 8080  # Pod port (where container listens)
  type: ClusterIP
```
- Only reachable inside the cluster
- Gets a virtual IP from the service CIDR (e.g., 10.100.x.x)
- DNS: `backend.default.svc.cluster.local` or just `backend` (same namespace)

### NodePort
```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080   # Optional: specify port (30000-32767 range)
```
- Exposes on every node's IP at a static port
- Accessible at `<any-node-ip>:30080`
- Rarely used directly in production (use LoadBalancer or Ingress instead)

### LoadBalancer
```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
```
- On EKS: creates an AWS Network Load Balancer (NLB) by default
- Gets an external hostname/IP
- Each LoadBalancer Service = one AWS LB = cost ($$$)
- Fine for 1-2 services, expensive for many

### Headless Service (ClusterIP: None)
```yaml
spec:
  clusterIP: None
  selector:
    app: my-db
```
- No virtual IP assigned
- DNS returns individual pod IPs directly
- Used for StatefulSets where you need to address specific pods

## Ingress & AWS Load Balancer Controller

**Problem:** LoadBalancer creates one LB per service. If you have 20 services, that's 20 LBs ($$$).

**Solution:** Ingress — one load balancer, routes traffic to different services based on hostname or path.

### Load Balancing Concepts

Before diving into Ingress, understand the two layers of load balancing:

| Layer | OSI Layer | What it sees | AWS Resource | Use case |
|-------|-----------|-------------|--------------|----------|
| **L4 (Transport)** | TCP/UDP | IP + port only | NLB (Network Load Balancer) | Raw TCP, gRPC, high performance, TLS passthrough |
| **L7 (Application)** | HTTP/HTTPS | Headers, paths, cookies | ALB (Application Load Balancer) | HTTP routing, path-based, host-based, redirects |

**Key load balancing features:**
- **Session affinity (sticky sessions)**: route same client to same pod. Use when app stores session state in memory (bad practice, but sometimes necessary).
  ```yaml
  spec:
    sessionAffinity: ClientIP
    sessionAffinityConfig:
      clientIP:
        timeoutSeconds: 3600
  ```
- **Connection draining**: when a pod is terminating, finish in-flight requests before killing it. Controlled by `terminationGracePeriodSeconds` on the pod and deregistration delay on the LB.
- **Health checks**: LB only sends traffic to healthy targets. ALB checks HTTP path, NLB checks TCP port.
- **Cross-zone load balancing**: distribute traffic evenly across AZs (enabled by default on ALB, optional on NLB).

**Cost awareness (2026 pricing):**
| Resource | Cost |
|----------|------|
| ALB | ~$0.0225/hour + $0.008/LCU-hour (~$16-30/month typical) |
| NLB | ~$0.0225/hour + $0.006/NLCU-hour (~$16-25/month typical) |
| Classic LB | ~$0.025/hour (legacy, avoid) |

20 services × $20/month = $400/month in LBs alone. Ingress solves this.

### Install AWS Load Balancer Controller
```bash
# This controller watches Ingress resources and creates ALBs/NLBs
# Install via Helm (after setting up Pod Identity/IRSA for it)
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks-learning \
  --set serviceAccount.create=true
```

### Ingress Resource
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:xxx:certificate/xxx
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /users
            pathType: Prefix
            backend:
              service:
                name: users-service
                port:
                  number: 80
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: orders-service
                port:
                  number: 80
    - host: admin.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 80
```

**One ALB → multiple services** based on host/path rules. Much cheaper than multiple LoadBalancer services.

### Key Annotations
```yaml
# Internet-facing vs internal
alb.ingress.kubernetes.io/scheme: internet-facing  # or "internal"

# Target type: ip (recommended) vs instance
alb.ingress.kubernetes.io/target-type: ip

# Health check
alb.ingress.kubernetes.io/healthcheck-path: /healthz

# SSL/TLS
alb.ingress.kubernetes.io/certificate-arn: <acm-cert-arn>
alb.ingress.kubernetes.io/ssl-redirect: "443"
```

## CoreDNS (Service Discovery)

CoreDNS runs as a Deployment in kube-system. It provides DNS for the cluster.

**DNS naming convention:**
```
<service-name>.<namespace>.svc.cluster.local
```

Examples:
- `backend` → resolves if you're in the same namespace
- `backend.default` → explicit namespace
- `backend.default.svc.cluster.local` → fully qualified

```bash
# Test DNS from inside a pod
kubectl run dns-test --image=busybox:1.36 --rm -it -- nslookup backend.default.svc.cluster.local

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml
```

## ExternalDNS

Automatically creates DNS records (Route53) for your Services/Ingresses.

```yaml
# When you create an Ingress with host: api.example.com
# ExternalDNS automatically creates a Route53 A record pointing to the ALB
```

```bash
# Install ExternalDNS via Helm
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm install external-dns external-dns/external-dns \
  --set provider=aws \
  --set domainFilters[0]=example.com \
  --set policy=sync
```

## Network Policies

By default, all pods can talk to all pods. Network Policies restrict this.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend    # Only frontend pods can reach backend
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database    # Backend can only reach database
      ports:
        - protocol: TCP
          port: 5432
    - to:                      # Allow DNS
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

**Important:** On EKS, Network Policies require a CNI that supports them. The VPC CNI supports Network Policies natively (enabled via add-on configuration).

```bash
# Enable Network Policy on EKS VPC CNI
aws eks update-addon --cluster-name eks-learning \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy": "true"}'
```

## Under the Hood: How kube-proxy Works

kube-proxy runs on every node and implements Service routing:

1. Watches API server for Service/Endpoint changes
2. Programs iptables rules (or IPVS rules) on the node
3. When traffic hits a Service ClusterIP, iptables redirects to a random backend pod IP

```bash
# See iptables rules for a service (on a node)
iptables -t nat -L KUBE-SERVICES -n | grep <service-name>
```

## Labs

### Lab 5.1: Service Types
1. Deploy an app with 3 replicas
2. Create a ClusterIP service — verify it works from inside the cluster
3. Change to NodePort — access via node IP
4. Change to LoadBalancer — access via external URL
5. Create a headless service — verify DNS returns pod IPs

### Lab 5.2: Ingress with ALB
1. Install AWS Load Balancer Controller
2. Create two different deployments (app-a, app-b)
3. Create an Ingress that routes /a → app-a and /b → app-b
4. Verify both paths work through a single ALB
5. Add TLS with an ACM certificate

### Lab 5.3: DNS and Service Discovery
1. Deploy two apps in different namespaces
2. From app-a, curl app-b using: `<service>.<namespace>.svc.cluster.local`
3. Verify short names work within same namespace
4. Check CoreDNS logs to see queries

### Lab 5.4: Network Policies
1. Deploy frontend, backend, and database pods
2. Verify all can talk to all (default)
3. Apply a Network Policy: only frontend → backend → database
4. Verify frontend can't reach database directly
5. Verify backend can't reach frontend

### Lab 5.5: Break Things
1. Delete CoreDNS pods — watch service discovery break
2. Create a Service with wrong selector — no endpoints
3. Set up Ingress with wrong path — 404s
4. Block all egress with Network Policy — watch DNS fail (forgot to allow port 53)

## Self-Test Questions

1. Every pod in Kubernetes gets its own IP address. On EKS, where does that IP come from? Is it a "virtual" IP or a real VPC IP?
2. You have 3 pods behind a ClusterIP Service. How does traffic get distributed? What component handles this?
3. What's the DNS name for a Service called `backend` in namespace `production`? What's the short form you can use from the same namespace?
4. You create a Service with `type: LoadBalancer`. On EKS, what AWS resource gets created? What's the cost implication if you have 20 such services?
5. How does Ingress solve the cost problem from question 4?
6. What's the difference between `path: /api` with `pathType: Prefix` vs `pathType: Exact`?
7. You apply a Network Policy that allows ingress only from pods with label `app: frontend`. A pod without that label tries to connect. What happens?
8. You create a Network Policy with only `ingress` rules. Can the pod still make outbound connections? (Trick question — think carefully)
9. CoreDNS pods are in `kube-system`. If you delete them, what breaks? Can pods still talk to each other by IP?
10. What's a headless Service (`clusterIP: None`)? When would you use it?
11. You have a pod in namespace `A` trying to reach a service in namespace `B`. The short name `my-service` doesn't work. Why? What's the fix?
12. On EKS with VPC CNI, a t3.medium node can only run ~17 pods. Why this limit? What determines it?
13. What's the difference between the AWS Load Balancer Controller and the old `service.beta.kubernetes.io/aws-load-balancer-type` annotation approach?
14. You set up ExternalDNS. What does it actually do when you create an Ingress with `host: api.example.com`?
15. A Network Policy blocks all egress. Now your pod can't resolve DNS. Why? What port/protocol do you need to allow?

## Checklist Before Moving On

- [ ] Understand the four networking problems K8s solves
- [ ] Know how VPC CNI works (real VPC IPs for pods)
- [ ] Can create and use all Service types (ClusterIP, NodePort, LoadBalancer, Headless)
- [ ] Can set up Ingress with AWS Load Balancer Controller
- [ ] Understand path-based and host-based routing
- [ ] Know how CoreDNS provides service discovery
- [ ] Can write Network Policies to restrict traffic
- [ ] Understand how kube-proxy implements Service routing
- [ ] Know when to use Ingress vs LoadBalancer Service
