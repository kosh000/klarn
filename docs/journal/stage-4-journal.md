# Stage 4: Workloads & Application Lifecycle — Learning Journal

## Rules
- Timestamped, append-only, corrections documented
- Every insight, every mistake, every fix
- Misconceptions documented with what was thought vs what is correct

---

## Progress Timeline

| Date | What was done | Status |
|------|--------------|--------|
| 2026-06-13 | Stage 4 started — steering file reviewed, journal initialized | ✅ |
| 2026-06-13 | Renamed `exec_01_cluster` → `exec_03_cluster` (aligns folder with stage 3) | ✅ |

---

## Detailed Entries

### 2026-06-13 — Stage 4 Kickoff

Stage 4 begins. Topics to cover:
1. Deployments (rolling updates, recreate strategy, rollbacks)
2. ConfigMaps (env vars, file mounts, reload behavior)
3. Secrets (base64, not encrypted, production alternatives)
4. Health Checks (liveness, readiness, startup probes)
5. Resource Management (requests vs limits, CPU throttling vs OOM kill)
6. Jobs and CronJobs
7. DaemonSets
8. Init Containers
9. Multi-Container Pod Patterns (sidecar, ambassador, adapter, native sidecars)
10. ResourceQuota & LimitRange

Following the session flow: concepts first (one at a time), then labs after all concepts are covered.
