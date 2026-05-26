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

### 2026-05-26 — Stage 0 Begins + Session Flow Decision

**Context:** Learner starting Stage 0 (Prerequisites). Before diving in, established the preferred learning flow.

**Decision Made:**
- Learning flow: **Concepts first, then labs** (one topic at a time)
- Added to master roadmap steering as "Session Flow (STRICT)" rule
- This overrides the original "do it first" philosophy for the teaching interaction — concepts are discussed first, then practiced in labs after

**What was done:**
- Updated `00-eks-master-roadmap.md` with Session Flow section
- Stage 0 officially started

---

### 2026-05-26 — Gap Fix: App Exposure for On-Prem, Bare-Metal, and Local Clusters

**Context:** Learner identified that the curriculum only covered app exposure for EKS (AWS Load Balancer Controller + Ingress) but missed on-prem/bare-metal and local (kind) exposure patterns. This is a best-practice gap — a complete Kubernetes education must cover how to expose apps regardless of where the cluster runs.

**Research Conducted:**
- Ingress NGINX officially retired March 2026 (archived, no security patches)
- Gateway API is the official successor to Ingress API (GA since K8s 1.29, core resources stable)
- AWS Load Balancer Controller v3+ supports Gateway API in GA (announced early 2026)
- Envoy Gateway is the recommended on-prem/bare-metal Gateway API implementation
- MetalLB remains the standard for bare-metal LoadBalancer IP assignment (L2/BGP modes)
- Cilium also offers built-in L2/BGP LB as an alternative to MetalLB
- kind requires extraPortMappings at cluster creation time for NodePort access from host

**Sources:**
- https://aws.amazon.com/blogs/networking-and-content-delivery/aws-load-balancer-controller-adds-general-availability-support-for-kubernetes-gateway-api/
- https://gateway.envoyproxy.io/ (quickstart recommends MetalLB for bare-metal)
- https://metallb.io/
- https://github.com/kubernetes/ingress-nginx (archived March 24, 2026)
- https://tasrieit.com/blog/migrate-nginx-ingress-to-envoy-gateway-complete-guide

**What was added:**

1. **Stage 5 — "Exposing Apps: The Full Picture" section** (new)
   - Comparison table: EKS vs on-prem vs local exposure stacks
   - kubectl port-forward explanation
   - kind extraPortMappings with config example
   - MetalLB full section (install, IP pool config, L2 vs BGP comparison)
   - Envoy Gateway introduction for on-prem
   - Note about Cilium LB as MetalLB alternative

2. **Stage 5 — "Gateway API" section** (new, major)
   - Why Gateway API replaces Ingress (limitations of annotations-based config)
   - Ingress vs Gateway API comparison table
   - Three-layer model (GatewayClass → Gateway → HTTPRoute)
   - Full Gateway API example for EKS (AWS LB Controller v3+)
   - Full Gateway API example for bare metal (Envoy Gateway)
   - Traffic splitting example (native, no Istio needed)
   - Decision table: when to use Ingress vs Gateway API
   - Full exposure stack comparison table (EKS vs on-prem vs local)

3. **Stage 5 — Labs expanded**
   - Lab 5.3: Gateway API on EKS (new)
   - Lab 5.4: MetalLB + Envoy Gateway on kind (new)
   - Lab 5.7: Break Things expanded with on-prem failure scenarios

4. **Stage 5 — Self-test questions expanded**
   - Added 10 new questions (16-25) covering MetalLB, Gateway API, on-prem exposure, kind access, Ingress NGINX retirement

5. **Stage 5 — Checklist expanded**
   - Added 6 new items covering Gateway API, MetalLB, Envoy Gateway, local dev access

6. **Stage 2 — Local cluster access section** (new)
   - Three options for accessing services from kind: port-forward, extraPortMappings, MetalLB
   - Config examples for each

7. **Stage 13 — Hybrid Nodes ingress patterns** (new)
   - Three patterns: Cloud Ingress → on-prem pods, fully local ingress, split ingress
   - Control plane disconnection behavior for data-path

8. **Master roadmap — Current Stack table updated**
   - Added Gateway API and on-prem ingress row (MetalLB + Envoy Gateway)
   - Updated Stage 5 description

