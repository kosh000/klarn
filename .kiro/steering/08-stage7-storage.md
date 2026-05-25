---
inclusion: manual
---

# Stage 7: Storage & Stateful Workloads

## Goal
Run stateful applications on Kubernetes — databases, caches, message queues — with persistent storage that survives pod restarts.

## The Problem

Pods are ephemeral. When a pod dies, its filesystem is gone. For stateless apps (web servers, APIs), this is fine — they don't store data locally. But databases, caches, and file stores NEED persistent data.

## Storage Concepts

### Volume Types

| Type | Lifecycle | Use case |
|------|-----------|----------|
| **emptyDir** | Same as pod (deleted when pod dies) | Temp files, cache, shared between containers in a pod |
| **hostPath** | Same as node (dangerous) | Testing only, never in production |
| **PersistentVolume (PV)** | Independent of pod | Databases, file storage |
| **ConfigMap/Secret** | Mounted as read-only files | Configuration |

### PersistentVolume (PV) and PersistentVolumeClaim (PVC)

**PV** = the actual storage (an EBS volume, an EFS filesystem)
**PVC** = a request for storage (pod says "I need 10Gi of fast storage")
**StorageClass** = defines HOW storage is provisioned (what type of EBS, what IOPS)

Flow:
1. Pod references a PVC
2. PVC requests storage from a StorageClass
3. StorageClass dynamically provisions a PV (creates an EBS volume)
4. PV is bound to the PVC
5. Pod mounts the PV

### StorageClass
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3           # EBS volume type
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete   # Delete EBS volume when PVC is deleted
volumeBindingMode: WaitForFirstConsumer  # Don't create until pod is scheduled
allowVolumeExpansion: true  # Allow resizing
```

### PersistentVolumeClaim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce    # Can be mounted by one node at a time (EBS)
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
```

### Using in a Pod
```yaml
spec:
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: postgres-data
```

## Access Modes

| Mode | Abbreviation | What it means | AWS storage |
|------|-------------|---------------|-------------|
| ReadWriteOnce | RWO | One node can mount read-write | EBS |
| ReadOnlyMany | ROX | Many nodes can mount read-only | EBS (snapshot), EFS |
| ReadWriteMany | RWX | Many nodes can mount read-write | EFS |

**EBS = block storage** (like a hard drive attached to one machine)
**EFS = network filesystem** (like NFS, shared across machines)

## EBS CSI Driver

Allows Kubernetes to create/attach/delete EBS volumes.

```bash
# Install EBS CSI Driver add-on
aws eks create-addon --cluster-name eks-learning --addon-name aws-ebs-csi-driver

# Verify
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

The driver needs IAM permissions to manage EBS volumes. Use Pod Identity:
```bash
aws eks create-pod-identity-association \
  --cluster-name eks-learning \
  --namespace kube-system \
  --service-account ebs-csi-controller-sa \
  --role-arn arn:aws:iam::123456789:role/EBS-CSI-Role
```

## EFS CSI Driver

For shared storage (ReadWriteMany) — multiple pods on different nodes can read/write the same filesystem.

```bash
# Create EFS filesystem
aws efs create-file-system --creation-token eks-efs --encrypted

# Install EFS CSI Driver
aws eks create-addon --cluster-name eks-learning --addon-name aws-efs-csi-driver
```

```yaml
# StorageClass for EFS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0123456789abcdef
  directoryPerms: "700"
```

## StatefulSets

For workloads that need:
- Stable network identity (pod-0, pod-1, pod-2 — not random names)
- Stable persistent storage (each pod gets its own PVC that follows it)
- Ordered deployment and scaling (pod-0 starts first, then pod-1, etc.)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres  # Required: headless service name
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
  volumeClaimTemplates:    # Each pod gets its own PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 20Gi
---
# Headless service (required for StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

**StatefulSet behavior:**
- Pods are named: `postgres-0`, `postgres-1`, `postgres-2`
- Each gets its own PVC: `data-postgres-0`, `data-postgres-1`, `data-postgres-2`
- DNS: `postgres-0.postgres.default.svc.cluster.local`
- Scale up: creates pod-3, then pod-4 (ordered)
- Scale down: deletes pod-4 first, then pod-3 (reverse order)
- PVCs are NOT deleted when pods are deleted (data preserved)

## Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random (my-app-7f8d9-abc) | Ordered (my-app-0, my-app-1) |
| Storage | Shared or none | Each pod gets its own PVC |
| Scaling | Parallel (all at once) | Ordered (one at a time) |
| Network identity | Random, interchangeable | Stable DNS per pod |
| Use case | Stateless apps (APIs, web) | Databases, distributed systems |

## Reclaim Policies

What happens to the PV when the PVC is deleted:

| Policy | Behavior |
|--------|----------|
| **Delete** | PV and underlying storage (EBS volume) are deleted |
| **Retain** | PV is kept, must be manually cleaned up |

**For databases:** Use `Retain` so you don't accidentally lose data.

## Volume Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: data-postgres-0
```

