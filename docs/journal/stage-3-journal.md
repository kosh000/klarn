# Stage 3: EKS Cluster Setup — Learning Journal

## Rules
- Same as main journal (timestamped, append-only, corrections documented)

---

## Progress Timeline

| Date | What was done | Status |
|------|--------------|--------|
| 2026-05-26 | Stage 3 started — What is EKS, node types | ✅ |
| 2026-06-12 | Tools verified — eksctl 0.227.0, kubectl v1.36.1, helm v4.2.0 | ✅ |
| 2026-06-12 | Cluster created (t3a.medium, public subnets, no NAT) | ✅ |
| 2026-06-12 | Tags applied to cluster and stacks | ✅ |
| 2026-06-12 | Cluster verified — 2 nodes running, kube-system pods healthy | ✅ |
| 2026-06-12 | Explored nodes (public IPs, different AZs, containerd, AL2023) | ✅ |
| 2026-06-12 | Created hello-eks.yaml — paused, nodes scaling down | ⏸️ |
| 2026-06-13 | Resumed — nodes scaled back up (2 new nodes, NotReady → Ready) | ✅ |
| 2026-06-13 | hello-eks.yaml explained (Deployment + Service structure) | ✅ |
| 2026-06-13 | Deployed hello-eks — 3 pods + LoadBalancer service running | ✅ |
| 2026-06-13 | Verified end-to-end: curl → ELB → Service → nginx pod → response | ✅ |
| 2026-06-13 | Self-healing tested: deleted pod → auto-recreated (q94kb replaced 2hjc7) | ✅ |
| 2026-06-13 | Scaling tested: 3→5→3, pods distributed across both nodes | ✅ |
| 2026-06-13 | Note: learner prefers sequential flow — no skipping, go in order | ✅ |
| 2026-06-13 | Cordon tested: cordoned node 85-219, new pods only went to 28-100 | ✅ |
| 2026-06-13 | Deleted pod on cordoned node — replacement also landed on 28-100 | ✅ |
| 2026-06-13 | Drain tested: node drained, DaemonSet pods (aws-node, kube-proxy) ignored | ✅ |
| 2026-06-13 | Uncordoned node, scaled back to 3 replicas | ✅ |
| 2026-06-13 | Full node ops cycle understood: cordon → drain → maintain → uncordon | ✅ |
| 2026-06-13 | **Stage 3 complete** | ✅ |

---

## Detailed Entries

### 2026-05-26 — What is EKS + Node Types

(Entry below)

