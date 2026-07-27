# Session 3 — Kubernetes Basics

## Lab 07 — Kubernetes Architecture & Core Components

### 🎯 Objective

Understand **Kubernetes (K8s) architecture**, including the **Control Plane**, **Worker Nodes**, and core Kubernetes API objects.

By the end of this lab, you will understand:

* Kubernetes Control Plane components.
* Worker Node components.
* Pods and containers.
* ReplicaSets and Deployments.
* Kubernetes Services.
* Namespaces and isolation.
* Basic `kubectl` commands.
* Kubernetes cluster inspection and troubleshooting.

---

# 🏗️ Kubernetes Architecture Overview

A Kubernetes cluster consists primarily of:

1. **Control Plane**
2. **Worker Nodes**

The Control Plane manages the cluster, while Worker Nodes run application workloads.

```text
                         Kubernetes Cluster
                                │
                ┌───────────────┴───────────────┐
                │                               │
          Control Plane                     Worker Nodes
                │                               │
       ┌────────┼────────┐              ┌───────┼────────┐
       │        │        │              │       │        │
   API Server  etcd  Scheduler       kubelet kube-proxy containerd
       │
       ▼
 Controllers
```

---

# 🎛️ Control Plane

The **Control Plane** manages the overall Kubernetes cluster and maintains the desired state of workloads.

The primary components are:

### API Server

The **Kubernetes API Server** provides the central interface for interacting with the Kubernetes cluster.

Tools such as `kubectl` communicate with the API Server.

```text
kubectl → API Server → Kubernetes Resources
```

### etcd

`etcd` is the Kubernetes distributed key-value store.

It stores important cluster state and configuration information.

### Scheduler

The **kube-scheduler** determines which Worker Node should run newly created Pods.

### Controller Manager

The **Controller Manager** runs Kubernetes controllers that continuously work to bring the actual cluster state toward the desired state.

---

# 🖥️ Worker Node

Worker Nodes run application workloads.

The primary components are:

### kubelet

The **kubelet** runs on each Worker Node and communicates with the Kubernetes API Server to ensure that Pods assigned to the node are running as expected.

### kube-proxy

`kube-proxy` provides network-related functionality for Kubernetes Services and traffic routing.

### Container Runtime

The **Container Runtime** runs containers.

The workbook uses:

```text
containerd
```

---

# 📦 Core Kubernetes Objects

## Pod

A **Pod** is the smallest deployable unit in Kubernetes.

A Pod contains:

* One or more containers
* Shared networking
* Shared storage resources where configured

```text
Pod
 ├── Container
 └── Container
```

Most commonly, a Pod contains one application container, although multi-container Pods are also supported.

---

## ReplicaSet

A **ReplicaSet** ensures that the desired number of replicas of a Pod are running.

For example:

```text
ReplicaSet
    │
    ├── Pod 1
    ├── Pod 2
    └── Pod 3
```

If one Pod fails, the ReplicaSet attempts to create another Pod to maintain the desired replica count.

---

## Deployment

A **Deployment** manages ReplicaSets and provides higher-level application deployment capabilities.

Deployments support features such as:

* Replica management
* Rolling updates
* Application version changes
* Rollback capabilities

The relationship is:

```text
Deployment
     │
     ▼
ReplicaSet
     │
     ├── Pod
     ├── Pod
     └── Pod
```

---

## Service

A **Service** provides a stable network endpoint for a set of Pods.

Because Pod IP addresses can change when Pods are recreated, applications should generally communicate with a Service rather than relying on individual Pod IP addresses.

```text
Client
   │
   ▼
Service
   │
   ├── Pod
   ├── Pod
   └── Pod
```

---

## Namespace

A **Namespace** provides a logical boundary within a Kubernetes cluster.

Namespaces can be used for:

* Workload organization
* Environment separation
* Resource management
* Access control
* Isolation

For example:

```text
Kubernetes Cluster
│
├── dev
├── test
└── production
```

---

# 🚀 Kubernetes Environment Setup

You can complete this lab using either:

* **Option A — Minikube**
* **Option B — AWS EKS**

---

## Option A — Minikube (Local)

Start Minikube with Docker:

```bash
minikube start --driver=docker --cpus=2 --memory=4096
```

Check the Minikube status:

```bash
minikube status
```

> 💡 **Tip:** Minikube is useful for learning Kubernetes locally without creating an AWS EKS cluster.

---

# Option B — AWS EKS (Console)

Use the AWS Console to create an Amazon EKS cluster.

### Step 1 — Create the EKS Cluster

In the AWS Console:

```text
EKS → Create cluster
```

Use:

```text
Cluster Name: devops-cluster
Kubernetes Version: 1.28
```

> ⚠️ **Version Note:** Kubernetes versions have a limited support lifecycle. The original workbook specifies **Kubernetes 1.28**. When performing this lab today, select a currently supported EKS version if `1.28` is no longer available.

### Step 2 — Configure the Node Group

Create a node group using:

```text
Instance Type: t3.medium
Nodes: 2
Availability Zones:
  ap-south-1a
  ap-south-1b
```

