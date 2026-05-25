---
inclusion: manual
---

# Stage 4: Workloads & Application Lifecycle

## Goal
Master deploying real applications: rolling updates, configuration management, health checks, resource management, and job scheduling.

## Deployments Deep Dive

### Rolling Updates
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods ABOVE desired during update
      maxUnavailable: 1  # Max pods BELOW desired during update
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: my-app:v1
```

**How rolling update works:**
1. New ReplicaSet created with new image
2. New pods start (up to maxSurge extra)
3. Old pods terminate (up to maxUnavailable at a time)
4. Repeats until all pods are new version
5. Zero downtime if health checks are configured properly

```bash
# Trigger update by changing image
kubectl set image deployment/my-app my-app=my-app:v2

# Watch the rollout
kubectl rollout status deployment/my-app

# See history
kubectl rollout history deployment/my-app

# Rollback to previous version
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout undo deployment/my-app --to-revision=2
```

### Recreate Strategy
```yaml
strategy:
  type: Recreate  # Kill all old pods, then start new ones. Causes downtime.
```
Use only when you can't run two versions simultaneously (e.g., database migrations).

## ConfigMaps

### As Environment Variables
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"
  FEATURE_NEW_UI: "true"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:v1
          envFrom:
            - configMapRef:
                name: app-config
          # Or individual keys:
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DATABASE_HOST
```

### As Mounted Files
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend:8080;
      }
    }
---
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
  volumes:
    - name: config-volume
      configMap:
        name: nginx-config
```

**Important:** Changing a ConfigMap does NOT automatically restart pods. You need to either:
- Restart the deployment: `kubectl rollout restart deployment/my-app`
- Use a hash annotation pattern (Helm does this automatically)

## Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: cG9zdGdyZXM=      # base64 encoded "postgres"
  password: c3VwZXJzZWNyZXQ=  # base64 encoded "supersecret"
---
# Or use stringData (plain text, Kubernetes encodes it)
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: postgres
  password: supersecret
```

```bash
# Create secret from command line
kubectl create secret generic db-credentials \
  --from-literal=username=postgres \
  --from-literal=password=supersecret

# Encode/decode base64
echo -n "postgres" | base64          # cG9zdGdyZXM=
echo "cG9zdGdyZXM=" | base64 -d     # postgres
```

**Using in pods:**
```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: username
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

**Warning:** Kubernetes Secrets are base64 encoded, NOT encrypted. Anyone with API access can read them. For production, use External Secrets Operator + AWS Secrets Manager (Stage 6).

## Health Checks (Probes)

### Liveness Probe
"Is the container alive?" If it fails, kubelet RESTARTS the container.

### Readiness Probe
"Is the container ready to receive traffic?" If it fails, pod is removed from Service endpoints (no traffic sent to it).

### Startup Probe
"Has the container finished starting?" Disables liveness/readiness checks until it passes. For slow-starting apps.

```yaml
spec:
  containers:
    - name: my-app
      image: my-app:v1
      ports:
        - containerPort: 8080

      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10   # Wait before first check
        periodSeconds: 10         # Check every 10s
        failureThreshold: 3       # Restart after 3 failures

      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        failureThreshold: 3       # Remove from service after 3 failures

      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30      # 30 * 10s = 5 minutes to start
        periodSeconds: 10
```

**Probe types:**
- `httpGet`: HTTP request, success = 200-399
- `tcpSocket`: TCP connection attempt
- `exec`: Run a command, success = exit code 0

**Common mistake:** Making liveness probe check dependencies (database). If DB is down, all pods restart in a loop. Liveness should only check "is MY process healthy?"

## Resource Management

```yaml
resources:
  requests:
    cpu: 100m       # 100 millicores = 0.1 CPU core
    memory: 128Mi   # 128 mebibytes
  limits:
    cpu: 500m       # Max 0.5 CPU core
    memory: 256Mi   # Max 256 MiB — OOM killed if exceeded
```

**Requests vs Limits:**
- **Requests**: guaranteed minimum. Scheduler uses this to place pods. "I need at least this much."
- **Limits**: maximum allowed. Container is throttled (CPU) or killed (memory) if exceeded.

**CPU units:**
- 1 CPU = 1000m (millicores)
- 100m = 10% of one core
- CPU limits cause throttling (not killing)

**Memory units:**
- Mi = mebibytes (1024-based)
- Gi = gibibytes
- Memory limits cause OOM kill (container restarts)

**Best practice:**
- Always set requests (for proper scheduling)
- Set memory limits (prevent runaway memory)
- CPU limits are debatable — some teams skip them to avoid throttling

## Jobs and CronJobs

### Job (run once to completion)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3          # Retry up to 3 times on failure
  activeDeadlineSeconds: 300  # Timeout after 5 minutes
  template:
    spec:
      restartPolicy: Never  # Don't restart, let Job controller handle retries
      containers:
        - name: migrate
          image: my-app:v1
          command: ["npm", "run", "migrate"]
```

