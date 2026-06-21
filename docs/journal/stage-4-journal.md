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
| 2026-06-13 | Deployments concept covered (rolling updates, rollbacks, recreate) | ✅ |
| 2026-06-13 | Hands-on: rollout undo --to-revision, observed revision consumption | ✅ |
| 2026-06-13 | ConfigMaps concept introduced (env vars, file mounts, reload problem) | ✅ |
| 2026-06-14 | ConfigMaps deep-dive restart with more examples, binaryData correction | ✅ |
| 2026-06-14 | Secrets concept covered (base64, types, RBAC, EKS encryption, production pattern) | ✅ |
| 2026-06-20 | Hands-on: Created and applied ConfigMap + Secret files in exec_04/ | ✅ |
| 2026-06-20 | Health Checks concept covered (liveness, readiness, startup probes) | ✅ |
| 2026-06-20 | Resource Management concept covered (requests/limits, QoS, throttle vs OOM) | ✅ |
| 2026-06-20 | Jobs & CronJobs concept covered (run-to-completion, cron schedule, concurrency) | ✅ |
| 2026-06-20 | DaemonSets concept covered (one-per-node, nodeSelector, production config, logging pipeline) | ✅ |
| 2026-06-20 | Init Containers concept covered (prerequisite tasks, sequential, shared volumes) | ✅ |
| 2026-06-20 | Multi-Container Pod Patterns covered (sidecar, ambassador, adapter, native sidecars) | ✅ |
| 2026-06-20 | ResourceQuota & LimitRange covered (namespace caps, per-container defaults) | ✅ |
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


### 2026-06-13 — Deployments: Rollback to Specific Revision

**Question asked:** What does `kubectl rollout undo deployment/my-app --to-revision=2` do exactly?

**Concept covered:** Revision history, how `--to-revision` targets a specific past state rather than just "previous."


**Hands-on:** Learner wants to try rolling updates and rollbacks on the live `hello-eks` deployment (currently nginx:1.26, 3 replicas) before moving to ConfigMaps.


**Progress:** Applied hello-eks with nginx:1.26 (revision 1 from file), then updated to nginx:1.25 via command (revision 2). Two revisions now exist.


**Observation:** After rolling back to revision 1, history shows revisions 2, 3, 4. Revision 1 is gone — it was consumed and became revision 4. Learner confirmed this behavior matches the explanation.


**Correction:** Actual sequence was nginx:1.27 → 1.26 → 1.25 → 1.27 (4 updates total). Learner then rolled back using `--to-revision`, resulting in revisions 2, 3, 4 in history.


**Observation:** `kubectl rollout undo` produced a warning about `last-applied-configuration` annotation not being updated. This is the tension between imperative rollbacks and declarative `kubectl apply` — the annotation tracks what was last applied from a file, and `undo` doesn't update it.


**Question asked:** Can the local YAML file be updated from the live cluster state (pull from kube to file)?

**Answer:** Yes, using `kubectl get deployment -o yaml`, but the output includes cluster-managed fields (status, metadata.uid, etc.) that you wouldn't want in your source file. Not a clean round-trip.


### 2026-06-13 — Session End (Break)

**Topics covered:**
1. Deployments — rolling updates, ReplicaSet chain, maxSurge/maxUnavailable, recreate strategy
2. Rollbacks — revision history, `--to-revision`, revision consumed on rollback, `last-applied-configuration` warning
3. ConfigMaps — concept introduced (env vars vs file mounts, reload problem, namespace-scoped)

**Hands-on completed:**
- Multiple image updates on hello-eks (1.27 → 1.26 → 1.25 → 1.27)
- Rollback with `--to-revision=2`, observed revision 1 disappearing from history
- Discussed imperative vs declarative tension, noisy `kubectl get -o yaml` output

**Next up when resuming:** Continue with ConfigMaps discussion/questions, then Secrets (topic 3).


**Action:** Deleted hello-eks LoadBalancer service (saves cost — no idle ELB running during break). Deployment still running.


**Action:** Scaled nodegroup `workers` to 0 nodes (cost saving during break). Control plane still running (EKS charges $0.10/hr for it). To resume: scale back up with `eksctl scale nodegroup --cluster=eks-learning --name=workers --nodes=2 --nodes-min=2 --profile eks-learning`.


