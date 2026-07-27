# Lab 09 — Services & Load Balancing

## 🎯 Objective

Expose Kubernetes Deployments internally and externally using Kubernetes **Services**.

In this lab, you will work with:

* **ClusterIP** — internal cluster access.
* **NodePort** — external access through a Kubernetes node.
* **LoadBalancer** — external access through a cloud load balancer.
* Kubernetes Service selectors and endpoints.
* Service-based load balancing across multiple Pods.
* Blue-green application switching.
* `ExternalName` Services.
* Headless Services and DNS-based Pod discovery.

---

# 🏗️ Kubernetes Service Architecture

A Kubernetes Service provides a stable network endpoint for a group of Pods.

Pods are temporary and their IP addresses can change. A Service provides a stable way for clients to reach those Pods.

```text
                    Kubernetes Service
                           │
                    selector: app=nginx
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Pod 1          Pod 2          Pod 3
        10.0.1.10      10.0.1.11      10.0.1.12
```

The Service identifies Pods using labels:

```yaml
selector:
  app: nginx
```

---

# 🌐 Service Types

Kubernetes provides several Service types.

| Service Type     | Purpose                                    | Typical Access              |
| ---------------- | ------------------------------------------ | --------------------------- |
| **ClusterIP**    | Internal cluster access                    | Pods inside cluster         |
| **NodePort**     | Expose service through node IP and port    | External clients            |
| **LoadBalancer** | Provision external cloud load balancer     | Internet / external clients |
| **ExternalName** | Map a Service name to an external DNS name | External services           |
| **Headless**     | Return individual Pod IPs through DNS      | Direct Pod discovery        |

---

# 🔵 ClusterIP

`ClusterIP` is the default Kubernetes Service type.

It provides an internal virtual IP that can be accessed by workloads inside the Kubernetes cluster.

```text
Pod A
  │
  │ HTTP
  ▼
ClusterIP Service
  │
  ├── Pod 1
  ├── Pod 2
  └── Pod 3
```

It is useful for internal application communication such as:

```text
Frontend → Backend
Backend → Database
Application → Internal API
```

---

# 🟠 NodePort

A `NodePort` exposes a Service on a port across each Kubernetes Node.

For this lab:

```text
NodePort: 30080
```

Traffic can be sent to:

```text
http://<NODE-IP>:30080
```

The NodePort range is generally:

```text
30000–32767
```

---

# 🟢 LoadBalancer

A `LoadBalancer` Service requests an external load balancer from the underlying cloud provider.

For example, in AWS environments, the exact load-balancing resource depends on the Kubernetes Service configuration, AWS integration, and controller in use.

> ⚠️ **Important:** A Kubernetes `Service` of type `LoadBalancer` does **not inherently create an AWS Application Load Balancer (ALB)**. On EKS, the AWS Load Balancer Controller is typically used when you specifically want an ALB through an `Ingress`. A `LoadBalancer` Service commonly provisions an AWS Network Load Balancer (NLB), depending on configuration and cluster setup.

---

# 📄 Service YAML — NodePort

Create:

```text
service.yaml
```

Use:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-svc
  namespace: dev

spec:
  type: NodePort

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

---

# 🔍 Understanding the Service YAML

## API Version

```yaml
apiVersion: v1
```

Services are part of Kubernetes core API version `v1`.

---

## Resource Type

```yaml
kind: Service
```

Defines the Kubernetes resource as a Service.

---

## Metadata

```yaml
metadata:
  name: nginx-svc
  namespace: dev
```

The Service is named:

```text
nginx-svc
```

and exists in:

```text
dev
```

---

## Service Type

```yaml
type: NodePort
```

This exposes the Service through a NodePort.

Other available types include:

```text
ClusterIP
NodePort
LoadBalancer
ExternalName
```

---

## Selector

```yaml
selector:
  app: nginx
```

This is extremely important.

The Service routes traffic to Pods with the label:

```text
app=nginx
```

The Nginx Deployment from Lab 08 uses:

```yaml
labels:
  app: nginx
```

Therefore, the Service can identify those Pods.

---

# 🔌 Service Ports

```yaml
ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

These represent three different concepts.

### Service Port

```text
port: 80
```

The port exposed by the Service.

### Target Port

```text
targetPort: 80
```

The port where Nginx is listening inside the selected Pods.

### Node Port

```text
nodePort: 30080
```

The port exposed on Kubernetes Nodes.

Traffic flow:

```text
Client
   │
   │ :30080
   ▼
