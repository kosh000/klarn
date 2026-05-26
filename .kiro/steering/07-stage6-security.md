---
inclusion: manual
description: "Stage 6: Security — RBAC, Pod Identity, Pod Security Standards, Kyverno"
---

# Stage 6: Security

## Goal
Lock down your cluster: control who can do what (RBAC), give pods AWS permissions safely (Pod Identity), enforce security standards, and manage secrets properly.

## RBAC (Role-Based Access Control)

### Concepts
- **Subject**: who (User, Group, ServiceAccount)
- **Role**: what they can do (list of permissions)
- **RoleBinding**: connects subject to role

### Scope
- **Role + RoleBinding**: namespace-scoped (permissions within one namespace)
- **ClusterRole + ClusterRoleBinding**: cluster-wide (all namespaces)

### Role Example
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]           # "" = core API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get"]
```

### ClusterRole Example
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: deployment-manager
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### Binding to a User
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-dev
  namespace: dev
subjects:
  - kind: User
    name: jane@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ServiceAccount (for pods)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Check what you can do
kubectl auth can-i create deployments
kubectl auth can-i delete pods --namespace=production

# Check what a service account can do
kubectl auth can-i get pods --as=system:serviceaccount:default:my-app-sa
```

### EKS Access Management

EKS uses **access entries** to map IAM principals to Kubernetes RBAC:

```bash
# Grant an IAM role cluster admin access
aws eks create-access-entry \
  --cluster-name eks-learning \
  --principal-arn arn:aws:iam::123456789:role/DevOpsRole \
  --type STANDARD

aws eks associate-access-policy \
  --cluster-name eks-learning \
  --principal-arn arn:aws:iam::123456789:role/DevOpsRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

## EKS Pod Identity (Giving Pods AWS Permissions)

**Problem:** Your pod needs to access S3, DynamoDB, SQS, etc. How do you give it AWS credentials without hardcoding access keys?

**Solution:** EKS Pod Identity — associates a Kubernetes ServiceAccount with an IAM Role.

### Setup
```bash
# 1. Install the Pod Identity Agent add-on
aws eks create-addon --cluster-name eks-learning --addon-name eks-pod-identity-agent

# 2. Create an IAM role with trust policy for Pod Identity
# Trust policy (allows EKS Pod Identity to assume this role):
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "pods.eks.amazonaws.com" },
    "Action": ["sts:AssumeRole", "sts:TagSession"]
  }]
}
EOF

aws iam create-role --role-name my-app-s3-role --assume-role-policy-document file://trust-policy.json

# 3. Attach permissions to the role
aws iam attach-role-policy --role-name my-app-s3-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 4. Create the association
aws eks create-pod-identity-association \
  --cluster-name eks-learning \
  --namespace default \
  --service-account my-app-sa \
  --role-arn arn:aws:iam::123456789:role/my-app-s3-role
```

### Use in Pod
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-sa  # This pod now has S3 access
      containers:
        - name: my-app
          image: my-app:v1
```

The pod automatically gets temporary AWS credentials. No access keys needed.

### Pod Identity vs IRSA

| Feature | Pod Identity (new, recommended) | IRSA (legacy, still works) |
|---------|-------------------------------|---------------------------|
| Setup complexity | Simple — no OIDC config needed | Complex — requires OIDC provider setup |
| Trust policy | Universal (same for all clusters) | Per-cluster (OIDC URL in trust policy) |
| Session tags | Automatic (cluster, namespace, SA) | Manual |
| Cross-account | Easier | Harder |
| When to use | New clusters (2024+) | Existing clusters already using it |

## Pod Security Standards

Kubernetes built-in security enforcement. Three levels:

| Level | What it allows |
|-------|---------------|
| **Privileged** | Anything (no restrictions) |
| **Baseline** | Prevents known privilege escalations (no hostNetwork, no privileged containers) |
| **Restricted** | Heavily restricted (must run as non-root, drop all capabilities, read-only root filesystem) |