### 2026-06-14 — Session Resumed

**Action:** Nodes scaled back up. Resuming from ConfigMaps (topic 2 introduced, no questions yet).


**Question asked:** How to try ConfigMaps hands-on?


**Request:** Restart ConfigMaps topic with deeper explanation and more examples.


**Question asked:** Why can't ConfigMaps be used for binary data?


**ConfigMaps topic complete.** Key correction: ConfigMaps CAN hold binary data via `binaryData` field — initial statement was oversimplified. Moving to Secrets.


### 2026-06-14 — Session End (Break)

**Topics covered today:**
- ConfigMaps — full restart with deeper examples (env vars, file mounts, reload problem, binaryData correction)
- Secrets — complete (base64 ≠ encryption, types, RBAC separation, EKS KMS encryption, production pattern preview)

**Next up when resuming:** Topic 4 — Health Checks (liveness, readiness, startup probes)


### 2026-06-14 — Hands-on: ConfigMap & Secret Files Created

**Learner created files in `exec_04/config/`:**
- `configmap.yml` — basic ConfigMap with LOG_LEVEL, DB_HOST, DB_PORT, ENABLE_CACHE
- `secret.yaml` — Secret using stringData with username/password
- `abc.conf` — test file for from-file ConfigMap creation


**Issue:** "Not Found" errors when trying to `kubectl get` the secret and configmap — learner hadn't applied them yet. Need to `kubectl apply -f` first before getting.


**Question asked:** Why can I see the Secret values in `kubectl get secret -o yaml`? (Seeing base64 values + plain text in `last-applied-configuration` annotation)

**Key observation:** The `last-applied-configuration` annotation stores the ORIGINAL input including `stringData` with plain-text values.


**Understanding confirmed:** Production uses external secrets stores (AWS Secrets Manager + External Secrets Operator), not YAML-committed secrets.


**Question asked:** How is a readiness probe actually set up in practice (the app-side implementation)?


**Health Checks topic complete.** Covered liveness, readiness, startup probes. Explained app-side readiness endpoint implementation (Node, Python, Spring Boot, nginx). Moving to Resource Management.


**Question asked:** Are Prometheus and VPA (Vertical Pod Autoscaler) covered in later stages?


**Resource Management topic complete.** Moving to Jobs and CronJobs.


**Question asked:** Are Jobs/CronJobs managed by a daemon?


**Jobs/CronJobs topic complete.** Clarified: managed by controllers in kube-controller-manager (control plane), not node-level daemons. Moving to DaemonSets.


**Questions asked:**
1. Does the node need to be created with the label for nodeSelector to work?
2. Requested a full best-practice production DaemonSet config.


**Question asked:** Explain the volumes section in the DaemonSet (hostPath and configMap volumes).


**Question asked:** If logs are on the node (hostPath), doesn't removing/replacing the node lose all that monitoring data?

**Key insight discussed:** hostPath is read-only source for Fluent Bit — it reads and SHIPS logs elsewhere. The node is not the storage destination.


**Question asked:** Confirming the logging pipeline — containers log to host, Fluent Bit reads from there, ships to visualizers?


**DaemonSets topic complete.** Covered: nodeSelector, labeling nodes, full production DaemonSet config, hostPath volumes, and the logging pipeline (container → node → Fluent Bit → storage → visualizer). Moving to Init Containers.


**Init Containers topic complete.** Learner understood the pattern: small utility images doing prerequisite tasks before the main app starts. Moving to Multi-Container Pod Patterns.


**Multi-Container Pod Patterns topic complete.** Covered sidecar, ambassador, adapter patterns + native sidecar containers (restartPolicy: Always). Moving to ResourceQuota & LimitRange.


**ResourceQuota & LimitRange topic complete.** All 10 concepts for Stage 4 are now covered. Ready to move to labs.

### 2026-06-20 — All Stage 4 Concepts Complete

