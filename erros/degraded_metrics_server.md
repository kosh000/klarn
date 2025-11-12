# Error
kosh@GoldenExperience:~/kosh_kube$ kubectl describe pod metrics-server-5c6bdf7f85-lwd9h -n kube-system
Name:                 metrics-server-5c6bdf7f85-lwd9h
Namespace:            kube-system
Priority:             2000000000
Priority Class Name:  system-cluster-critical
Service Account:      metrics-server
Node:                 <none>
Labels:               app.kubernetes.io/instance=metrics-server
                      app.kubernetes.io/name=metrics-server
                      pod-template-hash=5c6bdf7f85
Annotations:          <none>
Status:               Pending
IP:                   
IPs:                  <none>
Controlled By:        ReplicaSet/metrics-server-5c6bdf7f85
Containers:
  metrics-server:
    Image:           602401143452.dkr.ecr.ap-south-1.amazonaws.com/eks/metrics-server:v0.8.0-eksbuild.3
    Port:            10251/TCP (https)
    Host Port:       0/TCP (https)
    SeccompProfile:  RuntimeDefault
    Args:
      --secure-port=10251
      --cert-dir=/tmp
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      --kubelet-use-node-status-port
      --metric-resolution=15s
    Limits:
      memory:  400Mi
    Requests:
      cpu:        100m
      memory:     200Mi
    Liveness:     http-get https://:https/livez delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:    http-get https://:https/readyz delay=20s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /tmp from tmp (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4vl8q (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  tmp:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:     
    SizeLimit:  <unset>
  kube-api-access-4vl8q:
    Type:                     Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:   3607
    ConfigMapName:            kube-root-ca.crt
    Optional:                 false
    DownwardAPI:              true
QoS Class:                    Burstable
Node-Selectors:               <none>
Tolerations:                  CriticalAddonsOnly op=Exists
                              node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                              node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Topology Spread Constraints:  topology.kubernetes.io/zone:ScheduleAnyway when max skew 1 is exceeded for selector app.kubernetes.io/instance=metrics-server,app.kubernetes.io/name=metrics-server
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  30m (x10 over 75m)  default-scheduler  0/5 nodes are available: 5 node(s) had untolerated taint {eks.amazonaws.com/compute-type: fargate}. preemption: 0/5 nodes are available: 5 Preemption is not helpful for scheduling.
  Warning  FailedScheduling  45s                 default-scheduler  0/5 nodes are available: 5 node(s) had untolerated taint {eks.amazonaws.com/compute-type: fargate}. preemption: 0/5 nodes are available: 5 Preemption is not helpful for scheduling.

# What did fix it?

kubectl patch deployment metrics-server -n kube-system --type=merge --patch='
spec:
  template:
    spec:
      tolerations:
      - key: "eks.amazonaws.com/compute-type"
        operator: "Equal"
        value: "fargate"
        effect: "NoSchedule"
'

# Why?

| Concept        | Meaning                                  | Why it matters here             |
| -------------- | ---------------------------------------- | ------------------------------- |
| **Taint**      | Restricts what pods can run on a node    | All Fargate nodes are tainted   |
| **Toleration** | Lets a pod run on tainted nodes          | Metrics-server lacked one       |
| **Fix**        | Add toleration or create Fargate profile | Scheduler can now place the pod |

Your Fargate nodes are tainted with eks.amazonaws.com/compute-type=fargate:NoSchedule, which blocks any pod without a matching toleration.

The metrics-server pods lacked that toleration, so the scheduler couldn’t place them — they stayed Pending.

Adding a toleration (or creating a Fargate profile) lets Kubernetes schedule them on Fargate nodes, fixing the issue.