This provides two Worker Nodes across two Availability Zones.

---

# 🔧 Configure kubectl for EKS

After the EKS cluster is created, configure `kubectl`:

```bash
aws eks update-kubeconfig \
  --name devops-cluster \
  --region ap-south-1
```

This updates the local Kubernetes configuration so that `kubectl` can communicate with the EKS cluster.

---

# 🔍 Verify the Kubernetes Cluster

Check the Worker Nodes:

```bash
kubectl get nodes
```

List Pods across all Namespaces:

```bash
kubectl get pods -A
```

Check cluster information:

```bash
kubectl cluster-info
```

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                         |
| -------- | ---------------------------------------------------------------------------------- |
| **T1.1** | Start Minikube or connect to EKS. Run: `kubectl get nodes` — note `STATUS=Ready`.  |
| **T1.2** | Run: `kubectl get pods -n kube-system`. Identify `coredns`, `kube-proxy` Pods.     |
| **T1.3** | Run: `kubectl describe node <node-name>`. Find: Capacity, Allocatable, Conditions. |

---

## T1.1 — Verify Worker Nodes

Start Minikube or connect to your EKS cluster.

Run:

```bash
kubectl get nodes
```

Review the output and locate:

```text
STATUS
```

The expected status is:

```text
Ready
```

A simplified example:

```text
NAME              STATUS   ROLES    AGE   VERSION
worker-node-01    Ready    <none>   ...   ...
worker-node-02    Ready    <none>   ...   ...
```

> 💡 **Key Concept:** A `Ready` node indicates that Kubernetes considers the node healthy and available to run workloads.

---

## T1.2 — Inspect Kubernetes System Pods

Run:

```bash
kubectl get pods -n kube-system
```

Identify Pods such as:

```text
coredns
kube-proxy
```

These components are part of the Kubernetes cluster's system infrastructure.

> 📘 **Note:** The exact list of system Pods varies depending on the Kubernetes distribution and environment. EKS and Minikube do not necessarily expose exactly the same Control Plane components as Pods visible to the user.

---

## T1.3 — Describe a Worker Node

First list the nodes:

```bash
kubectl get nodes
```

Select a node and run:

```bash
kubectl describe node <node-name>
```

For example:

```bash
kubectl describe node worker-node-01
```

Find the following sections:

### Capacity

Shows the total resources available on the node.

### Allocatable

Shows the resources available for Kubernetes workloads after reserving resources for system components.

### Conditions

Shows node health conditions.

Review conditions such as:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                      |
| -------- | ----------------------------------------------------------------------------------------------- |
| **T2.1** | Create namespace `dev`: `kubectl create ns dev`. List: `kubectl get ns`.                        |
| **T2.2** | `kubectl api-resources` — list all K8s resource types. Find: `pods`, `deployments`, `services`. |
| **T2.3** | `kubectl explain pod.spec` — read the spec documentation inline.                                |

---

## T2.1 — Create a Namespace

Create a namespace named:

```text
dev
```

Run:

```bash
kubectl create ns dev
```

List all namespaces:

```bash
kubectl get ns
```

Verify that:

```text
dev
```

appears in the output.

---

## T2.2 — Explore Kubernetes API Resources

Run:

```bash
kubectl api-resources
```

This displays the Kubernetes resource types available through the API.

Find the following:

```text
pods
deployments
services
```

Consider how each resource is used:

```text
Pod
  ↓
Runs application containers

Deployment
  ↓
Manages application Pods

Service
  ↓
Provides stable network access to Pods
```

---

## T2.3 — Explore Pod Specification Documentation

Run:

```bash
kubectl explain pod.spec
```

Read the documentation returned directly by `kubectl`.

Explore specific fields if needed:

```bash
kubectl explain pod.spec.containers
```

You can also inspect a container specification:

```bash
kubectl explain pod.spec.containers.image
```

> 💡 **Tip:** `kubectl explain` is a useful way to explore Kubernetes resource definitions without leaving the terminal.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                             |
| -------- | ------------------------------------------------------------------------------------------------------ |
| **T3.1** | Run a one-off pod: `kubectl run debug --image=busybox -it --rm -- sh`. Explore networking from inside. |
| **T3.2** | Get resource usage: `kubectl top nodes && kubectl top pods -A` (enable metrics-server first).          |
| **T3.3** | Describe the kube-scheduler Pod in kube-system — identify its image and startup flags.                 |

---

## T3.1 — Run a Temporary Debug Pod

Run:

```bash
kubectl run debug \
  --image=busybox \
  -it \
  --rm \
  -- sh
```

This creates an interactive temporary Pod.

Inside the Pod, explore the network environment.

For example:

```bash
hostname
```

Inspect network interfaces where available:

```bash
ip addr
```

Inspect the routing table where available:

```bash
ip route
```

Test DNS resolution where the BusyBox image provides the required tools:

```bash
nslookup kubernetes.default
```

> 💡 **Tip:** A temporary debug Pod is useful for troubleshooting Kubernetes DNS, connectivity, and Service access.

When finished:

```bash
exit
```

