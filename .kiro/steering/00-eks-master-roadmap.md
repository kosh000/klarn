---
inclusion: auto
description: EKS learning roadmap — stage map, progress tracking, philosophy, and documentation rules
---

# EKS Mastery Learning Roadmap

## Philosophy

Learn by doing first, then understanding why. Each stage follows:
1. **Do it** — get something working, see the result
2. **Understand it** — learn what happened under the hood
3. **Break it** — intentionally cause failures to learn troubleshooting
4. **Best practices** — learn the production-grade way

## Session Flow (STRICT)

Within each stage, follow this interaction pattern:

1. **Concepts first** — walk through all concepts/subjects in the stage one by one, explaining each with discussion
2. **Labs after** — only after all concepts are covered, move to hands-on labs
3. **One topic at a time** — present one concept, discuss it, answer questions, then move to the next
4. **Learner controls pace** — wait for the learner to signal readiness before advancing to the next topic
5. **No skipping ahead** — do not jump to labs or later topics until the current concept is understood

## Current Stack (2026)

| Component | Tool | Why |
|-----------|------|-----|
| **AWS Profile** | **`--profile eks-learning`** | **ALL AWS commands use this. Region: ap-south-1, Output: json** |
| Kubernetes | v1.35 on EKS (upstream at v1.36) | Latest stable on EKS |
| Cluster creation | eksctl → Terraform | Fast start, then production IaC |
| Node scaling | Karpenter | Default recommendation, replaces Cluster Autoscaler |
| Pod IAM | EKS Pod Identity | Simpler than IRSA, new default |
| GitOps | ArgoCD | Best UI, broad adoption, CNCF Graduated |
| Packaging | Helm + Kustomize | Helm for charts, Kustomize for overlays |
| Monitoring | Prometheus + Grafana | Kubernetes-native, portable |
| Logging | Fluent Bit → CloudWatch / Loki | AWS-native + Grafana integration |
| Tracing | OpenTelemetry → X-Ray / Jaeger | Vendor-neutral standard |
| Ingress | AWS Load Balancer Controller + Gateway API | EKS-native, ALB/NLB. Gateway API for portability |
| Ingress (on-prem) | MetalLB + Envoy Gateway | Bare-metal LB + Gateway API controller |
| Security policy | Pod Security Admission + Kyverno | Built-in + simple policy engine |
| Secrets | External Secrets Operator + AWS Secrets Manager | Production-grade |
| Container runtime | containerd | Default since K8s 1.24 (Docker shim removed) |

## Stage Map

### Stage 0: Prerequisites
Linux fundamentals, networking, YAML, AWS basics (IAM, VPC, EC2)

### Stage 1: Docker & Containers
Containers from scratch, Dockerfiles, images, registries, Docker Compose

### Stage 2: Kubernetes Core Concepts
Architecture, control plane, data plane, core objects, kubectl, local cluster (kind)

### Stage 3: EKS Cluster Setup (EC2 Managed Nodes)
eksctl cluster creation, deploy first workload, explore nodes, understand what was created

### Stage 4: Workloads & Application Lifecycle
Deployments, ConfigMaps, Secrets, probes, resource management, Jobs

### Stage 5: Networking & Ingress
Services, Gateway API, Ingress, ALB Controller, MetalLB, Envoy Gateway, ExternalDNS, CoreDNS, Network Policies

### Stage 6: Security
RBAC, Pod Identity, Pod Security Standards, Kyverno, secrets management

### Stage 7: Storage & Stateful Workloads
PV/PVC, EBS CSI, EFS CSI, StatefulSets, running databases on EKS

### Stage 8: Scheduling & Autoscaling
Affinity, taints, tolerations, HPA, VPA, Karpenter, spot instances

### Stage 9: Observability
Prometheus, Grafana, Fluent Bit, OpenTelemetry, alerting

### Stage 10: CI/CD & GitOps
Helm, Kustomize, ArgoCD, CI pipelines, progressive delivery

### Stage 11: Infrastructure as Code
Terraform for EKS, modules, state management, production cluster setup

### Stage 12: Production Operations
Upgrades, backup (Velero), cost optimization, chaos engineering, troubleshooting

### Stage 13: Advanced & Serverless
Fargate, EKS Auto Mode, Hybrid Nodes, service mesh, custom operators