**Summary of all topics covered:**
1. ✅ Deployments (rolling updates, recreate strategy, rollbacks)
2. ✅ ConfigMaps (env vars, file mounts, reload problem, binaryData)
3. ✅ Secrets (base64 ≠ encryption, types, RBAC, EKS KMS, production pattern)
4. ✅ Health Checks (liveness, readiness, startup probes — app-side implementation)
5. ✅ Resource Management (requests vs limits, CPU throttle vs OOM kill, QoS classes)
6. ✅ Jobs and CronJobs (run-to-completion, schedules, concurrency policies)
7. ✅ DaemonSets (one-per-node, nodeSelector, production config, logging pipeline)
8. ✅ Init Containers (prerequisite tasks, sequential execution, shared volumes)
9. ✅ Multi-Container Pod Patterns (sidecar, ambassador, adapter, native sidecars)
10. ✅ ResourceQuota & LimitRange (namespace caps, per-container defaults, multi-tenancy)

**Next:** Labs (hands-on exercises from the steering file).


### 2026-06-20 — Starting Labs

Beginning hands-on labs. Lab 4.1 (Rolling Updates) was partially done earlier. Starting fresh from Lab 4.1.


**Lab 4.1 progress:**
- Scaled to 4 replicas, updated to nginx:1.25
- Patched strategy to maxSurge:0, maxUnavailable:1
- Updated to nginx:1.26 — rolled out successfully (conservative, one-at-a-time)
- Learner confirmed understanding of `kubectl set image` syntax: deployment name + container name + new image

**Learner's correct understanding:** `kubectl set image deployment/hello-eks nginx=nginx:1.25` — targets the container named `nginx` inside deployment `hello-eks` and changes its image to `nginx:1.25`.


**Observation:** maxSurge:2, maxUnavailable:2 was very fast. 
**Question:** Does changing the strategy (via patch) require a rollout restart to take effect?


**Lab 4.1 complete.** Moving to Lab 4.2 (Configuration — ConfigMaps with deployment).


**Lab 4.2 complete.** Successfully demonstrated:
- Attached ConfigMap to deployment as env vars
- Verified env vars inside pod
- Changed ConfigMap (LOG_LEVEL info→debug)
- Confirmed pod still had OLD value (no auto-reload)
- Restarted deployment, confirmed NEW value (debug)


**Lab 4.3 complete.** Successfully demonstrated:
- Added tcpSocket liveness and readiness probes to deployment
- Stopped nginx inside a pod (nginx -s stop)
- Observed: readiness failed first (READY 0/1), then liveness failed → container restarted (RESTARTS 1)
- Pod came back to 1/1 Running after restart


**Lab 4.4 issue:** `kubectl run --limits` flag doesn't exist in current kubectl version. Need to use a YAML manifest instead for setting resource limits on pods.


**Lab 4.4 Step 1 complete:** memory-hog pod went OOMKilled within 6 seconds (tried to allocate 100Mi with 50Mi limit). `restartPolicy: Never` means it stays in OOMKilled state and doesn't retry.


**Lab 4.4 Step 2 complete:** cpu-hog ran for ~30s and exited as `Completed` — NOT killed. Demonstrated: CPU exceeding limits = throttled (slow), memory exceeding limits = OOMKilled (dead). Both pods cleaned up.


**Lab 4.5 progress:**
- Job success: ran and completed in 11 seconds. Noticed `SuccessCriteriaMet` status before `Complete`.
- Job fail: ran 4 pods (not 3). Question: why 4 attempts with backoffLimit:3?
- CronJob: working, fires every minute as expected.
- Question: How to check what CronJobs are doing? Use cases?
- Also: Python traceback in terminal was unrelated — caused by Ctrl+C interrupting an AWS CLI background process.


**Issue:** `kubectl logs -l every-minute-29700773-vlqvs` — wrong syntax. The `-l` flag expects a label selector, not a pod name. Correct: use the pod name directly.


**Observation:** Old CronJob pod (`every-minute-29700773-vlqvs`) was deleted by the `successfulJobsHistoryLimit: 3` setting — only the 3 most recent Jobs (and their pods) are kept. Older ones are garbage collected. Learner saw this in action.


**Lab 4.5 complete.** 
- Successfully checked CronJob logs with label selector: `kubectl logs -l job-name=every-minute-29700779`
- Observed `successfulJobsHistoryLimit: 3` in action — old pods/jobs cleaned up automatically (29700773, 29700774, 29700775 deleted as newer ones arrived)
- Noted: output shows `Tick at $(date)` literally — because single quotes in the YAML prevented shell expansion. Minor issue, concept still demonstrated.

