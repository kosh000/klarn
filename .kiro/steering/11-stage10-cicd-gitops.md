---
inclusion: manual
---

# Stage 10: CI/CD & GitOps

## Goal
Automate the entire path from code commit to production deployment. No manual kubectl commands in production.

## The Pipeline

```
Code Push → CI (build, test, scan) → Image Push → Manifest Update → GitOps Sync → Cluster
```

## Helm (Package Manager)

### What is a Helm Chart?
A package of Kubernetes manifests with templating. Like apt/yum but for Kubernetes apps.

```
my-chart/
├── Chart.yaml          # Chart metadata (name, version)
├── values.yaml         # Default configuration values
├── templates/
│   ├── deployment.yaml # Templated manifests
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl    # Template helpers
│   └── NOTES.txt       # Post-install message
└── charts/             # Dependencies
```

### Chart.yaml
```yaml
apiVersion: v2
name: my-app
description: My application Helm chart
version: 1.0.0        # Chart version
appVersion: "2.1.0"   # App version
```

### values.yaml
```yaml
replicaCount: 3
image:
  repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app
  tag: "v2.1.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: api.example.com

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

env:
  LOG_LEVEL: info
  DATABASE_HOST: postgres.default.svc.cluster.local
```

### Templated Deployment
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
```

### Helm Commands
```bash
# Install a chart
helm install my-release ./my-chart -f values-prod.yaml

# Upgrade (apply changes)
helm upgrade my-release ./my-chart -f values-prod.yaml

# Install or upgrade
helm upgrade --install my-release ./my-chart -f values-prod.yaml

# Rollback
helm rollback my-release 1

# List releases
helm list

# See what would be applied (dry run)
helm template my-release ./my-chart -f values-prod.yaml

# Uninstall
helm uninstall my-release
```

### Using Third-Party Charts
```bash
# Add a repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# Search
helm search repo postgresql

# Install with custom values
helm install my-db bitnami/postgresql \
  --set auth.postgresPassword=secret \
  --set primary.persistence.size=20Gi
```

## Kustomize (Configuration Overlays)

No templating — just patches on top of base YAML.

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patch-replicas.yaml
    └── production/
        ├── kustomization.yaml
        ├── patch-replicas.yaml
        └── patch-resources.yaml
```

### Base
```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

### Overlay
```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: production
namePrefix: prod-
patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml
images:
  - name: my-app
    newName: 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app
    newTag: v2.1.0
```

```yaml
# overlays/production/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
```

```bash
# Preview what would be applied
kubectl kustomize overlays/production

# Apply
kubectl apply -k overlays/production
```

### Helm + Kustomize Together
Use Helm for third-party charts, Kustomize for your own apps and environment overlays.

## ArgoCD (GitOps)

### What is GitOps?
- Git repository is the single source of truth for cluster state
- Changes to the cluster happen ONLY through git commits
- ArgoCD watches git and automatically syncs cluster to match
- If someone manually changes something in the cluster, ArgoCD reverts it (drift detection)

### Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
```

### ArgoCD Application
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app-manifests.git
    targetRevision: main
    path: overlays/production    # Kustomize path
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true       # Delete resources removed from git
      selfHeal: true    # Revert manual changes in cluster
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### ArgoCD with Helm
```yaml
spec:
  source:
    repoURL: https://github.com/myorg/my-app-charts.git
    targetRevision: main
    path: charts/my-app
    helm:
      valueFiles:
        - values-production.yaml
```

### The GitOps Workflow
```
1. Developer pushes code → triggers CI pipeline
2. CI: build image, run tests, push to ECR with tag (e.g., v2.1.0)
3. CI: update image tag in git manifests repo (or values.yaml)
4. ArgoCD detects git change
5. ArgoCD syncs: applies new manifests to cluster
6. Rolling update happens automatically
7. If something breaks: revert the git commit → ArgoCD syncs back
```

### App of Apps Pattern
Manage multiple applications with one parent:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/argocd-apps.git
    path: apps
  destination:
    server: https://kubernetes.default.svc