## Progress Tracking

| Stage | Status | Started | Completed |
|-------|--------|---------|-----------|
| Stage 0: Prerequisites | ✅ Complete | 2026-05-26 | 2026-05-26 |
| Stage 1: Docker & Containers | ✅ Complete | 2026-05-26 | 2026-05-26 |
| Stage 2: Kubernetes Core Concepts | ✅ Complete | 2026-05-26 | 2026-05-26 |
| Stage 3: EKS Cluster Setup | 🟡 In progress | 2026-05-26 | — |
| Stage 4: Workloads & Application Lifecycle | ⬜ Not started | — | — |
| Stage 5: Networking & Ingress | ⬜ Not started | — | — |
| Stage 6: Security | ⬜ Not started | — | — |
| Stage 7: Storage & Stateful Workloads | ⬜ Not started | — | — |
| Stage 8: Scheduling & Autoscaling | ⬜ Not started | — | — |
| Stage 9: Observability | ⬜ Not started | — | — |
| Stage 10: CI/CD & GitOps | ⬜ Not started | — | — |
| Stage 11: Infrastructure as Code | ⬜ Not started | — | — |
| Stage 12: Production Operations | ⬜ Not started | — | — |
| Stage 13: Advanced & Serverless | ⬜ Not started | — | — |

## Documentation Rules (STRICT)

1. **Verify facts** — before documenting anything technical, look it up on the internet to confirm it's accurate and current (2026).
2. **Learning journal** — every session, document what was learned, what was attempted, what failed, and what was corrected.
   - **Main journal:** `docs/journal/learning-journal.md` — for planning sessions, cross-stage decisions, and meta-progress.
   - **Per-stage journals:** `docs/journal/stage-{N}-journal.md` (e.g., `stage-0-journal.md`, `stage-1-journal.md`, etc.) — for all learning entries that happen DURING that stage's implementation/labs.
   - **Rule:** When the learner is actively working on a stage, write entries to that stage's journal file. Create the file on first use. The main journal is for cross-cutting entries only (planning, roadmap changes, stage transitions).
3. **Timeline format** — entries are chronological with timestamps. Never delete old entries.
4. **Corrections** — if the learner had a wrong understanding, document:
   - What they thought (the misconception)
   - What is actually correct (with source/reason)
   - Why the misconception is wrong
5. **Document as much as possible** — err on the side of over-documenting. Every insight, every mistake, every fix.
6. **Append-only** — the journal is a living history. Old entries stay forever.
7. **Auto-assess every prompt** — for EVERY message the learner sends, assess whether it contains something documentable (a question asked, a concept discussed, a mistake made, a correction given, a decision taken, progress made). If yes, update the appropriate journal file directly and automatically. Do NOT ask permission — just do it. The learner is relying on this.
8. **Stage journal structure** — each per-stage journal follows this template:
   ```
   # Stage N: [Title] — Learning Journal
   
   ## Rules
   - Same as main journal (timestamped, append-only, corrections documented)
   
   ---
   
   ## Progress Timeline
   
   | Date | What was done | Status |
   |------|--------------|--------|
   | YYYY-MM-DD | Completed Lab X.1 | ✅ |
   | YYYY-MM-DD | Read section on Y | ✅ |
   | YYYY-MM-DD | Attempted Lab X.2, hit issue with Z | ⚠️ |
   
   ---
   
   ## Detailed Entries
   
   ### YYYY-MM-DD — [Session Topic]
   ...entries...
   ```
   - The **Progress Timeline** table is a quick-glance summary of everything done in that stage — labs completed, sections read, questions answered, issues hit.
   - The **Detailed Entries** section has the full narrative (what was learned, mistakes, corrections, insights).
   - Both are append-only. Update the progress table every session.

## Notes

- Add your own notes, links, and discoveries below as you progress
- Each stage has its own steering file and reference doc
- Labs are embedded in each stage's steering file

---

## Your Notes

- **AWS CLI Profile**: This project uses `--profile eks-learning` for all AWS commands. Region: `ap-south-1`, output: `json`. See `00-aws-profile.md` steering file for details.

---

## Referenced Files (auto-included as context)

#[[file:docs/journal/learning-journal.md]]
#[[file:docs/reference/00-learning-resources.md]]