Kubernetes Node
   │
   │ Service :80
   ▼
nginx-svc
   │
   │ targetPort :80
   ▼
Nginx Pod :80
```

---

# 🚀 Deploy the Service

Apply the Service:

```bash
kubectl apply -f service.yaml
```

Check Services:

```bash
kubectl get svc -n dev
```

Describe the Service:

```bash
kubectl describe svc nginx-svc -n dev
```

---

# 🔎 Check Service Endpoints

The Service should identify the Nginx Pods using the selector.

Run:

```bash
kubectl describe svc nginx-svc -n dev
```

Look for:

```text
Endpoints
```

You should see Pod IP addresses and ports.

You can also use:

```bash
kubectl get endpoints nginx-svc -n dev
```

On newer Kubernetes versions, you can inspect EndpointSlices:

```bash
kubectl get endpointslices -n dev
```

> 💡 **Key Concept:** If a Service has no endpoints, check whether the Service selector matches the Pod labels.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                |
| -------- | ----------------------------------------------------------------------------------------- |
| **T1.1** | Create a NodePort Service for `nginx-app`. Access it on `:30080`.                         |
| **T1.2** | `kubectl describe svc nginx-svc` — find **Endpoints** (list of Pod IPs).                  |
| **T1.3** | Create a ClusterIP Service for a backend Pod. Verify only same-cluster Pods can reach it. |

---

# T1.1 — Create NodePort Service

Create:

```text
service.yaml
```

Add the NodePort Service definition.

Apply:

```bash
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get svc -n dev
```

Expected output will contain information similar to:

```text
NAME        TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)
nginx-svc   NodePort   10.x.x.x       <none>        80:30080/TCP
```

Get your node information:

```bash
kubectl get nodes -o wide
```

Find the appropriate Node IP.

Then access:

```text
http://<NODE-IP>:30080
```

Or test using:

```bash
curl http://<NODE-IP>:30080
```

> ⚠️ **AWS EC2 Security Group Note:** If you are using EKS with NodePort access from outside the cluster, ensure the relevant security-group/network rules permit TCP `30080` where required. Avoid opening broad access unnecessarily.

---

# T1.2 — Find Service Endpoints

Run:

```bash
kubectl describe svc nginx-svc -n dev
```

Find:

```text
Endpoints
```

You should see one endpoint for each selected Pod.

For example:

```text
Endpoints: 10.0.1.10:80,10.0.1.11:80,10.0.1.12:80
```

You can also run:

```bash
kubectl get endpoints nginx-svc -n dev
```

Compare the endpoint addresses with:

```bash
kubectl get pods -n dev -o wide
```

This demonstrates how the Service connects to its selected Pods.

---

# T1.3 — Create a ClusterIP Service

Create:

```text
backend-service.yaml
```

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: backend-svc
  namespace: dev

spec:
  type: ClusterIP

  selector:
    app: backend

  ports:
    - port: 80
      targetPort: 80
```

> 💡 Make sure a backend Pod with the label `app: backend` exists before testing the Service.

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc -n dev
```

A ClusterIP Service is intended for internal cluster communication.

Conceptually:

```text
Inside Cluster
     │
     ▼
backend-svc
     │
     ▼
Backend Pods

Outside Cluster
     │
     X
     │
ClusterIP not directly externally accessible
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                       |
| -------- | ------------------------------------------------------------------------------------------------ |
| **T2.1** | Change the Service to `LoadBalancer` on EKS. Watch `EXTERNAL-IP` appear. Test in a browser.      |
| **T2.2** | Run curl in a BusyBox Pod: `kubectl run test --image=busybox -it --rm -- wget -qO- nginx-svc`    |
| **T2.3** | Scale Nginx to 6 replicas. Reload the page repeatedly — observe traffic distributed across Pods. |

---

# T2.1 — Change Service to LoadBalancer

Edit:

```text
service.yaml
```

Change:

```yaml
type: NodePort
```

to:

```yaml
type: LoadBalancer
```

Your Service can be:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-svc
  namespace: dev