```

## CI Pipeline (GitHub Actions Example)

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push image
        run: |
          IMAGE_TAG=${{ github.sha }}
          docker build -t $ECR_REPO:$IMAGE_TAG .
          docker push $ECR_REPO:$IMAGE_TAG

      - name: Update manifest
        run: |
          # Update image tag in manifests repo
          cd manifests
          kustomize edit set image my-app=$ECR_REPO:${{ github.sha }}
          git commit -am "Update image to ${{ github.sha }}"
          git push
```

## Progressive Delivery

### Canary Deployment (with Argo Rollouts)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10      # Send 10% traffic to new version
        - pause: {duration: 5m}
        - setWeight: 30
        - pause: {duration: 5m}
        - setWeight: 60
        - pause: {duration: 5m}
        - setWeight: 100     # Full rollout
      canaryMetadata:
        labels:
          version: canary
```

### Blue-Green Deployment
```yaml
strategy:
  blueGreen:
    activeService: my-app-active
    previewService: my-app-preview
    autoPromotionEnabled: false  # Manual promotion
    prePromotionAnalysis:
      templates:
        - templateName: success-rate
```

## Labs

### Lab 10.1: Create a Helm Chart
1. Create a Helm chart for your app
2. Define values for dev and production
3. Install in dev namespace with dev values
4. Install in production namespace with production values
5. Upgrade with a new image tag

### Lab 10.2: Kustomize Overlays
1. Create base manifests for your app
2. Create overlays for dev, staging, production
3. Each overlay: different replicas, resources, image tags
4. Apply each overlay, verify differences

### Lab 10.3: ArgoCD Setup
1. Install ArgoCD
2. Create a git repo with your manifests
3. Create an ArgoCD Application pointing to it
4. Push a change to git — watch ArgoCD sync
5. Manually change something in cluster — watch ArgoCD revert it

### Lab 10.4: Full CI/CD Pipeline
1. Set up GitHub Actions (or similar) to build and push images
2. Pipeline updates image tag in manifests repo
3. ArgoCD picks up the change and deploys
4. Verify end-to-end: code push → running in cluster

### Lab 10.5: Canary Deployment
1. Install Argo Rollouts
2. Deploy with canary strategy (10% → 30% → 100%)
3. Deploy a new version, watch traffic shift gradually
4. Introduce a bug in new version — observe and rollback

## Self-Test Questions

1. What's the difference between Helm and Kustomize? When would you use each? When would you use both together?
2. In a Helm chart, what's `values.yaml`? What happens if you pass `-f values-prod.yaml` — does it override or merge?
3. You run `helm upgrade my-app ./chart` and it breaks. How do you rollback? What command shows you the history?
4. What's the difference between `helm install` and `helm upgrade --install`?
5. In Kustomize, what's a "base" and what's an "overlay"? Can an overlay modify things the base doesn't define?
6. What is GitOps in one sentence? What's the "single source of truth"?
7. Someone runs `kubectl apply` manually in production, changing a deployment. With ArgoCD's `selfHeal: true`, what happens?
8. What's the difference between ArgoCD's `prune: true` and `prune: false`? What happens if you delete a manifest from git with prune disabled?
9. You push a new image tag to ECR. Does ArgoCD automatically deploy it? Why or why not?
10. What's the "App of Apps" pattern in ArgoCD? Why is it useful?
11. In a canary deployment (10% → 30% → 100%), you notice errors at 30%. What do you do? How is this better than a regular rolling update?
12. What's the difference between blue-green and canary deployment strategies?
13. Your CI pipeline builds an image and pushes to ECR. What's the NEXT step to trigger a GitOps deployment? (Hint: it's not "kubectl apply")
14. Why should you NEVER use `:latest` tag in production manifests managed by GitOps?
15. You have the same app deployed in dev, staging, and production. With Kustomize overlays, how do you manage different replica counts and resource limits without duplicating YAML?

## Checklist Before Moving On

- [ ] Can create Helm charts with templates and values
- [ ] Can use Kustomize for environment-specific overlays
- [ ] Know when to use Helm vs Kustomize (or both)
- [ ] Can install and configure ArgoCD
- [ ] Understand GitOps principles (git as source of truth, auto-sync, drift detection)
- [ ] Can set up a CI pipeline that builds and pushes images
- [ ] Understand the full flow: code → image → manifest update → ArgoCD → cluster
- [ ] Can implement canary or blue-green deployments
- [ ] Know how to rollback with GitOps (revert git commit)