Because `--rm` was specified, Kubernetes removes the temporary Pod after the session ends.

---

# T3.2 — Monitor Kubernetes Resource Usage

First ensure that **Metrics Server** is enabled and available.

Then run:

```bash
kubectl top nodes
```

This displays resource usage for Worker Nodes.

Run:

```bash
kubectl top pods -A
```

This displays resource usage for Pods across all Namespaces.

The overall workflow is:

```text
Metrics Server
      │
      ▼
Kubernetes Metrics API
      │
      ▼
kubectl top
      │
      ├── Nodes
      └── Pods
```

> ⚠️ **Important:** `kubectl top` requires a functioning Metrics Server. It may not work immediately on a newly created cluster until Metrics Server is installed and healthy.

---

# T3.3 — Inspect the kube-scheduler

Run:

```bash
kubectl get pods -n kube-system
```

Locate the scheduler Pod where it is exposed as a Pod in your environment.

Then describe it:

```bash
kubectl describe pod <kube-scheduler-pod-name> -n kube-system
```

Identify:

### Image

Find the container image used by the scheduler.

### Startup Flags

Review the command and arguments used to start the scheduler.

For example, look for entries under:

```text
Command
Args
```

> ⚠️ **Important:** In managed Kubernetes services such as **Amazon EKS**, Control Plane components such as the scheduler are managed by AWS and are not necessarily visible as ordinary user-accessible Pods in the cluster. This task is therefore more applicable to environments where the Control Plane is directly observable, such as certain self-managed Kubernetes or local configurations.

---

# 🧠 Kubernetes Object Relationships

Understanding the relationship between the core objects is essential.

```text
Deployment
     │
     ▼
ReplicaSet
     │
     ├──────────┐
     ▼          ▼
   Pod        Pod
     │
     ▼
Container(s)

Service
     │
     ▼
Provides stable access
to selected Pods
```

Namespaces provide logical boundaries around these resources:

```text
Kubernetes Cluster
│
├── Namespace: dev
│   ├── Deployment
│   ├── ReplicaSet
│   ├── Pods
│   └── Service
│
└── Namespace: production
    ├── Deployment
    ├── ReplicaSet
    ├── Pods
    └── Service
```

---

# 🔧 Useful kubectl Commands

## View Nodes

```bash
kubectl get nodes
```

## View All Pods

```bash
kubectl get pods -A
```

## View Namespaces

```bash
kubectl get ns
```

## View Services

```bash
kubectl get services
```

## View Deployments

```bash
kubectl get deployments
```

## Describe a Resource

```bash
kubectl describe <resource-type> <resource-name>
```

## View Cluster Information

```bash
kubectl cluster-info
```

## View API Resources

```bash
kubectl api-resources
```

## Explore Kubernetes Documentation

```bash
kubectl explain <resource>
```

---

# 🛡️ Best-Practice Tips

* Use **Namespaces** to logically separate workloads.
* Use **Deployments** rather than manually managing long-running Pods.
* Use **Services** for stable application connectivity.
* Avoid relying on Pod IP addresses because Pods can be recreated.
* Use `kubectl describe` when troubleshooting resource state.
* Use `kubectl explain` to understand Kubernetes resource specifications.
* Monitor node and Pod resource consumption.
* Use temporary debug Pods for controlled troubleshooting.
* Keep Kubernetes versions supported and up to date.
* Apply least-privilege access using Kubernetes RBAC.
* In production, avoid exposing Kubernetes API access unnecessarily.
* For Amazon EKS, remember that AWS manages the Kubernetes Control Plane.

---

# 🧹 Lab Cleanup

If using Minikube, stop the local cluster when finished:

```bash
minikube stop
```

If you want to completely remove the Minikube cluster:

```bash
minikube delete
```

> ⚠️ **Caution:** `minikube delete` removes the local cluster and its workloads. Do not use it if you need to preserve the environment for subsequent labs.

If using EKS, remove lab resources when no longer required to avoid unnecessary AWS charges.

---

# ✅ Lab 07 Completion Checklist

* [ ] Understand Kubernetes Control Plane architecture.
* [ ] Understand Worker Node architecture.
* [ ] Understand API Server, etcd, Scheduler, and Controller Manager.
* [ ] Understand kubelet, kube-proxy, and containerd.
* [ ] Understand the Pod concept.
* [ ] Understand ReplicaSets.
* [ ] Understand Deployments.
* [ ] Understand Services.
* [ ] Understand Namespaces.
* [ ] Start a Minikube cluster or connect to EKS.
* [ ] Verify nodes with `kubectl get nodes`.
* [ ] Confirm node status is `Ready`.
* [ ] Inspect `kube-system` Pods.
* [ ] Create the `dev` Namespace.
* [ ] Explore Kubernetes API resources.
* [ ] Use `kubectl explain`.
* [ ] Run a temporary BusyBox debug Pod.
* [ ] Explore Kubernetes networking from inside a Pod.
* [ ] Use `kubectl top` for resource monitoring.
* [ ] Inspect Kubernetes system components where available.
* [ ] Understand the relationship between Deployment, ReplicaSet, Pod, and Service.