spec:
  type: LoadBalancer

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f service.yaml
```

Check:

```bash
kubectl get svc nginx-svc -n dev
```

Watch the Service:

```bash
kubectl get svc nginx-svc -n dev -w
```

On a correctly configured EKS environment, the external address should eventually appear.

Then test:

```text
http://<EXTERNAL-IP>
```

Or:

```bash
curl http://<EXTERNAL-IP>
```

> ⚠️ **AWS Note:** Provisioning the external load balancer can take several minutes. It may also require appropriate AWS permissions and cluster/controller configuration.

---

# T2.2 — Test Internal DNS

Run a temporary BusyBox Pod:

```bash
kubectl run test \
  --image=busybox \
  -it \
  --rm \
  -- wget -qO- nginx-svc
```

The command:

```text
nginx-svc
```

is resolved through Kubernetes DNS.

The traffic flow is:

```text
BusyBox Pod
     │
     │ DNS: nginx-svc
     ▼
Kubernetes DNS
     │
     ▼
nginx-svc
     │
     ▼
Nginx Pods
```

> 💡 **Key Concept:** Kubernetes Services provide DNS names that allow workloads to communicate without knowing individual Pod IP addresses.

---

# T2.3 — Scale Nginx to Six Replicas

Scale the Nginx Deployment:

```bash
kubectl scale deployment nginx-app --replicas=6 -n dev
```

Verify:

```bash
kubectl get pods -n dev -o wide
```

Check Service endpoints:

```bash
kubectl get endpoints nginx-svc -n dev
```

You should now see endpoints corresponding to the Nginx Pods selected by the Service.

Access the application repeatedly:

```text
http://<NODE-IP>:30080
```

or through your LoadBalancer if configured.

The Service distributes traffic across available backend Pods.

```text
                  nginx-svc
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      Pod 1        Pod 2        Pod 3
        │            │            │
        └────────────┼────────────┘
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      Pod 4        Pod 5        Pod 6
```

> 💡 **Note:** With identical Nginx content, repeatedly refreshing the page may not visibly demonstrate which Pod handled each request. To observe actual distribution, you can later customize the Pods to return unique identifiers or inspect application/access logs.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                                    |
| -------- | ------------------------------------------------------------------------------------------------------------- |
| **T3.1** | Deploy 2 versions of the app (`v1` and `v2`). Create Services for each. Implement a simple blue-green switch. |
| **T3.2** | Implement an `ExternalName` Service pointing to an external API (e.g., `api.github.com`).                     |
| **T3.3** | Create a Headless Service (`clusterIP: None`). Observe DNS returning individual Pod IPs.                      |

---

# T3.1 — Blue-Green Deployment with Services

Blue-green deployment maintains two application versions:

```text
Blue = Current Production
Green = New Version
```

A simple architecture:

```text
                   nginx-svc
                       │
                       ▼
                 Active Version
                       │
              ┌────────┴────────┐
              ▼                 ▼
           Blue v1            Green v2
```

Create two Deployments with different labels.

### Blue Version

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: app-v1
  namespace: dev

spec:
  replicas: 3

  selector:
    matchLabels:
      app: demo
      version: v1

  template:
    metadata:
      labels:
        app: demo
        version: v1

    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
```

### Green Version

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: app-v2
  namespace: dev

spec:
  replicas: 3

  selector:
    matchLabels:
      app: demo
      version: v2

  template:
    metadata:
      labels:
        app: demo
        version: v2

    spec:
      containers:
        - name: nginx
          image: nginx:1.26
          ports:
            - containerPort: 80
```

---

## Blue-Green Service

Create a Service that initially points to v1:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: demo-svc
  namespace: dev

spec:
  selector:
    app: demo
    version: v1

  ports:
    - port: 80
      targetPort: 80
```

The Service currently sends traffic to:

```text
version=v1
```

To switch production traffic to v2, change:

```yaml
version: v1
```

to:

```yaml
version: v2
```

Apply the change:

```bash
kubectl apply -f demo-service.yaml
```

Traffic now moves from:

```text
Service
   │
   ▼
v1
```

to:

```text
Service
   │
   ▼
v2
```

This is a simple **blue-green switch**.

> 💡 **Best Practice:** In a production environment, validate the green environment thoroughly before changing the Service selector.

---

# T3.2 — ExternalName Service

An `ExternalName` Service maps a Kubernetes Service name to an external DNS name.

Create:

```text
external-service.yaml
```

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: github-api
  namespace: dev

spec:
  type: ExternalName
  externalName: api.github.com