**Correction needed:** The CronJob command used single quotes which prevented `$(date)` from expanding. Should use double quotes in the shell command for variable expansion. Not critical — the CronJob ran and completed as expected.

| 2026-06-21 | Lab 4.1: Rolling Updates (maxSurge/maxUnavailable variations) | ✅ |
| 2026-06-21 | Lab 4.2: ConfigMap env vars, reload problem demonstrated | ✅ |
| 2026-06-21 | Lab 4.3: Health Checks — stopped nginx, saw restart | ✅ |
| 2026-06-21 | Lab 4.4: OOMKilled (memory) vs Completed (CPU throttle) | ✅ |
| 2026-06-21 | Lab 4.5: Jobs, failing-job retries, CronJob every minute | ✅ |


### 2026-06-21 — Self-Test Questions Attempt

**Scoring: 13/16 correct, 2 partially correct, 1 minor misconception**

**Correct answers:** Q1 (5 pods), Q2 (recreate = downtime, mostly correct), Q3 (rollback concept understood), Q5 (base64 not encryption, RBAC), Q7 (liveness vs readiness), Q8 (DB check in liveness is bad), Q9 (memory OOM, CPU throttle), Q10 (requests=minimum/scheduler, limits=max), Q12 (DaemonSet vs Deployment distinction), Q13 (auto-scheduled), Q14 (patterns understood, names forgotten), Q16 (ResourceQuota vs LimitRange).

**Corrections needed:**
- Q4: Stated "only changes like images are picked up immediately" — correction: ALL pod template changes trigger a rollout (env vars, resources, volumes, not just images). ConfigMap VALUE changes don't trigger a rollout.
- Q6: Said "yes it can be mounted into a dir" — but the question was "can your app WRITE to it?" Answer is NO — ConfigMap mounts are read-only.
- Q11: Said "set to run 1 parallelly" — correct concept but didn't name the field: `concurrencyPolicy: Forbid`.
- Q15: Said "native sidecar fixes this by having restart counter" — close but imprecise. Native sidecars use `restartPolicy: Always` on init containers. The mechanism is lifecycle-aware shutdown (sidecar stops AFTER main exits), not a restart counter.


**Round 2 Self-Test Results: 4/4 ✅**
- Q1 (ConfigMap edit → no rollout): Correct. Understands why — pod template unchanged, only ConfigMap data changed. Restart needed.
- Q2 (Write to ConfigMap mount): Correct — read-only, write fails.
- Q3 (CronJob concurrency): Understood the concept (Forbid needed). Didn't directly answer "how many running simultaneously with Allow" but grasps the problem.
- Q4 (Native sidecar lifecycle): Correct — sidecar shuts down after main exits.

Requested review of multi-container pattern names with real-world analogies.


### 2026-06-21 — Stage 4 Complete ✅

**Final score:** 13/16 first attempt, 4/4 on round 2. Solid understanding across all topics.

**Stage 4 Checklist:**
- [x] Can perform rolling updates and rollbacks
- [x] Understand maxSurge and maxUnavailable
- [x] Can create and use ConfigMaps (env vars and file mounts)
- [x] Can create and use Secrets
- [x] Understand liveness vs readiness vs startup probes
- [x] Know when each probe type is appropriate
- [x] Can set resource requests and limits correctly
- [x] Understand CPU throttling vs memory OOM kill
- [x] Can create Jobs and CronJobs
- [x] Understand init containers and their use cases
- [x] Can create DaemonSets and understand when to use them
- [x] Know the difference between Deployment, StatefulSet, DaemonSet, and Job
- [x] Know the multi-container pod patterns (sidecar, ambassador, adapter)
- [x] Understand native sidecar containers (restartPolicy: Always on init containers)
- [x] Can create ResourceQuotas to cap namespace resource usage
- [x] Can create LimitRanges to set per-container defaults and bounds
- [x] Understand how ResourceQuota and LimitRange work together for multi-tenancy


### 2026-06-21 — Session Closed

**Final actions:** Nodes descaled to 0. Steering updated with teaching rules, cheatsheet generation rule, no-trim rule. Stage 4 cheatsheet and command log created. Ready for Stage 5 in new chat.