9. **Learning resources — updated**
   - Added Gateway API, Envoy Gateway, MetalLB docs links
   - Added version table entries (Gateway API v1.2+, Envoy Gateway 1.8+, MetalLB 0.14+, AWS LB Controller 3.x)
   - Updated "Key 2026 Changes" with Gateway API GA and Ingress NGINX retirement details

**Key insight documented:** The Gateway API's portability is the real win — an HTTPRoute written for EKS works unchanged on bare metal with Envoy Gateway. Only the GatewayClass and Gateway (infra layer) change between environments. App developers write the same routing rules everywhere.

---

### 2026-05-26 — Gap Fix #2: Missing Core K8s Concepts (CKA/CKAD Coverage)

**Context:** After fixing the app exposure gap, performed a comprehensive cross-reference of the entire 14-stage curriculum against CKA/CKAD exam domains, roadmap.sh/kubernetes, and 2026 best practices. Identified 7 additional missing topics.

**Method:** Extracted every concept from all 15 steering files, then searched for gaps against:
- CKA exam domains (cluster architecture 25%, workloads 15%, services/networking 20%, storage 10%, troubleshooting 30%)
- CKAD exam domains (app design 20%, deployment 20%, environment/config/security 25%, services/networking 20%)
- roadmap.sh/kubernetes community roadmap
- Industry best practices for production Kubernetes

**Gaps Found and Fixed:**

1. **Multi-container pod patterns** (Stage 4) — HIGH priority
   - Sidecar pattern (log shipping, metrics, TLS proxy)
   - Ambassador pattern (database proxy, API gateway)
   - Adapter pattern (log/metrics format conversion)
   - Full YAML examples for each pattern
   - Native Sidecar Containers API (K8s 1.28+, restartPolicy: Always)
   - Why native sidecars fix the Job completion problem

2. **ResourceQuota & LimitRange** (Stage 4) — HIGH priority
   - ResourceQuota: namespace-level caps (CPU, memory, pod count, PVC count)
   - LimitRange: per-container defaults, min/max bounds
   - How they work together for multi-tenancy
   - Production pattern example (team-a vs team-b quotas)
   - Note: when ResourceQuota is set, ALL pods must specify requests/limits

3. **ServiceAccount token security** (Stage 6) — MEDIUM priority
   - automountServiceAccountToken: false (disable for most pods)
   - Projected volumes with short-lived tokens (expirationSeconds)
   - Default token vs projected token security comparison
   - When you DO need API access (operators, controllers)

4. **Container image scanning with Trivy** (Stage 6) — MEDIUM priority
   - CLI usage (scan local, remote, severity filter)
   - CI pipeline integration (GitHub Actions example)
   - Cluster enforcement with Kyverno (restrict to ECR images)
   - Best practices (minimal base images, pin digests, rebuild regularly)

5. **cert-manager** (Stage 6) — MEDIUM priority
   - Installation (kubectl apply or Helm)
   - ClusterIssuer with Let's Encrypt (HTTP-01 and DNS-01 solvers)
   - Certificate resource (auto-renewal, secretName)
   - Decision table: ACM (EKS) vs cert-manager (on-prem) vs enterprise CAs

6. **etcd backup/restore** (Stage 13) — LOW priority (EKS handles it)
   - What etcd stores (ALL cluster state)
   - etcdctl snapshot save/restore commands
   - Why it matters for EKS users (CKA exam, hybrid clusters, debugging understanding)

7. **Native Sidecar Containers API** (Stage 4) — LOW priority
   - K8s 1.28+ feature (GA in 1.29)
   - restartPolicy: Always on initContainers
   - Solves Job completion problem with sidecars
   - Proper startup/shutdown ordering

**Self-test questions added:** 5 new (Stage 4) + 5 new (Stage 6) = 10 total
**Checklist items added:** 5 new (Stage 4) + 5 new (Stage 6) = 10 total
**Learning resources updated:** cert-manager docs, Trivy docs, version entries

**What was NOT added (intentionally):**
- etcd backup is covered conceptually but not as a lab (EKS handles it, and we don't have a self-managed cluster)
- OPA/Gatekeeper not added (Kyverno is the chosen tool, OPA is more complex and less recommended for beginners)
- Pod Security Policies not added (removed in K8s 1.25, replaced by Pod Security Admission which is already covered)

---