### CronJob (scheduled)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-job
spec:
  schedule: "0 2 * * *"   # Every day at 2 AM (cron syntax)
  concurrencyPolicy: Forbid  # Don't run if previous is still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: my-app:v1
              command: ["npm", "run", "cleanup"]
```

## Init Containers

Run BEFORE the main container starts. Use for setup tasks.

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting for db; sleep 2; done']
    - name: run-migrations
      image: my-app:v1
      command: ['npm', 'run', 'migrate']
  containers:
    - name: my-app
      image: my-app:v1
```

Init containers run sequentially. Main container only starts after ALL init containers succeed.

## Labs

### Lab 4.1: Rolling Updates
1. Deploy an app with 4 replicas
2. Update the image tag
3. Watch the rolling update: `kubectl get pods -w`
4. Check rollout history
5. Rollback to previous version
6. Try with `maxSurge: 0, maxUnavailable: 1` — observe the difference

### Lab 4.2: Configuration
1. Create a ConfigMap with app settings
2. Mount it as environment variables in your deployment
3. Change a value in the ConfigMap
4. Verify the pod still has the OLD value (it doesn't auto-reload)
5. Restart the deployment, verify new value

### Lab 4.3: Health Checks
1. Deploy an app with liveness and readiness probes
2. Make the readiness probe fail (e.g., delete the /ready endpoint)
3. Observe: pod stays running but gets no traffic
4. Make the liveness probe fail
5. Observe: pod gets restarted

### Lab 4.4: Resource Limits
1. Deploy a pod with `memory limit: 50Mi`
2. Run a memory-hungry process inside it
3. Watch it get OOM killed: `kubectl describe pod <name>` → look for OOMKilled
4. Deploy with `cpu limit: 50m`
5. Run a CPU-intensive task, observe throttling (slower, not killed)

### Lab 4.5: Jobs
1. Create a Job that runs a simple script and exits
2. Verify it completed: `kubectl get jobs`
3. Create a Job that fails — observe retries (backoffLimit)
4. Create a CronJob that runs every minute
5. Watch it create Jobs on schedule

## Self-Test Questions

1. You set `maxSurge: 1` and `maxUnavailable: 0` with 4 replicas. During a rolling update, what's the maximum number of pods running at any point?
2. You update a ConfigMap. Do running pods automatically get the new values? How do you make them pick up changes?
3. What's the difference between a liveness probe and a readiness probe? What happens when each fails?
4. Your liveness probe checks if the database is reachable. Why is this a terrible idea?
5. You set `memory limit: 256Mi` and your app uses 300Mi. What happens? What about if CPU limit is 500m and app tries to use 1000m?
6. What's the difference between resource `requests` and `limits`? Which one does the scheduler use?
7. You set `requests: cpu: 100m`. What does `100m` mean in plain English?
8. A pod has `restartPolicy: Always`. The container exits with code 0 (success). Does it restart?
9. What's an init container? Give a real-world use case.
10. You have a CronJob that takes 10 minutes to run, but it's scheduled every 5 minutes. What happens? How do you prevent overlap?
11. Your Deployment has `strategy: Recreate`. What happens during an update? Is there downtime?
12. You run `kubectl rollout undo deployment/my-app`. What actually happens under the hood?
13. A Secret is "base64 encoded." Is it encrypted? Can someone with kubectl access read it?
14. You mount a ConfigMap as a file at `/etc/config/app.conf`. Can your app write to that file?
15. What's the difference between `kubectl set image` and editing the Deployment YAML and applying it?

## Checklist Before Moving On

- [ ] Can perform rolling updates and rollbacks
- [ ] Understand maxSurge and maxUnavailable
- [ ] Can create and use ConfigMaps (env vars and file mounts)
- [ ] Can create and use Secrets
- [ ] Understand liveness vs readiness vs startup probes
- [ ] Know when each probe type is appropriate
- [ ] Can set resource requests and limits correctly
- [ ] Understand CPU throttling vs memory OOM kill
- [ ] Can create Jobs and CronJobs
- [ ] Understand init containers and their use cases
