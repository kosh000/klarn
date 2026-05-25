---
inclusion: manual
---

# Stage 12: Production Operations

## Goal
Run EKS reliably in production — handle upgrades, backups, cost optimization, chaos testing, and troubleshooting.

## Cluster Upgrades

### Upgrade Strategy
EKS supports N-2 versions. You must upgrade incrementally (1.33 → 1.34 → 1.35, not 1.33 → 1.35).

### Upgrade Order
1. **Control plane** (AWS manages this, takes ~25 min)
2. **Add-ons** (CoreDNS, kube-proxy, VPC CNI, EBS CSI)
3. **Worker nodes** (managed node group update)

```bash
# 1. Check current version
aws eks describe-cluster --name production-eks --query 'cluster.version'

# 2. Check available versions
aws eks describe-addon-versions --addon-name coredns --kubernetes-version 1.35

# 3. Upgrade control plane
aws eks update-cluster-version --name production-eks --kubernetes-version 1.35

# 4. Wait for completion
aws eks wait cluster-active --name production-eks

# 5. Update add-ons
aws eks update-addon --cluster-name production-eks --addon-name vpc-cni --resolve-conflicts OVERWRITE
aws eks update-addon --cluster-name production-eks --addon-name coredns --resolve-conflicts OVERWRITE
aws eks update-addon --cluster-name production-eks --addon-name kube-proxy --resolve-conflicts OVERWRITE

# 6. Update node group (launches new nodes with new AMI, drains old ones)
aws eks update-nodegroup-version --cluster-name production-eks --nodegroup-name workers
```

### Pre-Upgrade Checklist
- [ ] Read Kubernetes changelog for deprecations/removals
- [ ] Test upgrade in dev/staging first
- [ ] Check all workloads use supported API versions
- [ ] Verify PodDisruptionBudgets are set (prevent all pods being evicted at once)
- [ ] Ensure enough cluster capacity for rolling node replacement
- [ ] Back up critical data (Velero)

### PodDisruptionBudget (PDB)
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2    # Always keep at least 2 pods running
  # OR: maxUnavailable: 1  # At most 1 pod can be down
  selector:
    matchLabels:
      app: my-app
```

PDBs prevent node drains from killing too many pods at once. Essential for upgrades.

## Backup & Restore (Velero)

### Install Velero
```bash
# Install Velero CLI
curl -L https://github.com/vmware-tanzu/velero/releases/latest/download/velero-linux-amd64.tar.gz | tar xz
sudo mv velero /usr/local/bin/

# Install Velero in cluster (with AWS plugin)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket my-velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero
```

### Backup Operations
```bash
# Full cluster backup
velero backup create full-backup --include-namespaces '*'

# Namespace backup
velero backup create app-backup --include-namespaces production

# Scheduled backup (daily)
velero schedule create daily-backup --schedule="0 2 * * *" --include-namespaces production --ttl 168h

# Check backup status
velero backup get
velero backup describe full-backup

# Restore
velero restore create --from-backup full-backup

# Restore specific namespace
velero restore create --from-backup full-backup --include-namespaces production
```

### What Velero Backs Up
- All Kubernetes resources (Deployments, Services, ConfigMaps, etc.)
- PersistentVolume data (via EBS snapshots)
- Cluster-scoped resources (ClusterRoles, Namespaces)

## Cost Optimization

### Right-Sizing with Karpenter
Karpenter automatically picks the cheapest instance that fits your workload. But you can optimize further:

```yaml
# NodePool with cost-optimized settings
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]  # Prefer spot (60-90% cheaper)
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r", "t"]  # Wide selection for best price
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s  # Aggressively consolidate
```

### Cost Strategies

| Strategy | Savings | Risk |
|----------|---------|------|
| Spot instances for stateless workloads | 60-90% | Interruption (2 min warning) |
| Right-size pods (VPA recommendations) | 20-40% | None |
| Karpenter consolidation | 15-30% | Brief pod rescheduling |
| Scale to zero (dev/staging off-hours) | 50-70% | Cold start time |
| Savings Plans / Reserved Instances | 30-60% | Commitment |

### Scale Down Non-Production
```bash
# Scale dev cluster to 0 nodes at night
# CronJob or external scheduler
eksctl scale nodegroup --cluster=dev-eks --name=workers --nodes=0 --nodes-min=0

