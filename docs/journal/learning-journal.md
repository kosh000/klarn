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
