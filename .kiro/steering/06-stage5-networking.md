---
inclusion: manual
description: "Stage 5: Networking & Ingress — Services, Gateway API, ALB Controller, MetalLB, Envoy Gateway, ExternalDNS, Network Policies"
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

## Exposing Apps: The Full Picture (EKS, On-Prem, Local)

Before diving into Ingress specifics, understand that **how you expose an app depends on where your cluster runs**:

| Environment | LoadBalancer type: Service | Ingress / Gateway | Typical stack |
|-------------|---------------------------|-------------------|---------------|
| **EKS (cloud)** | AWS LB Controller provisions NLB/ALB automatically | AWS LB Controller (Ingress or Gateway API) | ALB + ExternalDNS + ACM certs |
| **On-prem / bare-metal** | Needs MetalLB (or Cilium LB) to assign IPs | Envoy Gateway or Traefik (with Gateway API) | MetalLB + Envoy Gateway + cert-manager |
| **Local (kind/minikube)** | No real LB — stays `<pending>` forever | kubectl port-forward or extraPortMappings | port-forward for dev, extraPortMappings for testing |

This is a critical distinction. On EKS, `type: LoadBalancer` "just works" because the cloud controller manager talks to AWS. On bare metal, there's no cloud — you need software to fill that gap.

### kubectl port-forward (Quick Dev Access)

The simplest way to reach a Service from your machine — works everywhere, no setup needed:

```bash
# Forward local port 8080 to Service port 80
kubectl port-forward svc/my-app 8080:80

# Now access at http://localhost:8080
# Ctrl+C to stop

# Forward to a specific pod (bypasses Service)
kubectl port-forward pod/my-app-abc123 8080:8080
```

**Limitations:** single connection, no load balancing, stops when you Ctrl+C. Fine for debugging, not for real traffic.

### kind Cluster: extraPortMappings

kind runs nodes as Docker containers. To reach NodePort services from your host, you must map ports at cluster creation time:

```yaml
# kind-config.yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080   # Must match your Service's nodePort
        hostPort: 30080        # Port on your machine
        listenAddress: "0.0.0.0"
      - containerPort: 30443
        hostPort: 30443
        listenAddress: "0.0.0.0"
```

```bash
kind create cluster --config kind-config.yaml
```

Then create a NodePort Service with matching port:
```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080    # Must match extraPortMappings containerPort
```

Access at `http://localhost:30080`. This is how you test NodePort/Ingress locally.

## On-Prem / Bare-Metal: MetalLB + Envoy Gateway

### The Problem

On bare metal, `type: LoadBalancer` stays in `<pending>` state forever — there's no cloud provider to create a load balancer. You need two things:

1. **MetalLB** — assigns real IPs to LoadBalancer Services (fills the cloud LB gap)
2. **An ingress/gateway controller** — routes L7 traffic (fills the ALB gap)

### MetalLB (LoadBalancer IP Assignment)

MetalLB makes `type: LoadBalancer` work on bare metal by announcing IPs via ARP (Layer 2) or BGP.

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml

# Wait for pods to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

**Configure an IP pool (Layer 2 mode — simplest):**
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.240-192.168.1.250   # Range of IPs on your network
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
```

Now `type: LoadBalancer` Services get a real IP from that pool. Any machine on the same L2 network can reach it.

**Layer 2 vs BGP:**

| Mode | How it works | Pros | Cons |
|------|-------------|------|------|
| **L2 (ARP)** | One node answers ARP for the VIP | Simple, no router config needed | Single-node bottleneck, failover takes ~10s |
| **BGP** | Advertises routes to your router | True load distribution, fast failover | Requires BGP-capable router |

For learning and small clusters, L2 is fine. For production on-prem, BGP is preferred.

**Alternative: Cilium LB**
If you're already using Cilium as your CNI, it has built-in L2/BGP LoadBalancer support — no MetalLB needed. Cilium uses eBPF for high-performance packet processing.

### Envoy Gateway (On-Prem Ingress — Gateway API)

With Ingress NGINX retired (March 2026), the recommended on-prem ingress controller is **Envoy Gateway** using the **Kubernetes Gateway API**.

```bash
# Install Envoy Gateway
kubectl apply --server-side -f \
  https://github.com/envoyproxy/gateway/releases/download/v1.8.0/install.yaml
