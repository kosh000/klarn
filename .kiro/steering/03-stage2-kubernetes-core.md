---
inclusion: manual
---

# Stage 2: Kubernetes Core Concepts

## Goal
Understand the Kubernetes architecture, core objects, and be able to deploy workloads on a local cluster. This is the foundation everything else builds on.

## Why Kubernetes Exists

Docker runs containers on ONE machine. In production you need:
- Run containers across MANY machines
- Restart containers that crash
- Scale up/down based on load
- Route traffic to healthy containers
- Roll out updates without downtime
- Manage configuration and secrets

Kubernetes solves all of this. It's a container orchestration platform.

## Architecture

### Control Plane (the brain)

| Component | What it does |
|-----------|-------------|
| **API Server** (kube-apiserver) | Front door to the cluster. All communication goes through here. RESTful API. |
| **etcd** | Key-value store. Stores ALL cluster state. The single source of truth. |
| **Scheduler** (kube-scheduler) | Decides which node a new pod runs on. Considers resources, affinity, taints. |
| **Controller Manager** (kube-controller-manager) | Runs controllers that watch state and make reality match desired state. |
| **Cloud Controller Manager** | Integrates with cloud provider (AWS). Manages load balancers, nodes, routes. |

### Data Plane (the workers)

| Component | What it does |
|-----------|-------------|
| **kubelet** | Agent on each node. Ensures containers are running as specified. Reports to API server. |
| **kube-proxy** | Network proxy on each node. Implements Service networking (iptables/IPVS rules). |
| **Container Runtime** | Actually runs containers. containerd is the standard (Docker shim removed in 1.24). |

### How They Work Together
1. You submit a Deployment to the API Server (via kubectl)
2. API Server stores it in etcd
3. Controller Manager sees the Deployment, creates ReplicaSet, which creates Pod specs
4. Scheduler sees unscheduled Pods, assigns them to nodes
5. kubelet on the assigned node pulls the image and starts the container
6. kube-proxy sets up networking so the Pod is reachable

## Core Objects

### Pod
- Smallest deployable unit (NOT a container — a pod CAN have multiple containers)
- Usually 1 container per pod (multi-container is for sidecars, init containers)
- Has its own IP address
- Containers in a pod share network namespace (localhost) and can share volumes
- Pods are ephemeral — they die and get replaced, never "healed"

### Deployment
- Manages a set of identical pods (replicas)
- Handles rolling updates and rollbacks
- You almost never create pods directly — you create Deployments

### ReplicaSet
- Ensures N copies of a pod are running
- Created automatically by Deployments
- You rarely interact with these directly

### Service
- Stable network endpoint for a set of pods
- Pods come and go (new IPs each time) — Services provide a fixed IP/DNS name
- Types:
  - **ClusterIP**: internal only (default)
  - **NodePort**: exposes on each node's IP at a static port
  - **LoadBalancer**: provisions external load balancer (ALB/NLB on AWS)