# Or with Karpenter: set NodePool limits to 0
```

### Monitor Costs
```bash
# AWS Cost Explorer tags
# Tag all EKS resources with:
# - Environment: dev/staging/production
# - Team: platform/backend/frontend
# - Service: my-app

# Use kubecost or OpenCost for per-pod cost attribution
helm install kubecost kubecost/cost-analyzer --namespace kubecost --create-namespace
```

## Chaos Engineering

### Why?
Test that your system handles failures gracefully BEFORE they happen in production.

### Litmus Chaos
```bash
# Install Litmus
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm install litmus litmuschaos/litmus --namespace litmus --create-namespace
```

### Common Chaos Experiments

```yaml
# Pod kill — randomly kill pods
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: pod-kill-test
spec:
  appinfo:
    appns: default
    applabel: app=my-app
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "10"
            - name: FORCE
              value: "false"
```

### Manual Chaos Tests
```bash
# Kill random pods
kubectl delete pod -l app=my-app --field-selector=status.phase=Running | head -1

# Network partition (block traffic between services)
kubectl apply -f network-policy-block-all.yaml

# Node failure (cordon + drain)
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# DNS failure (scale CoreDNS to 0)
kubectl scale deployment coredns -n kube-system --replicas=0

# Resource exhaustion (deploy a memory hog)
kubectl run stress --image=polinux/stress -- stress --vm 1 --vm-bytes 512M
```

### What to Verify During Chaos
- Does the app stay available? (check HTTP responses)
- Do alerts fire correctly?
- Does auto-healing work? (pods restart, traffic reroutes)
- What's the recovery time?

## Troubleshooting Methodology

### The Debug Flow
```
1. What's the symptom? (pod not starting, service unreachable, high latency)
2. Where is the problem? (pod, node, network, DNS, storage)
3. What changed recently? (deployment, config change, upgrade)
4. Gather evidence (events, logs, metrics, describe)
5. Fix and verify
```

### Common Issues and Fixes

**Pod stuck in Pending:**
```bash
kubectl describe pod <name>  # Check Events section
# Common causes:
# - Insufficient resources (no node has enough CPU/memory)
# - Node selector/affinity doesn't match any node
# - PVC can't be bound (wrong StorageClass, no capacity)
```

**Pod in CrashLoopBackOff:**
```bash
kubectl logs <pod> --previous  # Logs from crashed container
kubectl describe pod <pod>     # Check exit code
# Common causes:
# - App crashes on startup (missing config, can't connect to DB)
# - OOMKilled (memory limit too low)
# - Liveness probe failing
```

**Pod in ImagePullBackOff:**
```bash
kubectl describe pod <pod>  # Check image name and pull errors
# Common causes:
# - Wrong image name/tag
# - ECR auth expired (node IAM role missing ecr:GetAuthorizationToken)
# - Private registry, no imagePullSecret
```

**Service not reachable:**
```bash
# Check endpoints exist
kubectl get endpoints <service-name>
# If empty: selector doesn't match any pod labels

# Check from inside cluster
kubectl run debug --image=busybox --rm -it -- wget -qO- http://<service>:<port>

# Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

**Node NotReady:**
```bash
kubectl describe node <node>  # Check Conditions
# SSH into node:
systemctl status kubelet
journalctl -u kubelet --since "5 minutes ago"
# Common causes:
# - kubelet crashed
# - Disk pressure
# - Memory pressure
# - Network issues (can't reach API server)
```