```

Then define a Gateway and HTTPRoute (see Gateway API section below).

Envoy Gateway's quickstart docs explicitly recommend MetalLB for bare-metal clusters where no cloud LoadBalancer exists.

## Ingress & AWS Load Balancer Controller (EKS)

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

## Gateway API (The Future — Replaces Ingress)

### Why Gateway API?

The Kubernetes Ingress API (2015) has fundamental limitations:
- All configuration via annotations (non-portable, controller-specific)
- No role separation (one resource for infra + app concerns)
- Limited to HTTP (no TCP/UDP/gRPC natively)
- No traffic splitting, header matching, or request mirroring

**Gateway API** (GA since K8s 1.29) is the official successor. It's not a controller — it's a standard API that multiple controllers implement.

### Key Differences: Ingress vs Gateway API

| Aspect | Ingress | Gateway API |
|--------|---------|-------------|
| Configuration | Annotations (non-portable) | Structured fields (portable) |
| Role separation | None (one resource) | GatewayClass → Gateway → Routes (3 layers) |
| Protocol support | HTTP/HTTPS only | HTTP, gRPC, TCP, UDP, TLS |
| Traffic splitting | Not native | Built-in (weight-based) |
| Header matching | Annotation hacks | First-class support |
| Status | Legacy (still works) | Active development, future standard |

### The Three-Layer Model

```
Platform Admin → GatewayClass (which controller to use)
Cluster Ops    → Gateway (what ports/protocols to listen on, what certs)
App Developer  → HTTPRoute / GRPCRoute / TCPRoute (how to route traffic)
```

This separation means app developers don't need to know about infrastructure details.

### Gateway API on EKS (AWS Load Balancer Controller v3+)

The AWS Load Balancer Controller supports Gateway API in GA since v3. Same controller, new API:

```yaml
# GatewayClass — tells K8s to use AWS LB Controller
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: aws-alb
spec:
  controllerName: gateway.k8s.aws/alb-controller
---
# Gateway — creates an ALB
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  annotations:
    gateway.k8s.aws/scheme: internet-facing
    gateway.k8s.aws/certificate-arn: arn:aws:acm:us-east-1:xxx:certificate/xxx
spec:
  gatewayClassName: aws-alb
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-cert
---
# HTTPRoute — routes traffic to services (app developer creates this)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-routes
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /users
      backendRefs:
        - name: users-service
          port: 80
    - matches:
        - path:
            type: PathPrefix
            value: /orders
      backendRefs:
        - name: orders-service
          port: 80
```

### Gateway API on Bare Metal (Envoy Gateway)

Same API, different controller:

```yaml
# GatewayClass — Envoy Gateway (on-prem)
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
---
# Gateway — Envoy creates an Envoy proxy Deployment + LoadBalancer Service
# MetalLB assigns the external IP
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: envoy
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-tls-cert    # From cert-manager
---
# HTTPRoute — IDENTICAL to the EKS version above
# This is the portability win: app developers write the same routes everywhere
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-routes
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - "api.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /users
      backendRefs:
        - name: users-service
          port: 80
```

### Traffic Splitting (Gateway API Native)

```yaml
# Canary: 90% to v1, 10% to v2 — no annotations, no Istio needed
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
    - name: my-gateway
  rules:
    - backendRefs:
        - name: my-app-v1
          port: 80
          weight: 90
        - name: my-app-v2
          port: 80
          weight: 10