```

Apply:

```bash
kubectl apply -f external-service.yaml
```

Check:

```bash
kubectl get svc github-api -n dev
```

The Kubernetes Service does not create Pods or a ClusterIP for this configuration. Instead, Kubernetes DNS provides the external DNS mapping.

Conceptually:

```text
Application Pod
      │
      │ github-api
      ▼
Kubernetes DNS
      │
      ▼
api.github.com
      │
      ▼
GitHub API
```

Test DNS resolution from a suitable test Pod:

```bash
kubectl run dns-test \
  --image=busybox \
  -it \
  --rm \
  -- nslookup github-api.dev.svc.cluster.local
```

> ⚠️ **Note:** ExternalName is a DNS aliasing mechanism. It does not provide the same proxying, load balancing, health checking, or security controls as a conventional Kubernetes Service backed by Pods.

---

# T3.3 — Create a Headless Service

A **Headless Service** does not allocate a ClusterIP.

Set:

```yaml
clusterIP: None
```

Create:

```text
headless-service.yaml
```

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-headless
  namespace: dev

spec:
  clusterIP: None

  selector:
    app: nginx

  ports:
    - port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f headless-service.yaml
```

Check:

```bash
kubectl get svc nginx-headless -n dev
```

You should see:

```text
CLUSTER-IP
None
```

---

# 🔎 Test Headless Service DNS

Run a DNS test Pod:

```bash
kubectl run dns-test \
  --image=busybox \
  -it \
  --rm \
  -- nslookup nginx-headless.dev.svc.cluster.local
```

The DNS response can contain the individual Pod IP addresses rather than one virtual Service IP.

Conceptually:

```text
nginx-headless.dev.svc.cluster.local
             │
       Kubernetes DNS
             │
     ┌───────┼───────┐
     ▼       ▼       ▼
   Pod IP  Pod IP  Pod IP
```

This is useful for applications that need direct Pod discovery, such as certain stateful or distributed applications.

---

# 🔬 Compare Normal vs. Headless Services

## Normal ClusterIP Service

```text
DNS
 │
 ▼
ClusterIP
 │
 ├── Pod
 ├── Pod
 └── Pod
```

## Headless Service

```text
DNS
 │
 ├── Pod IP
 ├── Pod IP
 └── Pod IP
```

The key difference is:

```yaml
# Normal Service
clusterIP: <allocated IP>

# Headless Service
clusterIP: None
```

---

# 🌐 Kubernetes Service DNS

Within a Namespace, a Service can normally be reached using:

```text
service-name
```

For example:

```text
nginx-svc
```

The fully qualified Service DNS name follows the Kubernetes pattern:

```text
<service>.<namespace>.svc.cluster.local
```

For this lab:

```text
nginx-svc.dev.svc.cluster.local
```

From another Pod:

```bash
curl http://nginx-svc
```

or:

```bash
curl http://nginx-svc.dev.svc.cluster.local
```

---

# 🔄 Service Traffic Flow

For a NodePort Service:

```text
External Client
      │
      │ :30080
      ▼
Kubernetes Node
      │
      ▼
NodePort Service
      │
      ▼
ClusterIP / Service routing
      │
      ├──────────┬──────────┐
      ▼          ▼          ▼
    Pod 1      Pod 2      Pod 3
```

For an internal ClusterIP Service:

```text
Client Pod
    │
    │ nginx-svc:80
    ▼
ClusterIP Service
    │
    ├── Pod 1
    ├── Pod 2
    └── Pod 3
```

---

# 🔧 Useful Service Commands

## List Services

```bash
kubectl get svc -n dev
```

## Detailed Service Information

```bash
kubectl describe svc nginx-svc -n dev
```

## View Service Endpoints

```bash
kubectl get endpoints nginx-svc -n dev
```

## View EndpointSlices

```bash
kubectl get endpointslices -n dev
```

## View Service as YAML

```bash
kubectl get svc nginx-svc -n dev -o yaml
```

## View Pods and Their IPs

```bash
kubectl get pods -n dev -o wide
```

## Test Service from a Temporary Pod

```bash
kubectl run test \
  --image=busybox \
  -it \
  --rm \
  -- wget -qO- nginx-svc
```

## Delete a Service

```bash
kubectl delete svc nginx-svc -n dev
```

---

# 🛡️ Best-Practice Tips

