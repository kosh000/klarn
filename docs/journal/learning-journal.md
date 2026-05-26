# EKS Learning Journal

## Rules
- Every entry is timestamped
- Document: what was learned, what went wrong, what was corrected, and WHY
- If a misconception is corrected, record the wrong understanding AND the correct one
- If something was looked up / verified on the internet, note the source
- Entries are append-only (never delete history)
- Kiro (teach) auto-assesses every prompt and documents if relevant

---

## Timeline

### 2026-05-25 — Initial Planning Session

**Context:** First session. Learner wants to master EKS from scratch — full Kubernetes journey from containers to production operations.

**Learner Profile:**
- Knows Docker and networking already (but wants to be treated as a complete beginner)
- Learning style: "do it first, then understand why it works"
- Prefers progressive learning: get something working → then peel back layers
- Trusts the teacher to make all structural/ordering decisions
- Wants maximum documentation of the journey

**Key Decisions Made:**

1. **Learning approach:** EC2 managed node groups FIRST, Fargate/Auto Mode LAST
   - Reason: EC2 shows the full picture (nodes, kubelet, scheduling). Fargate hides the worker plane. On-prem Kubernetes is always EC2-like. You need to understand what abstractions hide before using them.

2. **Progression philosophy:** Outside-in means "get it working, then understand the details"
   - NOT "learn theory first" — instead: deploy → see result → then learn mechanics

3. **Stage ordering (14 stages, 0-13):**
   - Prerequisites → Docker → K8s Core → EKS Setup → Workloads → Networking → Security → Storage → Autoscaling → Observability → CI/CD → Terraform → Production Ops → Advanced
   - Each stage: Do it → Understand it → Break it → Best practices

4. **Tool choices (2026 current):**
   - Karpenter over Cluster Autoscaler (default recommendation)
   - EKS Pod Identity over IRSA (simpler, new default)
   - ArgoCD for GitOps (best UI, broad adoption)
   - Helm + Kustomize together (packaging + overlays)
   - Prometheus + Grafana (K8s-native, portable)
   - AWS Load Balancer Controller (Ingress NGINX retiring 2026)

5. **Workspace structure:**
   - `.kiro/steering/` — master roadmap (always active) + per-stage files (manual inclusion)
   - `docs/reference/` — current links, versions, tool installs
   - `docs/journal/` — this file (timeline learning journal)

6. **Documentation rules:**
   - Verify everything on internet before documenting
   - Auto-document every relevant interaction
   - Timeline format, append-only
   - Record misconceptions WITH corrections AND reasons
   - Over-document rather than under-document

**Research Conducted:**
- Kubernetes latest: v1.36 "Haru" (April 2026) — GA features: User Namespaces, Fine-Grained Kubelet Auth, Mutating Admission Policies, SELinux Volume Labels
- EKS latest supported: v1.35
- EKS Auto Mode: fully managed compute using Karpenter + Bottlerocket
- EKS Pod Identity: new default over IRSA (universal trust policy, no OIDC setup)
- Karpenter: default node autoscaler, replaces Cluster Autoscaler
- EKS Hybrid Nodes Gateway: GA April 2026, simplifies on-prem networking
- Ingress NGINX: retiring upstream March 2026
- Helm 4: native server-side apply patterns
- ArgoCD & Flux: both CNCF Graduated, ArgoCD recommended for UI/onboarding

**Materials Created:**
- 15 steering files (master + 14 stages)
- 1 reference doc (tools, links, versions)
- 1 learning journal (this file)
- 195 self-test questions across all stages
- Labs in every stage (hands-on exercises + "break things" exercises)

**Learner's Explicit Instructions:**
- "I am not gonna adjust anything, you make the map for me"
- "You are my Teacher"
- "Document EVERYTHING, in timeline manner"
- "Auto-assess every prompt for documentation needs"
- "Verify facts on internet"
- "I am relying on this to you"

---

(Next entries will be added as learning progresses)

### 2026-05-26 — Roadmap Gap Analysis (roadmap.sh/kubernetes)

**Context:** Learner found the roadmap.sh/kubernetes roadmap and asked to cross-reference it against the existing 14-stage curriculum to identify missing topics.

**Source:** https://roadmap.sh/kubernetes (community Kubernetes learning roadmap, 2026)

**Gap Analysis Performed:**

Compared roadmap.sh topics against all 14 stages. Identified 13 missing/underrepresented topics.

**Topics Added (by stage):**

1. **Stage 2 — Kubernetes Alternatives** (Docker Swarm, Nomad, ECS, Mesos, OpenShift)
   - Why K8s won, when you might NOT use K8s
   - Added 2 self-test questions

2. **Stage 4 — DaemonSets** (full section)
   - What they are, when to use, manifest example
   - Difference from Deployments, targeting specific nodes, update strategies
   - Note: DaemonSets don't work on Fargate
   - Added 5 self-test questions, updated checklist

