# Stage 4: Workloads & Application Lifecycle — Cheatsheet

## Deployments

| Term | Meaning |
|------|---------|
| **Deployment** | Manages ReplicaSets which manage Pods. Chain: Deployment → ReplicaSet → Pods |
| **Rolling Update** | Default strategy. Gradually replaces old pods with new ones. Zero downtime if probes configured. |
| **Recreate** | Kills ALL old pods first, then starts new ones. Causes downtime. Use when two versions can't coexist. |
| **maxSurge** | How many extra pods (above desired) can exist during update. Higher = faster but more resources. |
| **maxUnavailable** | How many pods (below desired) can be unavailable during update. Higher = faster but less capacity. |
| **Revision** | Each pod template change creates a numbered revision. `rollout undo` consumes the target revision and creates a new highest number. |
| **revisionHistoryLimit** | How many old ReplicaSets to keep (default: 10). Your rollback history. |

### Key Commands
```bash
kubectl set image deployment/<name> <container>=<image>    # Trigger update
kubectl rollout status deployment/<name>                   # Watch rollout
kubectl rollout history deployment/<name>                  # See revisions
kubectl rollout history deployment/<name> --revision=N     # Inspect specific revision
kubectl rollout undo deployment/<name>                     # Rollback to previous
kubectl rollout undo deployment/<name> --to-revision=N     # Rollback to specific
kubectl rollout restart deployment/<name>                  # Restart all pods (new revision)
kubectl patch deployment <name> -p '{"spec":{...}}'        # Patch in-place
```

### Gotchas (from labs)
- Only `.spec.template` changes trigger a rollout. Changing `replicas` or deployment labels does NOT.
- `rollout undo` produces a warning about `last-applied-configuration` if you used `kubectl apply`. The annotation gets stale. Fix: update your YAML file and re-apply.
- Changing strategy (maxSurge/maxUnavailable) does NOT trigger a rollout — it applies on the next update.
- `kubectl get -o yaml` output is too noisy to use as a source file (includes uid, resourceVersion, managedFields, status).

---

## ConfigMaps

| Term | Meaning |
|------|---------|
| **ConfigMap** | Key-value store for non-sensitive config. Namespace-scoped. Max 1 MiB. |
| **envFrom** | Inject ALL keys from a ConfigMap as environment variables |
| **configMapKeyRef** | Cherry-pick a specific key from a ConfigMap into one env var |
| **Volume mount** | Mount ConfigMap keys as files in a directory inside the container |
| **binaryData** | Field for storing binary content (base64-encoded). `data` is for UTF-8 strings only. |

### Key Commands
```bash
kubectl create configmap <name> --from-literal=KEY=VALUE
kubectl create configmap <name> --from-file=<filename>
kubectl get configmap <name> -o yaml
kubectl patch configmap <name> -p '{"data":{"KEY":"new-value"}}'
kubectl set env deployment/<name> --from=configmap/<cm-name>   # Attach to deployment
```

### Gotchas (from labs)
- **Env vars do NOT auto-reload.** You must `rollout restart` to pick up ConfigMap changes.
- **File mounts DO update** (~60s) but most apps don't watch for file changes anyway.
- Referencing a non-existent ConfigMap → pod stuck in `CreateContainerConfigError`.
- ConfigMap is namespace-scoped — pod and ConfigMap must be in same namespace.

---

## Secrets

| Term | Meaning |
|------|---------|
| **Secret** | Key-value store for sensitive data. Base64 encoded, NOT encrypted (unless etcd encryption enabled). |
| **stringData** | Convenience field — write plain text, K8s auto-encodes to base64. |
| **data** | The stored field — always base64. What you see in `kubectl get -o yaml`. |
| **Opaque** | Default Secret type. Generic key-value. |
| **kubernetes.io/tls** | TLS cert secret (requires `tls.crt` + `tls.key`). |
| **kubernetes.io/dockerconfigjson** | Image pull credentials for private registries. |

