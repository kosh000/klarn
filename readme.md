# EKS - Deployment

Deploying app on Kube, it wil have Ingress and its controller and will be accessing on ALB DNS. VPC will have Public and Private Subnets.

In kube there are 2 components:

- Control Plane (Master Node)
- Data Plane (Worker Node)

If HA is required it's recommend to go with 3 Master Nodes, each master node will have 1 EC2.

## Master Node Components

Listed components are main components which are responsible for managing the Kube Cluster. Those mentioned components are user facing components, if you want to access any app then those controller are accessed and your request goes to the data plane where the app is actually installed.

- API
- ETCD (_configuration information for large scale distributed systems_)
- Scheduler
- Cloud Control Manager
- Controller Manager

## Worker Node Components

Listed components are required for worker nodes for the deployments. Once those are installed you have to make join those worker nodes to the control plane.

- Container Network Interface
- Container Runtime
- DNS Service
- Kube Proxy

Note: _Kubeadm Reference: Kubeadm is a tool built to provide kubeadm init and kubeadm join as best-practice "fast paths" for creating Kubernetes clusters. kubeadm performs the actions necessary to get a minimum viable cluster up and running. By design, it cares only about bootstrapping, not about provisioning machines._

## EKS

The EKS is provided by AWS which is a managed service. The Control Plane part of the Kube is managed by AWS themselves. You go to AWS and get a Cluster for yourself and you get a managed control plane. It provides a very easy way to manage Control Plane.
The Master Nodes are managed by AWS themselves, you may not know where they are, for attaching the Worker Nodes there are 2 options which are provided by AWS.

TLDR; EKS is fully-managed kube service by AWS which takes your pain away (❁´◡`❁). You do not have to manage the certificates, api server slowness, ETCD crashes and more.

- Create EC2 Yourselves. (_SelfManaged, you will have to manage the number of instances you will have your deployments deployed on, if you need more compute for your deployment then you will have to deploy more EC2 instances_)
- Fargate. (_Serverless Compute for containers, the Autoscaling/HA is managed by the AWS_)

### Ways to use EKS on AWS

1. Create VMs yourself and use KOPS or Kubeadm.
2. USE EKS XD.
3. Installing Kube Cluster on-prem.

## Kube Tree

```mermaid
graph TB
  A[Kubernetes Cluster] --> B[Node]
  A --> C[Pod]
  A --> D[Service]
  A --> E[Ingress]
  A --> F[ConfigMap & Secret]

  B --> G[Kubelet]
  B --> H[Container Runtime]

  C --> I[Container]
  C --> J[Volumes]

  D --> K[Load Balancer]
  D --> L[DNS]

  E --> M[HTTPs Routing]

  F --> N[Key-Value Store]
  F --> O[Configuration Data]

  subgraph Control Plane
    P[Kube API Server]
    P --> Q[Controller Manager]
    P --> R[Scheduler]
    P --> S[ETCD]
  end

  subgraph Worker Node
    B
    G
    H
    C
    D
  end
```

## Accessing the Pod

There are 3 ways to access a POD.

- Cluster IP (_this works only inside the Cluster, ie masternodes or workernodes._)
- Node Port (_Exposing on Node Level, as we will deploy in private only ppl within the VPC will be able to access it_)
- Load Balancer (_Exposing through the LB, yka_)

## What is?

- Namespace
  - A Namespace partitions a single cluster so resource names only need to be unique within that namespace, enabling isolation, policy, and quota boundaries.​
  - Most Kubernetes objects, including Deployments and Services, are namespaced; label selectors belong to those resources, not to the Namespace itself.
  
- Deployment
  - A Deployment declares desired state for Pods indirectly via ReplicaSets, handling rolling updates, rollbacks, and self‑healing by recreating Pods.​
  - Because Pods are ephemeral, Deployments give a higher‑level, declarative interface for versioned rollouts of your application.​

- Service
  - A Service abstracts a logical set of Pods and provides a stable virtual IP and DNS name, with endpoints tracked automatically (via EndpointSlices) as Pods change.​
  - It targets Pods usually by label selector and can be exposed as ClusterIP (internal), NodePort, or LoadBalancer, depending on how you need to reach it.​
  - Services are namespaced; cross‑namespace access uses DNS (svc.ns.svc.cluster.local) or a Service without selector plus Endpoints mapping, not a selector across namespaces.​

- Ingress
  - Ingress defines HTTP/HTTPS routing rules (host/path, TLS) that route external traffic to Services inside the cluster and must be fulfilled by an Ingress controller.​
  - Ingress is stable, but new features are moving to the Gateway API; if you use Ingress, review your controller’s specifics since behavior varies.​

- EKS nuance
  - On EKS, an Ingress only takes effect when an Ingress controller (commonly the AWS Load Balancer Controller) is present to provision ALB/NLB and program routes to Services.​
  - If exposing a single backend, a Service of type LoadBalancer is straightforward; Ingress is preferred when routing multiple Services under one hostname or IP.​

## Notes

Exposing Pods or SVC or Ingress:

- This will enable the pod/alb to be able to talk to the pod.
- Here, what is happening - When the controller is deployed an ALB is deployed within the Kube Environment which is exposed to the Internet and all of the traffic which is coming is handled by that pod 

TRASH:
For the Tutorial, that I had watched; doing this would require you to use the OIDC for IAM for the cluster.

## Abbreviation

- High Availability (HA)
- Application Load Balancer (ALB)
- Load Balancer (LB)
- Network Load Balancer (NLB)
- Kubernetes (Kube)
- EKS (Elastic Kubernetes Service)