3. **Stage 5 — Load Balancing Concepts**
   - L4 vs L7 load balancing, session affinity, connection draining
   - ALB/NLB/Classic LB pricing (2026)
   - Cost math showing why Ingress matters

4. **Stage 6 — Admission Controllers & Webhooks** (full section)
   - Mutating vs Validating admission webhooks
   - How the API request flow works (auth → admission → etcd)
   - Real-world examples (Istio, Kyverno, Pod Security, Pod Identity)
   - Webhook configuration YAML example
   - K8s 1.36 CEL-based ValidatingAdmissionPolicy (no webhook server needed)
   - Updated checklist

5. **Stage 8 — Pod Priorities and Preemption** (full section)
   - PriorityClass resource, built-in system priorities
   - How preemption works, production priority hierarchy pattern
   
6. **Stage 8 — Evictions** (full section)
   - Four types: node-pressure, preemption, API-initiated, taint-based
   - Node-pressure thresholds (memory, disk, PIDs)
   - Taint-based eviction (automatic taints from node controller)
   - Graceful shutdown flow (SIGTERM → grace period → SIGKILL)
   - preStop hooks for LB draining
   - Updated checklist with 4 new items

7. **Stage 9 — Resource Health Monitoring** (full section)
   - Node health conditions and PromQL queries
   - PVC health monitoring
   - Cluster-level health queries
   - kubectl health check commands

8. **Stage 9 — Observability Engines / Managed Platforms** (full section)
   - Table of managed options with pricing: CloudWatch, AMP, AMG, Datadog, New Relic, Grafana Cloud, Dynatrace, Splunk
   - When to use managed vs self-hosted (decision criteria)
   - AWS-native observability stack pattern
   - Updated checklist with 2 new items

9. **Stage 10 — Blue-Green Deployments** (expanded from snippet to full section)
   - Blue-Green vs Canary comparison table
   - Full blue-green flow explanation
   - Argo Rollouts blue-green manifest with preview/active services
   - Promote/abort commands

10. **Stage 13 — Custom Schedulers and Extenders** (full section)
    - Scheduler extenders (webhook approach)
    - Scheduling framework plugins (Go plugins, phase hooks)
    - Running multiple schedulers, `schedulerName` field
    - Use cases: GPU-aware, data locality, license-aware, cost-aware

11. **Stage 13 — Kubernetes Extensions and APIs** (full section)
    - API aggregation layer
    - Extension points summary table (CRDs, webhooks, schedulers, API aggregation, CSI, CNI, device plugins)

12. **Stage 13 — Self-Managed Clusters** (full section)
    - What EKS hides vs what you'd do with kubeadm
    - kubeadm init/join flow
    - Node bootstrapping process
    - Why this matters for EKS users (debugging, security, interviews)

13. **Stage 13 — Multi-Cluster Management** (expanded significantly)
    - Multi-cluster tools table with cost (ArgoCD, KubeFed, Liqo, Admiralty, Rancher, Rafay, Crossplane)
    - ApplicationSet pattern for multi-cluster ArgoCD
    - Cross-cluster service discovery options (Cloud Map, Istio, DNS, Skupper)
    - Three multi-cluster patterns: Hub-and-Spoke, Active-Active, Specialized Clusters

**Paid Services Noted:**
- Datadog (~$15-23/host/month)
- New Relic (~$0.30/GB ingested, 100GB free)
- Grafana Cloud (per metric/log/trace)
- Dynatrace (~$21/host/month)
- Splunk (per GB ingested)
- Rafay (paid multi-cluster platform)
- Amazon Managed Prometheus (~$0.003/10K samples)
- Amazon Managed Grafana ($9/active editor/month)
- ALB (~$16-30/month), NLB (~$16-25/month)

**Total additions:** ~13 new sections, ~27 new self-test questions, ~11 new checklist items across 7 stage files.

---

(Next entries will be added as learning progresses)

### 2026-05-26 — AWS CLI Profile Setup

**Context:** Learner requested a dedicated AWS CLI profile for the project before starting hands-on labs.

**Decision Made:**
- Profile name: `eks-learning`
- Region: `ap-south-1` (Mumbai)
- Output format: `json`
- All AWS commands in this project must use `--profile eks-learning`

**What was done:**
- Created steering file `.kiro/steering/00-aws-profile.md` — documents the profile convention, setup instructions, and a rule for Kiro to always include the profile flag in generated commands
- Added note to master roadmap under "Your Notes"
- Profile does NOT exist yet on the system — learner needs to create it with `aws configure --profile eks-learning`

**Why this matters:**
- Keeps EKS learning isolated from any other AWS work
- Prevents accidental operations against wrong accounts/regions
- Makes commands reproducible (explicit profile = no ambient credential surprises)

---

(Next entries will be added as learning progresses)