### Namespace
- Virtual cluster within a cluster
- Isolates resources (pods in namespace A can't see pods in namespace B by default)
- Default namespaces: `default`, `kube-system`, `kube-public`, `kube-node-lease`

### ConfigMap
- Stores non-sensitive configuration as key-value pairs
- Injected into pods as environment variables or mounted as files

### Secret
- Like ConfigMap but for sensitive data (passwords, tokens, keys)
- Base64 encoded (NOT encrypted by default — just encoded)
- Can be encrypted at rest with KMS on EKS

## kubectl — Your Primary Tool

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes

# Namespaces
kubectl get namespaces
kubectl create namespace my-namespace

# Pods
kubectl get pods                          # In default namespace
kubectl get pods -n kube-system           # In kube-system namespace
kubectl get pods -A                       # All namespaces
kubectl describe pod <name>               # Detailed info
kubectl logs <pod-name>                   # View logs
kubectl logs <pod-name> -f                # Follow logs
kubectl exec -it <pod-name> -- bash       # Shell into pod
kubectl delete pod <pod-name>             # Delete (will be recreated by Deployment)

# Deployments
kubectl get deployments
kubectl describe deployment <name>
kubectl scale deployment <name> --replicas=5
kubectl rollout status deployment <name>
kubectl rollout history deployment <name>
kubectl rollout undo deployment <name>

# Services
kubectl get services
kubectl describe service <name>

# Apply manifests
kubectl apply -f manifest.yaml            # Create or update
kubectl delete -f manifest.yaml           # Delete
kubectl diff -f manifest.yaml             # Preview changes

# Debugging
kubectl get events --sort-by='.lastTimestamp'
kubectl top pods                          # Resource usage (needs metrics-server)
kubectl top nodes
```

## Writing Manifests

### Pod (you rarely write these directly)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
    - name: my-container
      image: nginx:1.27
      ports:
        - containerPort: 80
```

### Deployment (this is what you use)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v1
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app          # Matches pods with this label
  ports:
    - port: 80           # Service port
      targetPort: 8080   # Container port
  type: ClusterIP        # Internal only
```

### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  LOG_LEVEL: "info"
  config.json: |
    {
      "feature_flags": {
        "new_ui": true
      }
    }
```

## Labels and Selectors

Labels are key-value pairs attached to objects. Selectors filter objects by labels.

```yaml
# On a pod/deployment
metadata:
  labels:
    app: my-app
    tier: backend
    environment: production
    version: v2

# On a service (selector)
spec:
  selector:
    app: my-app
    tier: backend
```

This is how Services find Pods, how Deployments manage Pods, how you query objects.

## Local Cluster with kind

```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name learning

# Verify
kubectl cluster-info
kubectl get nodes

# Delete when done
kind delete cluster --name learning
```

## Labs

### Lab 2.1: Local Cluster + First Deployment
1. Install kind and kubectl
2. Create a local cluster
3. Deploy nginx: `kubectl create deployment nginx --image=nginx:1.27 --replicas=3`
4. Expose it: `kubectl expose deployment nginx --port=80 --type=NodePort`
5. List pods, describe them, check events
6. Delete a pod and watch it get recreated

### Lab 2.2: Write Manifests from Scratch
1. Write a Deployment manifest for your Docker app from Stage 1
2. Write a Service manifest (ClusterIP)
3. Write a ConfigMap and mount it as environment variables
4. Apply all three, verify the app works
5. Update the image tag, apply again, watch the rolling update

### Lab 2.3: Explore the Control Plane
```bash
# See control plane pods
kubectl get pods -n kube-system

# Check API server
kubectl get --raw /healthz

# See what the scheduler decided
kubectl describe pod <any-pod> | grep -A5 "Events"

# Watch the controller manager work
kubectl scale deployment nginx --replicas=5
kubectl get pods -w  # Watch pods appear in real-time
```

### Lab 2.4: Break Things
1. Set memory limit to 10Mi on a pod and watch it OOM
2. Use a non-existent image and see ImagePullBackOff
3. Delete all nodes (kind) and see what happens to pods
4. Create a Service with wrong selector — traffic goes nowhere
5. Fill up a pod's filesystem and observe behavior

### Lab 2.5: Namespaces
1. Create namespaces: `dev`, `staging`, `production`
2. Deploy the same app in each namespace
3. Try to access a service across namespaces (hint: `<service>.<namespace>.svc.cluster.local`)
4. Set a default namespace: `kubectl config set-context --current --namespace=dev`

## Key Mental Models

### Desired State vs Actual State
- You tell Kubernetes WHAT you want (desired state): "I want 3 replicas of my app"
- Kubernetes continuously works to MAKE REALITY match: controllers reconcile
- If a pod dies, the controller sees "actual=2, desired=3" and creates a new one
- This is called the **reconciliation loop**

### Everything is a Resource
- Pods, Services, Deployments, ConfigMaps — all are "resources" in the API
- Each has: apiVersion, kind, metadata, spec (what you want), status (what exists)
- You interact with all of them the same way: kubectl get/describe/apply/delete

### Labels are Everything
- Kubernetes uses labels to connect things
- Service → finds Pods by label selector
- Deployment → manages Pods by label selector
- You query things by labels: `kubectl get pods -l app=my-app,tier=backend`

## Self-Test Questions

1. What are the 5 components of the Kubernetes control plane? What does each one do in ONE sentence?
2. If etcd dies, what happens to the cluster? Can existing pods keep running?
3. You create a Deployment with 3 replicas. Walk me through what happens step by step (which components are involved and in what order).
4. What's the difference between a Pod and a Deployment? Why don't you create Pods directly?
5. A pod has IP `10.0.1.15`. You delete it and the Deployment creates a new one. Does the new pod have the same IP? Why is this a problem?
6. How does a Service solve the problem from question 5?
7. What's the difference between ClusterIP and NodePort? When would you use each?
8. You have a Service with selector `app: backend` but no pods have that label. What happens when you curl the Service?
9. What does `kubectl describe pod <name>` show you that `kubectl get pod` doesn't?
10. A pod is in `Pending` state. Name 3 possible reasons.
11. A pod is in `CrashLoopBackOff`. What does this mean? How do you debug it?
12. What's a namespace? If you have a pod in namespace `dev` and a pod in namespace `prod`, can they communicate by default?
13. You run `kubectl delete pod my-pod` but it comes back immediately. Why?
14. What's the difference between `kubectl apply -f` and `kubectl create -f`?
15. Explain the "reconciliation loop" in your own words. What's "desired state" vs "actual state"?

## Checklist Before Moving On

- [ ] Can explain control plane vs data plane and what each component does
- [ ] Can create a local cluster with kind
- [ ] Can write Deployment, Service, ConfigMap manifests from scratch
- [ ] Understand the reconciliation loop (desired vs actual state)
- [ ] Can use kubectl fluently: get, describe, logs, exec, apply, delete
- [ ] Understand labels and selectors
- [ ] Know the difference between Pod, Deployment, ReplicaSet
- [ ] Know the Service types: ClusterIP, NodePort, LoadBalancer
- [ ] Can debug basic issues: ImagePullBackOff, CrashLoopBackOff, Pending