### Enforce at Namespace Level
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

Now any pod in `production` namespace that violates "restricted" policy will be rejected.

### What "Restricted" Requires
```yaml
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: my-app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
      # If you need to write temp files:
      volumeMounts:
        - name: tmp
          mountPath: /tmp
  volumes:
    - name: tmp
      emptyDir: {}
```

## Kyverno (Policy Engine)

More flexible than Pod Security Standards. Can enforce custom rules.

### Admission Controllers & Webhooks (How Policy Enforcement Works)

Before understanding Kyverno, you need to understand the mechanism it uses: **admission controllers**.

**What happens when you `kubectl apply`:**
```
kubectl apply → API Server → Authentication → Authorization (RBAC) → Admission Controllers → etcd (stored)
```

Admission controllers intercept requests AFTER auth but BEFORE persistence. Two types:

1. **Mutating Admission Webhooks** — can MODIFY the request (add labels, inject sidecars, set defaults)
2. **Validating Admission Webhooks** — can ACCEPT or REJECT the request (enforce policies)

Order: Mutating runs first → then Validating.

**Real-world examples:**
- Istio sidecar injection = mutating webhook (adds envoy container to every pod)
- Kyverno policy enforcement = validating webhook (rejects pods without resource limits)
- Pod Security Admission = built-in validating admission (rejects privileged pods)
- AWS EKS Pod Identity Agent = mutating webhook (injects credentials into pods)

**Why this matters:**
- Every policy tool (Kyverno, OPA/Gatekeeper, Pod Security) uses this mechanism
- If admission webhooks are down, pod creation can be blocked cluster-wide
- Understanding this helps you debug "why can't I create this pod?" issues

```yaml
# Example: what a webhook configuration looks like (you rarely write these manually)
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: my-policy-webhook
webhooks:
  - name: validate.example.com
    rules:
      - apiGroups: [""]
        resources: ["pods"]
        operations: ["CREATE", "UPDATE"]
    clientConfig:
      service:
        name: policy-service
        namespace: policy-system
        path: /validate
    failurePolicy: Fail    # If webhook is down: Fail (block) or Ignore (allow)
    sideEffects: None
```

**Kubernetes 1.36 addition:** Mutating Admission Policies (CEL-based) — write admission policies without webhooks, using CEL expressions directly. Simpler than deploying a webhook server.

```yaml
# CEL-based validation (no webhook server needed) — K8s 1.36+
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-resource-limits
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        resources: ["pods"]
        operations: ["CREATE"]
  validations:
    - expression: "object.spec.containers.all(c, has(c.resources) && has(c.resources.limits))"
      message: "All containers must have resource limits"
```

Now let's look at Kyverno, which uses validating/mutating webhooks under the hood:

```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

### Example Policies
```yaml
# Require resource limits on all containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-limits
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "CPU and memory limits are required"
        pattern:
          spec:
            containers:
              - resources:
                  limits:
                    cpu: "?*"
                    memory: "?*"
---
# Require specific labels
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-team-label
      match:
        any:
          - resources:
              kinds: ["Deployment"]
      validate:
        message: "The label 'team' is required"
        pattern:
          metadata:
            labels:
              team: "?*"
---
# Disallow latest tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-image-tag
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Using ':latest' tag is not allowed"
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

## Secrets Management (Production-Grade)

### ServiceAccount Token Security

By default, every pod gets a ServiceAccount token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This token can be used to call the Kubernetes API.

**Problem:** Most pods don't need API access. A compromised pod with a mounted token can enumerate the cluster.

**Best practice:** Disable automatic token mounting for pods that don't need it:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
automountServiceAccountToken: false   # Don't mount token by default
---
# Or per-pod:
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: my-app-sa
      automountServiceAccountToken: false  # Override at pod level
```

**When you DO need API access** (e.g., operators, controllers), use projected volumes with short-lived tokens:

```yaml
spec:
  containers:
    - name: my-controller
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600    # 1 hour, auto-rotated
              audience: "https://kubernetes.default.svc"
