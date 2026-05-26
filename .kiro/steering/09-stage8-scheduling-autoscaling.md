---
inclusion: manual
description: "Stage 8: Scheduling & Autoscaling — affinity, taints, HPA, VPA, Karpenter"
---

# Stage 8: Scheduling & Autoscaling

## Goal
Control where pods run, how they scale, and how the cluster itself grows and shrinks based on demand.

## Pod Scheduling

The scheduler decides which node a pod runs on. You can influence this.

### Node Selector (simplest)
```yaml
spec:
  nodeSelector:
    node-type: gpu        # Only schedule on nodes with this label
    kubernetes.io/arch: amd64
```

### Node Affinity (more expressive)
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # MUST match
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-type
                operator: In
                values: ["compute", "general"]
      preferredDuringSchedulingIgnoredDuringExecution:  # PREFER but not required
        - weight: 80
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-east-1a"]
```

### Pod Affinity / Anti-Affinity
```yaml
spec:
  affinity:
    # Co-locate with other pods (same node/zone)
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: cache
          topologyKey: kubernetes.io/hostname  # Same node

    # Spread away from other pods
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname  # Don't put 2 replicas on same node
```

**Common pattern:** Anti-affinity to spread replicas across nodes/zones for high availability.

### Topology Spread Constraints (modern approach)
```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1                              # Max difference between zones
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule        # or ScheduleAnyway
      labelSelector:
        matchLabels:
          app: my-app
```

### Taints and Tolerations

**Taint** = on a node: "don't schedule here unless you tolerate me"
**Toleration** = on a pod: "I can handle that taint"

```bash
# Taint a node
kubectl taint nodes node-1 dedicated=gpu:NoSchedule

# Remove taint
kubectl taint nodes node-1 dedicated=gpu:NoSchedule-
```

```yaml
# Pod that tolerates the taint
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
```

**Taint effects:**
- `NoSchedule`: don't schedule new pods (existing stay)
- `PreferNoSchedule`: try not to schedule (soft)
- `NoExecute`: evict existing pods too

**Use cases:**
- Dedicated node groups (GPU nodes, high-memory nodes)
- Spot instance nodes (pods must tolerate interruption)
- Maintenance (taint before draining)

## Horizontal Pod Autoscaler (HPA)

Scales the NUMBER of pods based on metrics.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale up when avg CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 60s before scaling up again
      policies:
        - type: Percent
          value: 100                    # Can double pods at once
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
      policies:
        - type: Percent
          value: 25                     # Remove max 25% at a time
          periodSeconds: 60
```

**Requirements:**
- Metrics Server must be installed (provides CPU/memory metrics)
- Pods MUST have resource requests set (HPA calculates % of request)

```bash
# Install metrics server (if not present)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check HPA status
kubectl get hpa
kubectl describe hpa my-app-hpa

# Watch scaling
kubectl get hpa -w
```

### Custom Metrics HPA
Scale on application-specific metrics (requests per second, queue depth):
```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```
Requires Prometheus Adapter or KEDA for custom metrics.

## Vertical Pod Autoscaler (VPA)

Adjusts resource REQUESTS/LIMITS of existing pods (right-sizing).

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"    # Auto, Recreate, Initial, Off
  resourcePolicy:
    containerPolicies:
      - containerName: my-app
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
```

**Modes:**
- `Off`: only recommends (check with `kubectl describe vpa`)
- `Initial`: sets resources only at pod creation
- `Auto`: evicts and recreates pods with new resources

**Warning:** Don't use VPA and HPA on the same metric (CPU). They'll fight. Use VPA for memory, HPA for CPU, or use HPA only.

## Karpenter (Node Autoscaling)

Karpenter is the modern node autoscaler for EKS. It watches for unschedulable pods and provisions the RIGHT instance type automatically.

### How it Works
1. Pod can't be scheduled (no node has enough resources)
2. Karpenter sees the pending pod
3. Karpenter calculates what instance type fits best (considers CPU, memory, GPU, architecture)
4. Launches an EC2 instance
5. Pod gets scheduled on the new node
6. When nodes are underutilized, Karpenter consolidates (moves pods, terminates nodes)

### Install Karpenter
```bash
helm repo add karpenter https://charts.karpenter.sh
helm install karpenter karpenter/karpenter \
  --namespace karpenter --create-namespace \
  --set clusterName=eks-learning \
  --set clusterEndpoint=$(aws eks describe-cluster --name eks-learning --query "cluster.endpoint" --output text)