### Key Differences from ConfigMaps
- `kubectl describe secret` hides values (shows byte count only)
- Mounted as tmpfs (RAM) — never written to node disk
- RBAC can restrict Secret access separately from ConfigMaps
- **EKS encrypts Secrets at rest** with AWS KMS (more secure than self-managed clusters)

### Gotchas (from labs)
- `kubectl apply` with `stringData` stores plain-text values in the `last-applied-configuration` annotation. Security pitfall.
- Same reload problem as ConfigMaps — restart needed.
- Production: use External Secrets Operator + AWS Secrets Manager (Stage 6). Never commit secrets to git.

---

## Health Checks (Probes)

| Probe | Question it answers | On failure |
|-------|-------------------|------------|
| **Liveness** | "Is my process alive?" | Container RESTARTED |
| **Readiness** | "Can I serve traffic right now?" | Pod removed from Service endpoints (no traffic) |
| **Startup** | "Has my app finished booting?" | Disables liveness/readiness until it passes once |

| Probe Type | Mechanism | Success condition |
|-----------|-----------|-------------------|
| `httpGet` | HTTP request to path/port | Response 200-399 |
| `tcpSocket` | TCP connection attempt | Connection established |
| `exec` | Run command in container | Exit code 0 |

### Timing Parameters
```yaml
initialDelaySeconds: 10   # Wait before first check
periodSeconds: 10         # Check interval
timeoutSeconds: 1         # Each check timeout
failureThreshold: 3       # Consecutive failures before action
successThreshold: 1       # Successes needed to be "healthy"
```

### Golden Rules
1. **Liveness:** ONLY check your own process. NEVER check external dependencies (DB, Redis).
2. **Readiness:** CAN check dependencies. This is where you gate traffic.
3. **Startup:** Use for slow-starting apps to prevent liveness from killing them during boot.
4. **Always set readiness.** Without it, pods get traffic before they're ready.

---

## Resource Management

| Term | Meaning |
|------|---------|
| **requests** | Guaranteed minimum. Scheduler uses this for placement. "I need at least this much." |
| **limits** | Maximum allowed. Enforced at runtime by kernel cgroups. |
| **CPU throttle** | When container exceeds CPU limit — slowed down, NOT killed. |
| **OOM Kill** | When container exceeds memory limit — process killed, container restarted. |
| **QoS: Guaranteed** | requests = limits (both CPU and memory). Last to be evicted. |
| **QoS: Burstable** | requests < limits. Middle eviction priority. |
| **QoS: BestEffort** | No requests or limits. First to be evicted. |

### Units
- CPU: `1000m` = 1 core. `100m` = 10% of one core.
- Memory: `Mi` = mebibytes (1024-based). `Gi` = gibibytes.

### Best Practices
- Always set requests (scheduler needs them)
- Always set memory limits (prevent OOM of the whole node)
- CPU limits are debatable (throttling causes latency spikes)

---

## Jobs & CronJobs

| Term | Meaning |
|------|---------|
| **Job** | Run pods to completion. Retries on failure. |
| **backoffLimit** | Number of RETRIES (not total attempts). `backoffLimit: 3` = 4 total attempts. |
| **activeDeadlineSeconds** | Timeout for the whole Job. |
| **ttlSecondsAfterFinished** | Auto-delete Job N seconds after completion. |
| **CronJob** | Creates Jobs on a cron schedule. |
| **concurrencyPolicy** | `Allow` (overlap OK), `Forbid` (skip if previous running), `Replace` (kill previous). |
| **successfulJobsHistoryLimit** | How many completed Jobs to keep (default: 3). Older ones auto-deleted. |

### Key Commands
```bash
kubectl get jobs
kubectl get pods --selector=job-name=<job-name>
kubectl logs job/<job-name>
kubectl logs -l job-name=<job-name>              # By label
kubectl delete job <name>
kubectl delete cronjob <name>                    # Also deletes its Jobs
```