```

Projected tokens are short-lived (auto-expire and rotate) vs the default token which never expires.

### Container Image Security (Trivy)

Before deploying images, scan them for known vulnerabilities:

```bash
# Scan a local image
trivy image my-app:v1

# Scan with severity filter
trivy image --severity HIGH,CRITICAL my-app:v1

# Scan in CI pipeline (fail build on critical CVEs)
trivy image --exit-code 1 --severity CRITICAL my-app:v1

# Scan a remote image from ECR
trivy image 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1
```

**Integrate in CI pipeline (GitHub Actions):**
```yaml
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.ECR_REPO }}:${{ github.sha }}
    format: 'table'
    exit-code: '1'
    severity: 'CRITICAL,HIGH'
```

**Enforce in cluster with Kyverno:**
```yaml
# Block images that haven't been scanned or have critical CVEs
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-from-ecr
spec:
  validationFailureAction: Enforce
  rules:
    - name: only-ecr-images
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Images must come from our ECR registry"
        pattern:
          spec:
            containers:
              - image: "123456789.dkr.ecr.*.amazonaws.com/*"
```

**Best practices for image security:**
- Scan in CI (block vulnerable images from being deployed)
- Use minimal base images (Alpine, distroless) — fewer packages = fewer CVEs
- Pin image digests in production (not just tags — tags can be overwritten)
- Regularly rebuild images to pick up base image security patches
- Use ECR image scanning (automatic on push)

### cert-manager (TLS Certificate Automation)

On EKS, you use ACM (AWS Certificate Manager) for free TLS certs. On bare metal or non-AWS environments, **cert-manager** automates certificate issuance and renewal.

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.0/cert-manager.yaml

# Or via Helm
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

**Set up Let's Encrypt issuer:**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: envoy    # Or your gateway class
      - dns01:
          route53:
            region: us-east-1          # For wildcard certs on AWS
```

**Request a certificate:**
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-tls
  namespace: default
spec:
  secretName: api-tls-secret       # K8s Secret created with cert + key
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.example.com
    - "*.api.example.com"
  duration: 2160h                   # 90 days
  renewBefore: 360h                 # Renew 15 days before expiry
```

cert-manager automatically renews certificates before they expire. The resulting Secret can be referenced in Gateway API or Ingress resources.

**When to use what:**

| Environment | TLS Solution | Cost |
|-------------|-------------|------|
| EKS (AWS) | ACM (via ALB annotation) | Free |
| On-prem / bare-metal | cert-manager + Let's Encrypt | Free |
| Internal services (mTLS) | cert-manager + self-signed CA | Free |
| Enterprise (compliance) | cert-manager + Venafi/DigiCert | Paid |

### External Secrets Operator + AWS Secrets Manager

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

```yaml
# 1. Create a SecretStore (connects to AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa  # Uses Pod Identity
---
# 2. Create an ExternalSecret (syncs from AWS to K8s Secret)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets
    kind: SecretStore
  target:
    name: db-credentials    # K8s Secret that gets created
  data:
    - secretKey: username
      remoteRef:
        key: production/database  # AWS Secrets Manager secret name
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

Now your secrets live in AWS Secrets Manager (encrypted, audited, rotatable) and automatically sync to Kubernetes Secrets.

## Labs

### Lab 6.1: RBAC
1. Create a ServiceAccount `dev-sa` in namespace `dev`
2. Create a Role that only allows reading pods and logs
3. Bind the Role to the ServiceAccount
4. Test: `kubectl auth can-i` with `--as` flag
5. Try to create a deployment with that SA — should be denied

### Lab 6.2: Pod Identity
1. Create an IAM role with S3 read access
2. Set up Pod Identity association
3. Deploy a pod that lists S3 buckets using AWS SDK
4. Verify it works without any hardcoded credentials
5. Remove the association — verify access is denied

