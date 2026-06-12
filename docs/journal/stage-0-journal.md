# Stage 0: Prerequisites — Learning Journal

## Rules
- Same as main journal (timestamped, append-only, corrections documented)

---

## Progress Timeline

| Date | What was done | Status |
|------|--------------|--------|
| 2026-05-26 | Started Linux Fundamentals — Process Model concept | ✅ |
| 2026-05-26 | Misconception corrected: orphaning vs SIGKILL | ✅ |
| 2026-05-26 | Discussed zombie processes in depth | ✅ |
| 2026-05-26 | Filesystem & permissions concept | ✅ |
| 2026-05-26 | systemd & services concept | ✅ |
| 2026-05-26 | Environment variables, stdin/stdout/stderr, pipes concept | ✅ |
| 2026-05-26 | Linux Fundamentals section complete | ✅ |
| 2026-05-26 | Started Networking — IP addresses & CIDR | ✅ |
| 2026-05-26 | Networking — TCP vs UDP, ports, sockets | ✅ |
| 2026-05-26 | Networking — DNS | ✅ |
| 2026-05-26 | Noted: learner has Cloud Map / service discovery experience | ✅ |
| 2026-05-26 | Networking — Subnets, routing, NAT | ✅ |
| 2026-05-26 | Networking — Firewalls (Security Groups, NACLs, iptables) | ✅ |
| 2026-05-26 | Networking section complete | ✅ |
| 2026-05-26 | Started YAML section | ✅ |
| 2026-05-26 | YAML — full syntax covered (maps, lists, nesting, multi-line, anchors, types) | ✅ |
| 2026-05-26 | YAML section complete | ✅ |
| 2026-05-26 | Started AWS Basics section | ✅ |
| 2026-05-26 | AWS Basics — Regions/AZs, IAM, VPC, EC2, S3, CLI covered | ✅ |
| 2026-05-26 | AWS Basics section complete | ✅ |
| 2026-05-26 | Stage 0 concepts complete — moving to labs | ✅ |
| 2026-05-26 | Labs 0.1 & 0.2 skipped (Linux/Networking already demonstrated) | ✅ |
| 2026-05-26 | Lab 0.3 — YAML practice file created & validated | ✅ |
| 2026-05-26 | Lab 0.4 — AWS profile verified (eks-learning, ap-south-1, account 851060550361) | ✅ |
| 2026-05-26 | **Stage 0 complete** | ✅ |

---

## Detailed Entries

### 2026-05-26 — Linux Fundamentals: Process Model

**Topics covered:**
- What a process is (PID, parent/child, memory space)
- PID 1 and why it's special (kernel panic if it dies, signal handling in containers)
- Signals: SIGTERM (15) vs SIGKILL (9)
- Why SIGTERM matters for Kubernetes (graceful shutdown, 30s grace period)
- Parent/child relationships, orphans, zombies
- Why containers use tini/dumb-init

**Misconception corrected:**

| What learner thought | What is actually correct | Why the misconception is wrong |
|---------------------|------------------------|-------------------------------|
| Orphaning (children reparented to PID 1) only happens when parent is killed with `kill -9` | Orphaning happens whenever the parent exits for ANY reason — SIGKILL, SIGTERM, crash, normal exit | The kernel reparents based on "parent no longer exists", not based on how it died. SIGKILL just makes it more likely because the parent can't clean up children first, but the mechanism is signal-agnostic. |