Creates an EBS snapshot. Can be used to restore data or create new volumes.

## Labs

### Lab 7.1: Basic Persistent Storage
1. Create a StorageClass for gp3 EBS volumes
2. Create a PVC requesting 5Gi
3. Deploy a pod that writes a file to the mounted volume
4. Delete the pod
5. Create a new pod with the same PVC — verify the file is still there

### Lab 7.2: StatefulSet with PostgreSQL
1. Deploy PostgreSQL as a StatefulSet with 1 replica
2. Connect and create a table, insert data
3. Delete the pod (not the StatefulSet)
4. Wait for it to restart — verify data is still there
5. Scale to 3 replicas — observe ordered creation

### Lab 7.3: EFS Shared Storage
1. Create an EFS filesystem
2. Deploy two pods on different nodes, both mounting the same EFS PVC
3. Write a file from pod-1
4. Read it from pod-2 — verify shared access

### Lab 7.4: Volume Expansion
1. Create a PVC with 5Gi
2. Fill it up (dd if=/dev/zero)
3. Expand the PVC to 10Gi: `kubectl edit pvc` → change storage
4. Verify the filesystem grew

### Lab 7.5: Backup and Restore
1. Create a VolumeSnapshot of your PostgreSQL data
2. Delete the PVC and StatefulSet
3. Restore from snapshot (create new PVC from snapshot)
4. Deploy PostgreSQL again — verify data is restored

## Self-Test Questions

1. What's the difference between a PersistentVolume (PV) and a PersistentVolumeClaim (PVC)? Who creates each?
2. You delete a pod that has a PVC mounted. Is the data lost? What about if you delete the PVC?
3. What does `reclaimPolicy: Delete` mean? Why is this dangerous for databases?
4. What's the difference between `ReadWriteOnce` (RWO) and `ReadWriteMany` (RWX)? Which AWS storage supports each?
5. You have a pod on node-1 with an EBS volume. The pod gets rescheduled to node-2. What happens to the EBS volume?
6. Why can't two pods on different nodes share an EBS volume? What storage type would you use instead?
7. What's a StatefulSet? Name 3 things it gives you that a Deployment doesn't.
8. You have a StatefulSet with 3 replicas. You scale down to 1. Which pods get deleted first? (pod-0, pod-1, or pod-2?)
9. You delete a StatefulSet. Are the PVCs deleted too? Why or why not?
10. What's `volumeBindingMode: WaitForFirstConsumer`? Why is it important for EBS volumes?
11. You create a PVC requesting 10Gi but your StorageClass has `allowVolumeExpansion: true`. Can you later change it to 20Gi? Does the pod need to restart?
12. What's the difference between `emptyDir` and a PVC? When would you use `emptyDir`?
13. You deploy PostgreSQL as a Deployment with a PVC. Why is this problematic compared to using a StatefulSet?
14. What's a VolumeSnapshot? How is it different from an EBS snapshot you create manually?
15. Your StatefulSet pod `postgres-0` has DNS name `postgres-0.postgres.default.svc.cluster.local`. What Service type makes this possible?

## Checklist Before Moving On

- [ ] Understand PV, PVC, StorageClass relationship
- [ ] Can set up EBS CSI driver on EKS
- [ ] Can create StorageClasses with different performance tiers
- [ ] Understand access modes (RWO vs RWX) and when to use each
- [ ] Can deploy StatefulSets with volumeClaimTemplates
- [ ] Know the difference between Deployment and StatefulSet
- [ ] Understand reclaim policies (Delete vs Retain)
- [ ] Can create and restore from VolumeSnapshots
- [ ] Know when to use EBS vs EFS
- [ ] Can run a database (PostgreSQL) on EKS with persistent storage