### Useful Debug Commands
```bash
# Ephemeral debug container (K8s 1.25+)
kubectl debug -it <pod> --image=busybox --target=<container>

# Node debug
kubectl debug node/<node-name> -it --image=ubuntu

# Network debug pod
kubectl run netdebug --image=nicolaka/netshoot --rm -it -- bash
# Inside: curl, dig, nslookup, tcpdump, iperf, etc.

# Check resource usage
kubectl top pods --sort-by=memory
kubectl top nodes

# Events (sorted by time)
kubectl get events --sort-by='.lastTimestamp' -A

# API server audit
kubectl get events --field-selector reason=FailedScheduling
```

## Labs

### Lab 12.1: Cluster Upgrade
1. Create a cluster on version 1.34
2. Deploy workloads with PDBs
3. Upgrade to 1.35 (control plane → add-ons → nodes)
4. Verify zero downtime during upgrade
5. Check all workloads are healthy after

### Lab 12.2: Backup and Restore
1. Install Velero
2. Deploy an app with a database (StatefulSet + PVC)
3. Insert data
4. Take a backup
5. Delete the namespace
6. Restore from backup — verify data is intact

### Lab 12.3: Cost Optimization
1. Install kubecost/OpenCost
2. Identify over-provisioned pods (VPA recommendations)
3. Configure Karpenter with spot instances
4. Set up consolidation
5. Compare costs before/after

### Lab 12.4: Chaos Testing
1. Deploy a multi-service app (frontend → backend → database)
2. Kill backend pods — verify frontend handles it gracefully
3. Drain a node — verify pods reschedule
4. Block network between services — verify timeouts work
5. Scale CoreDNS to 0 — observe and recover

### Lab 12.5: Troubleshooting Challenge
1. Deploy a broken app (intentionally misconfigured)
2. Diagnose: why is it not working?
3. Fix each issue using the debug methodology
4. Document what you found and how you fixed it

## Self-Test Questions

1. You need to upgrade EKS from 1.33 to 1.35. Can you skip 1.34? Why or why not?
2. What's the correct order for upgrading an EKS cluster? (Control plane, add-ons, nodes — which first?)
3. What's a PodDisruptionBudget? What happens during a node drain if a PDB says `minAvailable: 3` but you only have 3 replicas?
4. You upgrade your node group. How does EKS replace nodes without downtime? (Hint: what's the process?)
5. What does Velero back up? Does it back up the actual data in your EBS volumes or just the Kubernetes manifests?
6. You restore a Velero backup to a new cluster. The PVCs are restored but the pods can't mount them. What might be wrong?
7. Your EKS cluster costs $2000/month. Name 3 specific things you'd check to reduce costs.
8. Spot instances are 70% cheaper. Why don't you run EVERYTHING on spot? Name 3 workloads that should NOT use spot.
9. What's Karpenter consolidation? Give a scenario where it saves money.
10. You run a chaos experiment that kills 2 out of 3 backend pods. Your frontend shows errors. What's wrong with your setup?
11. A pod is stuck in `Pending` for 10 minutes. Walk me through your debugging steps (in order).
12. A pod is in `CrashLoopBackOff` but `kubectl logs` shows nothing. Where else do you look?
13. You notice a node is `NotReady`. What are the first 3 commands you run?
14. What's an ephemeral debug container? How is it different from `kubectl exec`?
15. Your app works in staging but not in production. Same image, same manifests. Name 5 things that could be different.

## Checklist Before Moving On

- [ ] Can perform EKS cluster upgrades (control plane + add-ons + nodes)
- [ ] Understand PodDisruptionBudgets and their role in upgrades
- [ ] Can install and use Velero for backup/restore
- [ ] Can set up scheduled backups
- [ ] Know cost optimization strategies (spot, right-sizing, consolidation)
- [ ] Can run chaos experiments (pod kill, node drain, network partition)
- [ ] Have a systematic troubleshooting methodology
- [ ] Can debug common pod issues (Pending, CrashLoopBackOff, ImagePullBackOff)
- [ ] Can use ephemeral debug containers
- [ ] Know how to check node health and kubelet status