* Use **ClusterIP** for services that should only be accessible inside the cluster.
* Use **NodePort** mainly when you specifically need node-level exposure or for learning/testing.
* For production cloud workloads, carefully choose between `LoadBalancer` Services and `Ingress`/AWS Load Balancer Controller.
* Avoid exposing unnecessary ports publicly.
* Restrict AWS Security Group access to trusted networks where possible.
* Use meaningful Service names.
* Ensure Service selectors correctly match Pod labels.
* Check Service endpoints when troubleshooting connectivity.
* Use Kubernetes DNS names instead of hard-coded Pod IP addresses.
* Remember that Pod IPs can change when Pods are recreated.
* Use Headless Services when direct Pod discovery is required.
* Use `ExternalName` when DNS aliasing is sufficient; do not treat it as a full proxy.
* Test load balancing with identifiable application responses when possible.
* Use blue-green deployments only after validating the new application version.
* Monitor cloud load balancer provisioning and associated AWS resources.

---

# 🚨 Troubleshooting

## Service Has No Endpoints

Check:

```bash
kubectl describe svc nginx-svc -n dev
```

Then check Pod labels:

```bash
kubectl get pods -n dev --show-labels
```

Compare them with the Service selector:

```bash
kubectl get svc nginx-svc -n dev -o yaml
```

For example, the Service expects:

```yaml
selector:
  app: nginx
```

Therefore, selected Pods must contain:

```yaml
labels:
  app: nginx
```

---

## NodePort Is Not Accessible

Check the Service:

```bash
kubectl get svc nginx-svc -n dev
```

Check Nodes:

```bash
kubectl get nodes -o wide
```

Check endpoints:

```bash
kubectl get endpoints nginx-svc -n dev
```

For AWS environments, verify the relevant:

* Security Groups
* Network ACLs
* Routing
* Subnets
* Node health
* Kubernetes Service configuration

---

## LoadBalancer Has No External Address

Check:

```bash
kubectl get svc nginx-svc -n dev
```

Then:

```bash
kubectl describe svc nginx-svc -n dev
```

Review the:

```text
Events
```

section.

Also verify that the cluster has the appropriate AWS integration, permissions, subnet configuration, and controller configuration required for the selected load-balancing architecture.

---

## Service DNS Does Not Resolve

Check CoreDNS:

```bash
kubectl get pods -n kube-system
```

Look for:

```text
coredns
```

Test from a Pod:

```bash
kubectl run dns-test \
  --image=busybox \
  -it \
  --rm \
  -- nslookup nginx-svc
```

---

# 🧹 Lab Cleanup

Remove the Services created during the lab:

```bash
kubectl delete svc nginx-svc -n dev
kubectl delete svc backend-svc -n dev
kubectl delete svc demo-svc -n dev
kubectl delete svc github-api -n dev
kubectl delete svc nginx-headless -n dev
```

Remove the challenge Deployments if created:

```bash
kubectl delete deployment app-v1 -n dev
kubectl delete deployment app-v2 -n dev
```

Verify:

```bash
kubectl get svc -n dev
kubectl get deployments -n dev
kubectl get pods -n dev
```

> ⚠️ **AWS Cost Warning:** If you created a `LoadBalancer` Service on EKS, deleting the Kubernetes Service is important because it allows the associated cloud load-balancing resources to be cleaned up.

---

# ✅ Lab 09 Completion Checklist

* [ ] Understand the purpose of Kubernetes Services.
* [ ] Understand `ClusterIP`.
* [ ] Understand `NodePort`.
* [ ] Understand `LoadBalancer`.
* [ ] Understand `ExternalName`.
* [ ] Understand Headless Services.
* [ ] Create a NodePort Service for `nginx-app`.
* [ ] Access Nginx through NodePort `30080`.
* [ ] Inspect Service endpoints.
* [ ] Create a ClusterIP backend Service.
* [ ] Test internal Service connectivity.
* [ ] Change a Service to `LoadBalancer`.
* [ ] Verify the external load balancer address.
* [ ] Test Kubernetes Service DNS.
* [ ] Scale Nginx to six replicas.
* [ ] Observe Service endpoints change with replica count.
* [ ] Understand Service-based traffic distribution.
* [ ] Deploy two application versions.
* [ ] Create a simple blue-green Service switch.
* [ ] Create an `ExternalName` Service.
* [ ] Point the `ExternalName` Service to `api.github.com`.
* [ ] Create a Headless Service.
* [ ] Verify `clusterIP: None`.
* [ ] Observe individual Pod IPs through Headless Service DNS.
* [ ] Practice Service troubleshooting.
* [ ] Clean up cloud resources after the lab.