### Lab 6.3: Pod Security Standards
1. Create a namespace with `enforce: restricted`
2. Try to deploy a pod running as root — should be rejected
3. Fix the pod spec to comply with restricted policy
4. Deploy successfully

### Lab 6.4: Kyverno Policies
1. Install Kyverno
2. Create a policy requiring resource limits
3. Try to deploy without limits — rejected
4. Create a policy disallowing `:latest` tag
5. Try to deploy with `:latest` — rejected

### Lab 6.5: External Secrets
1. Create a secret in AWS Secrets Manager
2. Install External Secrets Operator
3. Create SecretStore and ExternalSecret
4. Verify the K8s Secret is created automatically
5. Update the secret in AWS — verify it syncs

## Self-Test Questions

1. What's the difference between a Role and a ClusterRole? When would you use each?
2. You create a Role in namespace `dev` and bind it to a ServiceAccount. Can that ServiceAccount do anything in namespace `prod`?
3. A pod runs without specifying a `serviceAccountName`. What ServiceAccount does it use? What permissions does it have?
4. What's the difference between EKS Pod Identity and IRSA? Which should you use for a new cluster in 2026?
5. With Pod Identity, where do the AWS credentials come from? Are they long-lived access keys?
6. You label a namespace with `pod-security.kubernetes.io/enforce: restricted`. A developer tries to deploy a pod running as root. What happens?
7. What does `allowPrivilegeEscalation: false` prevent? Why is it important?
8. Kubernetes Secrets are "base64 encoded." A developer says "our secrets are encrypted." Are they right? What's actually needed for encryption?
9. You use External Secrets Operator to sync from AWS Secrets Manager. Someone rotates the secret in AWS. How long until the pod gets the new value?
10. What's an admission controller? How does Kyverno use them?
11. You write a Kyverno policy that requires all Deployments to have a `team` label. An existing Deployment without that label is already running. Does Kyverno delete it?
12. What's the principle of least privilege? Give an example of violating it in Kubernetes.
13. A pod needs to read from S3 and write to DynamoDB. Should you give it `AdministratorAccess`? What should you do instead?
14. What's the difference between `enforce`, `warn`, and `audit` modes in Pod Security Standards?
15. You have a ServiceAccount with no RBAC bindings. Can a pod using it still make API calls to the Kubernetes API server?
16. What's `automountServiceAccountToken: false`? Why should most pods have this set?
17. What's the difference between the default ServiceAccount token and a projected token with `expirationSeconds`? Which is more secure?
18. You run `trivy image my-app:v1` and find 3 CRITICAL CVEs. What should you do? Can you enforce this in the cluster?
19. What's cert-manager? When would you use it instead of ACM?
20. You deploy cert-manager with a Let's Encrypt ClusterIssuer. A Certificate resource requests `api.example.com`. What happens automatically?

## Checklist Before Moving On

- [ ] Can create Roles, ClusterRoles, and Bindings
- [ ] Understand ServiceAccounts and how pods use them
- [ ] Can set up EKS Pod Identity for AWS access
- [ ] Know the difference between Pod Identity and IRSA
- [ ] Can enforce Pod Security Standards at namespace level
- [ ] Understand admission controllers (mutating vs validating webhooks)
- [ ] Know how Kyverno/OPA use admission webhooks under the hood
- [ ] Can write Kyverno policies for custom enforcement
- [ ] Can set up External Secrets Operator with AWS Secrets Manager
- [ ] Understand the principle of least privilege in Kubernetes
- [ ] Know how EKS access entries work (IAM → K8s RBAC mapping)
- [ ] Know when and how to disable automountServiceAccountToken
- [ ] Understand projected ServiceAccount tokens (short-lived, auto-rotated)
- [ ] Can scan container images with Trivy and integrate into CI
- [ ] Can install and configure cert-manager for TLS automation
- [ ] Know when to use ACM (EKS) vs cert-manager (on-prem)