```

### Which Should You Learn?

**Both.** Ingress still works and you'll encounter it in existing clusters. Gateway API is what you'll use for new deployments. The mental model is the same (route external traffic to services), just better structured.

| Situation | Use |
|-----------|-----|
| Existing cluster with Ingress working fine | Keep Ingress, migrate when convenient |
| New EKS deployment (2026+) | Gateway API with AWS LB Controller v3+ |
| New on-prem deployment | Gateway API with Envoy Gateway + MetalLB |
| Need traffic splitting without service mesh | Gateway API (native weight-based routing) |
| Local dev (kind) | kubectl port-forward or Gateway API with Envoy Gateway |

### Comparison: Full Exposure Stack by Environment

| Layer | EKS | On-Prem (bare metal) | Local (kind) |
|-------|-----|---------------------|--------------|
| **IP assignment** | Cloud controller (automatic) | MetalLB (L2/BGP) | extraPortMappings or port-forward |
| **L7 routing** | AWS LB Controller (ALB) | Envoy Gateway | Envoy Gateway or port-forward |
| **TLS certs** | ACM (free, auto-renew) | cert-manager + Let's Encrypt | self-signed or mkcert |
| **DNS** | ExternalDNS → Route53 | ExternalDNS → your DNS, or manual | /etc/hosts |
| **API** | Ingress or Gateway API | Gateway API | Gateway API |

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

### Lab 5.2: Ingress with ALB (EKS)
1. Install AWS Load Balancer Controller
2. Create two different deployments (app-a, app-b)
3. Create an Ingress that routes /a → app-a and /b → app-b
4. Verify both paths work through a single ALB
5. Add TLS with an ACM certificate

### Lab 5.3: Gateway API on EKS
1. Ensure AWS Load Balancer Controller v3+ is installed
2. Create a GatewayClass and Gateway resource
3. Create HTTPRoutes for two services (path-based routing)
4. Verify traffic routes correctly through the ALB
5. Add traffic splitting: 90/10 between two versions of the same app

### Lab 5.4: MetalLB + Envoy Gateway (On-Prem / kind)
1. Create a kind cluster with extraPortMappings for ports 80 and 443
2. Install MetalLB and configure an IP pool (use Docker network range for kind)
3. Install Envoy Gateway
4. Create a GatewayClass, Gateway, and HTTPRoute
5. Deploy two apps and verify path-based routing works from your host machine
6. Compare: the HTTPRoute you wrote is identical to what you'd use on EKS

### Lab 5.5: DNS and Service Discovery
1. Deploy two apps in different namespaces
2. From app-a, curl app-b using: `<service>.<namespace>.svc.cluster.local`
3. Verify short names work within same namespace
4. Check CoreDNS logs to see queries

### Lab 5.6: Network Policies
1. Deploy frontend, backend, and database pods
2. Verify all can talk to all (default)
3. Apply a Network Policy: only frontend → backend → database
4. Verify frontend can't reach database directly
5. Verify backend can't reach frontend

### Lab 5.7: Break Things
1. Delete CoreDNS pods — watch service discovery break
2. Create a Service with wrong selector — no endpoints
3. Set up Ingress with wrong path — 404s
4. Block all egress with Network Policy — watch DNS fail (forgot to allow port 53)
5. On kind: forget extraPortMappings — observe NodePort unreachable from host
6. On bare metal: delete MetalLB — watch LoadBalancer Services go to `<pending>`

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
16. You deploy `type: LoadBalancer` on a bare-metal cluster (no cloud). The Service stays in `<pending>` forever. Why? What do you install to fix it?
17. What's MetalLB? What are its two modes (L2 and BGP)? When would you use each?
18. What's the difference between Ingress (legacy API) and Gateway API? Name 3 advantages of Gateway API.
19. In Gateway API, what are the three resource layers (GatewayClass, Gateway, HTTPRoute)? Who typically creates each one?
20. You write an HTTPRoute on EKS with AWS LB Controller. You then move the same HTTPRoute to an on-prem cluster with Envoy Gateway. Does it work without changes? Why is this significant?
21. You're on a kind cluster and create a NodePort Service on port 30080. You curl `localhost:30080` but get "connection refused." What did you forget?
22. `kubectl port-forward svc/my-app 8080:80` — what does this do? Is it suitable for production traffic? Why or why not?
23. Ingress NGINX was retired in March 2026. What are the two recommended replacements for (a) EKS and (b) on-prem?
24. You want traffic splitting (90% v1, 10% v2) without installing Istio. Which API supports this natively?
25. On bare metal with MetalLB in L2 mode, all traffic for a LoadBalancer VIP goes through a single node. Why? What's the production fix?

## Checklist Before Moving On

- [ ] Understand the four networking problems K8s solves
- [ ] Know how VPC CNI works (real VPC IPs for pods)
- [ ] Can create and use all Service types (ClusterIP, NodePort, LoadBalancer, Headless)
- [ ] Can set up Ingress with AWS Load Balancer Controller (EKS)
- [ ] Understand path-based and host-based routing
- [ ] Know how CoreDNS provides service discovery
- [ ] Can write Network Policies to restrict traffic
- [ ] Understand how kube-proxy implements Service routing
- [ ] Know when to use Ingress vs LoadBalancer Service
- [ ] Understand Gateway API (GatewayClass, Gateway, HTTPRoute) and why it replaces Ingress
- [ ] Can set up Gateway API on EKS (AWS LB Controller v3+)
- [ ] Know how to expose apps on bare metal (MetalLB + Envoy Gateway)
- [ ] Know the difference between MetalLB L2 and BGP modes
- [ ] Can use kubectl port-forward and kind extraPortMappings for local dev
- [ ] Understand the full exposure stack differences: EKS vs on-prem vs local