```

### NodePool (what Karpenter can provision)
```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]    # Use spot for cost savings
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]          # Compute, general, memory optimized
        - key: karpenter.k8s.aws/instance-size
          operator: In
          values: ["medium", "large", "xlarge", "2xlarge"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: "100"          # Max 100 CPUs total across all nodes
    memory: 200Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 60s
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
    - alias: al2023@latest    # Amazon Linux 2023
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: eks-learning
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: eks-learning
  role: KarpenterNodeRole
```

### Karpenter vs Cluster Autoscaler

| Feature | Karpenter | Cluster Autoscaler |
|---------|-----------|-------------------|
| Instance selection | Automatic (picks best fit) | You define node groups |
| Scaling speed | ~60 seconds | 2-5 minutes |
| Consolidation | Built-in (moves pods, removes nodes) | Limited |
| Spot handling | Native (diversified, interruption handling) | Basic |
| Configuration | NodePool (flexible constraints) | ASG per instance type |
| Recommendation | Default for EKS (2025+) | Legacy, still works |

## Spot Instances

Spot instances are 60-90% cheaper but can be interrupted with 2-minute warning.

```yaml
# In Karpenter NodePool
requirements:
  - key: karpenter.sh/capacity-type
    operator: In
    values: ["spot", "on-demand"]
```

**Best practices for spot:**
- Use for stateless workloads (web servers, workers)
- Don't use for databases or single-replica critical services
- Diversify instance types (Karpenter does this automatically)
- Handle graceful shutdown (SIGTERM → 2 min to clean up)

## Pod Priorities and Preemption

When the cluster is full, which pods matter most? PriorityClasses let you define a hierarchy.

### PriorityClass
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-production
value: 1000000          # Higher number = higher priority
globalDefault: false
preemptionPolicy: PreemptLowerPriority  # or Never
description: "For critical production workloads that must always run"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: standard
value: 100000
globalDefault: true     # Default for pods without explicit priority
description: "Standard workloads"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 10000
preemptionPolicy: Never  # Can be evicted but won't evict others
description: "Low-priority batch jobs that can be preempted"
```

### Using in a Pod
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      priorityClassName: critical-production  # This pod is high priority
      containers:
        - name: payment
          image: payment:v1
```

### How Preemption Works
1. High-priority pod can't be scheduled (no resources available)
2. Scheduler looks for nodes where evicting lower-priority pods would make room
3. Lower-priority pods get evicted (graceful termination)
4. High-priority pod gets scheduled on the freed node

**Built-in system priorities (don't override these):**
- `system-cluster-critical` (2000000000) — CoreDNS, kube-proxy
- `system-node-critical` (2000001000) — kubelet-critical pods

**Production pattern:**
```
system-node-critical    → kubelet, CNI (never evict)
system-cluster-critical → CoreDNS, kube-proxy, monitoring
critical-production     → Payment, auth, core APIs
standard                → Regular app workloads (default)
batch-low               → Batch jobs, data processing (evictable)
```

## Evictions

Pods can be removed from nodes in several ways. Understanding eviction is critical for production reliability.

### Types of Eviction

| Type | Trigger | Who does it | Graceful? |
|------|---------|-------------|-----------|
| **Node-pressure eviction** | Node runs low on memory/disk/PIDs | kubelet | Yes (respects grace period) |
| **Preemption** | Higher-priority pod needs resources | Scheduler | Yes |
| **API-initiated eviction** | `kubectl drain`, PDB-aware | API server | Yes (respects PDBs) |
| **Taint-based eviction** | Node gets `NoExecute` taint | Node controller | Yes (respects tolerationSeconds) |

### Node-Pressure Eviction

kubelet monitors node resources and evicts pods when thresholds are breached:

| Signal | Default Threshold | What happens |
|--------|------------------|--------------|
| `memory.available` | < 100Mi | Evict pods (lowest priority first) |
| `nodefs.available` | < 10% | Evict pods using most disk |
| `imagefs.available` | < 15% | Evict pods, garbage collect images |
| `pid.available` | < varies | Evict pods using most PIDs |

**Eviction order (kubelet decides):**
1. Pods exceeding resource requests
2. Lowest priority pods first
3. Pods using most of the starved resource

### Taint-Based Eviction

When a node becomes unhealthy, the node controller adds taints:
```
node.kubernetes.io/not-ready:NoExecute          — node is not ready
node.kubernetes.io/unreachable:NoExecute        — node is unreachable
node.kubernetes.io/memory-pressure:NoSchedule   — node is low on memory
node.kubernetes.io/disk-pressure:NoSchedule     — node is low on disk
node.kubernetes.io/pid-pressure:NoSchedule      — node is low on PIDs
```

Pods without matching tolerations get evicted. You can add `tolerationSeconds` to delay eviction:
```yaml
tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300    # Wait 5 min before evicting (default is 300)
```

### Graceful Shutdown

When a pod is evicted:
1. Pod status → `Terminating`
2. Pod removed from Service endpoints (no new traffic)
3. `preStop` hook runs (if defined)
4. SIGTERM sent to container
5. Wait `terminationGracePeriodSeconds` (default 30s)
6. SIGKILL if still running

```yaml
spec:
  terminationGracePeriodSeconds: 60  # Give app 60s to clean up
  containers:
    - name: my-app
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]  # Wait for LB to drain
```

## Labs

### Lab 8.1: Scheduling Control
1. Label nodes: `kubectl label node <name> tier=frontend`
2. Deploy pods with nodeSelector targeting that label
3. Use pod anti-affinity to spread replicas across nodes
4. Taint a node, deploy pods without toleration — observe Pending
5. Add toleration — observe scheduling

### Lab 8.2: HPA
1. Deploy an app with resource requests
2. Create an HPA targeting 50% CPU
3. Generate load: `kubectl run load --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://my-app; done"`
4. Watch pods scale up: `kubectl get hpa -w`
5. Stop load, watch scale down (after stabilization window)