### Gotchas (from labs)
- `restartPolicy` must be `Never` or `OnFailure` for Jobs (not `Always`).
- Completed pods pile up. Use `successfulJobsHistoryLimit` or `ttlSecondsAfterFinished`.
- Managed by controllers in kube-controller-manager (control plane), not node daemons.
- Shell variable expansion in YAML: use double quotes, not single quotes.

---

## DaemonSets

| Term | Meaning |
|------|---------|
| **DaemonSet** | One pod per node, automatically. Scales with cluster size. |
| **nodeSelector** | Restrict DaemonSet to nodes with specific labels. |
| **tolerations** | Allow DaemonSet pods to run on tainted nodes (e.g., control plane). |
| **priorityClassName: system-node-critical** | Prevents eviction under node pressure. |
| **hostPath** | Mounts a directory from the NODE's filesystem into the container. |

### DaemonSet vs Deployment
- Deployment: you specify replica count. Pods may land on same node.
- DaemonSet: one per node, automatic. New node → pod auto-scheduled.

### Logging Pipeline (key insight from discussion)
```
App stdout → containerd → node file (/var/log/containers/) → Fluent Bit (hostPath) → CloudWatch/Loki → Grafana
```
- Node is the SOURCE, not the destination.
- Fluent Bit reads and SHIPS logs. If node dies, already-shipped logs are safe.
- This is why external log storage matters — pods and nodes are ephemeral.

### DaemonSets don't work on Fargate (no nodes).

---

## Init Containers

| Term | Meaning |
|------|---------|
| **Init container** | Runs BEFORE main container. Sequential. Must succeed (exit 0) for next to start. |
| **Shared volumes** | Init and main containers share volumes (same pod). Used to pass data between them. |

### Use Cases
- Wait for dependency (`until nc -z postgres 5432`)
- Download config from S3
- Set file permissions (`chown`)
- Run database migrations

### Rules
- Run in order, one at a time
- All must succeed before main container starts
- Can use different images than main container
- Don't have probes

---

## Multi-Container Pod Patterns

| Pattern | Purpose | Key trait | Example |
|---------|---------|-----------|---------|
| **Sidecar** | Extends main container's capability | Main doesn't know sidecar exists | Fluent Bit (log shipping), Envoy (service mesh) |
| **Ambassador** | Simplifies outbound connections | Main talks to localhost, ambassador proxies to external | Cloud SQL Proxy, PgBouncer |
| **Adapter** | Transforms output to standard format | Main outputs custom format, adapter converts | Prometheus exporter for legacy app |

### Native Sidecar Containers (K8s 1.29+ GA)
```yaml
initContainers:
  - name: log-shipper
    image: fluent-bit:3.1
    restartPolicy: Always    # THIS makes it a native sidecar
```
- Starts BEFORE main container
- Stays alive DURING main container's life
- Shuts down AFTER main container exits
- Fixes the Job completion problem (regular sidecars block Job completion because they never exit)

---

## ResourceQuota & LimitRange

| Term | Scope | Purpose |
|------|-------|---------|
| **ResourceQuota** | Namespace-level | Total cap on resources for the whole namespace (CPU, memory, pod count) |
| **LimitRange** | Per-container/pod | Defaults, min/max bounds for individual containers |

### Key Rules
- When ResourceQuota requires CPU/memory, ALL pods MUST specify requests/limits (rejected otherwise)
- LimitRange provides defaults so pods without explicit resources still pass quota enforcement
- ResourceQuota tracks **requests**, not actual usage — idle pods still count against quota

---

## Quick Reference: What Triggers a Rollout?

| Change | Triggers rollout? |
|--------|------------------|
| Image change | ✅ Yes |
| Env var change (in pod template) | ✅ Yes |
| Resource requests/limits change | ✅ Yes |
| Volume mount change | ✅ Yes |
| ConfigMap DATA change (referenced by name) | ❌ No — restart needed |
| Secret DATA change (referenced by name) | ❌ No — restart needed |
| Replica count change | ❌ No |
| Strategy change (maxSurge etc) | ❌ No |
| Labels on Deployment itself | ❌ No |