### Lab 8.3: Karpenter
1. Install Karpenter
2. Create a NodePool allowing t3/m5/c5 instances
3. Deploy 50 replicas of a pod (more than current nodes can handle)
4. Watch Karpenter provision new nodes: `kubectl get nodes -w`
5. Scale down to 2 replicas — watch Karpenter consolidate and remove nodes

### Lab 8.4: Spot Instances
1. Configure Karpenter NodePool with spot + on-demand
2. Deploy workloads — observe spot instances being used
3. Check cost savings in AWS Cost Explorer
4. Simulate spot interruption (terminate instance) — watch pod rescheduling

### Lab 8.5: Topology Spread
1. Deploy 6 replicas with topology spread across 3 AZs
2. Verify even distribution: `kubectl get pods -o wide`
3. Cordon all nodes in one AZ — observe rebalancing behavior

## Self-Test Questions

1. What's the difference between `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`? What does "IgnoredDuringExecution" mean?
2. You taint a node with `dedicated=gpu:NoSchedule`. A pod without any tolerations is already running on that node. Does it get evicted?
3. What's the difference between `NoSchedule`, `PreferNoSchedule`, and `NoExecute` taint effects?
4. HPA says your deployment should have 8 replicas, but you manually set `replicas: 3` in the Deployment. Who wins?
5. You set up HPA targeting 50% CPU utilization, but your pods don't have resource requests defined. What happens?
6. What's the difference between HPA and VPA? Can you use both on the same deployment?
7. Karpenter sees 5 pending pods, each requesting 2 CPU and 4Gi memory. Does it launch 5 separate nodes or 1 big node? Why?
8. What's "consolidation" in Karpenter? When does it happen?
9. You configure Karpenter to use spot instances. A spot instance gets interrupted (2-minute warning). What happens to the pods?
10. What's `topologySpreadConstraints`? How is it different from pod anti-affinity?
11. Your HPA has `stabilizationWindowSeconds: 300` for scale-down. Why? What would happen without it?
12. You have a node with taint `node.kubernetes.io/not-ready:NoExecute`. Where did this taint come from? (You didn't add it.)
13. Karpenter's NodePool allows instance types c5, m5, r5 in sizes medium through 2xlarge. A pod requests 16 CPU. What happens?
14. What's the difference between node scaling (Karpenter) and pod scaling (HPA)? Which responds first to increased load?
15. You set `maxSkew: 1` in topology spread across 3 AZs with 6 replicas. What's the expected distribution?

## Checklist Before Moving On

- [ ] Can use nodeSelector, node affinity, and pod affinity/anti-affinity
- [ ] Understand taints and tolerations
- [ ] Can set up HPA with CPU/memory targets
- [ ] Understand HPA scaling behavior (stabilization windows, policies)
- [ ] Can install and configure Karpenter
- [ ] Understand NodePool and EC2NodeClass configuration
- [ ] Know when to use spot vs on-demand instances
- [ ] Understand Karpenter consolidation behavior
- [ ] Can use topology spread constraints for HA
- [ ] Know the difference between HPA (pod scaling) and Karpenter (node scaling)
- [ ] Understand Pod Priorities and PriorityClasses
- [ ] Know how preemption works (high-priority pods evict low-priority)
- [ ] Understand eviction types (node-pressure, preemption, API-initiated, taint-based)
- [ ] Know how graceful shutdown works (SIGTERM → grace period → SIGKILL)
