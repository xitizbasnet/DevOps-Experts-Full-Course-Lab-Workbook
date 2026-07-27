# Lab 08 — Pods, ReplicaSets & Deployments

## 🎯 Objective

Deploy applications using **Kubernetes Deployments**, scale application replicas, perform **rolling updates**, and practice **rollback and self-healing**.

By the end of this lab, you will understand how Kubernetes:

* Creates and manages Pods through Deployments.
* Uses ReplicaSets to maintain the desired number of Pods.
* Scales applications horizontally.
* Performs rolling updates.
* Rolls back failed deployments.
* Recreates failed Pods automatically.
* Uses resource requests and limits.
* Uses liveness and readiness probes.
* Supports Horizontal Pod Autoscaling (HPA).

---

# 🏗️ Deployment Architecture

A Kubernetes Deployment manages a ReplicaSet, which manages Pods.

```text
                    Deployment
                         │
                         ▼
                    ReplicaSet
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        Pod 1          Pod 2          Pod 3
          │              │              │
          ▼              ▼              ▼
       Nginx           Nginx           Nginx
```

For this lab:

```text
Namespace: dev
Application: nginx-app
Initial replicas: 3
Container image: nginx:1.25
```

---

# 📄 Deployment YAML

Create the following file:

```text
nginx-deployment.yaml
```

Use the following Kubernetes Deployment definition:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx-app
  namespace: dev
  labels:
    app: nginx

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
        - name: nginx
          image: nginx:1.25

          ports:
            - containerPort: 80

          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"

            limits:
              cpu: "200m"
              memory: "128Mi"
```

> 💡 **Important:** The `dev` Namespace must exist before applying this Deployment. If you have not created it in Lab 07, run:
>
> ```bash
> kubectl create namespace dev
> ```

---

# 🔍 Understanding the Deployment YAML

## API Version

```yaml
apiVersion: apps/v1
```

The Deployment API version used by modern Kubernetes clusters.

---

## Resource Type

```yaml
kind: Deployment
```

Defines this resource as a Kubernetes Deployment.

---

## Metadata

```yaml
metadata:
  name: nginx-app
  namespace: dev
  labels:
    app: nginx
```

This defines:

* Deployment name: `nginx-app`
* Namespace: `dev`
* Application label: `app: nginx`

---

## Replica Count

```yaml
replicas: 3
```

Kubernetes attempts to maintain **three Pods**.

```text
nginx-app
   │
   ├── Pod 1
   ├── Pod 2
   └── Pod 3
```

---

## Pod Selector

```yaml
selector:
  matchLabels:
    app: nginx
```

The Deployment manages Pods matching:

```text
app=nginx
```

The selector must correspond to the labels defined in the Pod template.

---

# 🔄 Rolling Update Strategy

The Deployment uses:

```yaml
strategy:
  type: RollingUpdate

  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### `maxSurge: 1`

Kubernetes can temporarily create **one additional Pod** above the desired replica count during an update.

For three replicas, the temporary maximum can therefore be:

```text
3 desired Pods
+
1 additional Pod
=
4 Pods
```

### `maxUnavailable: 0`

Kubernetes should not intentionally make any existing replicas unavailable during the rolling update.

This provides a safer update strategy for applications that require continuous availability.

---

# 📦 Container Configuration

```yaml
containers:
  - name: nginx
    image: nginx:1.25
```

The container:

```text
Name: nginx
Image: nginx:1.25
```

---

# 🌐 Container Port

```yaml
ports:
  - containerPort: 80
```

Nginx listens on TCP port `80` inside the container.

> 💡 **Note:** `containerPort` documents the intended container port. It does not by itself expose the application outside the Pod. A Kubernetes `Service` is normally used to provide network access to a Deployment.

---

# 📊 Resource Requests and Limits

The Deployment specifies:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "64Mi"

  limits:
    cpu: "200m"
    memory: "128Mi"
```

### Requests

```text
CPU:    100m
Memory: 64Mi
```

Requests represent the resources Kubernetes uses when scheduling the Pod.

### Limits

```text
CPU:    200m
Memory: 128Mi
```

Limits define the maximum resource consumption allowed for the container.

> 💡 `100m` CPU represents **0.1 CPU core**.

---

# 🚀 Deploy the Application

Apply the Deployment:

```bash
kubectl apply -f nginx-deployment.yaml
```

Expected output will indicate that the Deployment was created or configured.

---

# 🔎 Verify the Deployment

Check the Deployment:

```bash
kubectl get deployments -n dev
```

Check Pods:

```bash
kubectl get pods -n dev
```

For additional Pod details:

```bash
kubectl get pods -n dev -o wide
```

Check the ReplicaSet:

```bash
kubectl get replicasets -n dev
```

The expected relationship is:

```text
Deployment: nginx-app
        │
        ▼
ReplicaSet
        │
   ┌────┼────┐
   ▼    ▼    ▼
  Pod  Pod  Pod
```

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                            |
| -------- | ------------------------------------------------------------------------------------- |
| **T1.1** | Create `nginx-deployment.yaml` from above. Apply it. Verify 3 Pods are running.       |
| **T1.2** | Scale to 5 replicas: `kubectl scale ... --replicas=5`. Watch Pods appear.             |
| **T1.3** | Update image to `nginx:1.26`. Watch the rolling update with `kubectl rollout status`. |

---

## T1.1 — Create and Deploy Nginx

Create:

```text
nginx-deployment.yaml
```

Add the Deployment YAML from this lab.

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

Check the Deployment:

```bash
kubectl get deployment nginx-app -n dev
```

Check the Pods:

```bash
kubectl get pods -n dev
```

You should have three Pods:

```text
nginx-app-xxxxx-xxxxx
nginx-app-xxxxx-xxxxx
nginx-app-xxxxx-xxxxx
```

All should eventually show:

```text
STATUS: Running
```

You can monitor them continuously:

```bash
kubectl get pods -n dev -w
```

Press:

```text
Ctrl+C
```

to stop watching.

---

## T1.2 — Scale the Deployment

Scale the application from 3 to 5 replicas:

```bash
kubectl scale deployment nginx-app --replicas=5 -n dev
```

Verify:

```bash
kubectl get deployment nginx-app -n dev
```

You should see:

```text
READY   UP-TO-DATE   AVAILABLE
5/5
```

Watch the Pods:

```bash
kubectl get pods -n dev -w
```

The Deployment should create two additional Pods.

The desired state becomes:

```text
Deployment
     │
     ▼
ReplicaSet
     │
     ├── Pod 1
     ├── Pod 2
     ├── Pod 3
     ├── Pod 4
     └── Pod 5
```

---

## T1.3 — Perform a Rolling Update

Update the Nginx image:

```bash
kubectl set image deployment/nginx-app nginx=nginx:1.26 -n dev
```

Monitor the rollout:

```bash
kubectl rollout status deployment/nginx-app -n dev
```

You can also watch Pods:

```bash
kubectl get pods -n dev -w
```

Inspect the Deployment:

```bash
kubectl describe deployment nginx-app -n dev
```

The Deployment creates a new ReplicaSet for the new version while gradually replacing Pods from the previous ReplicaSet.

---

# 🔄 Rolling Update Workflow

The update process is conceptually:

```text
Before Update

ReplicaSet v1
 ├── Pod v1
 ├── Pod v1
 ├── Pod v1
 ├── Pod v1
 └── Pod v1


During Update

ReplicaSet v1          ReplicaSet v2
 ├── Pod v1             └── Pod v2
 ├── Pod v1
 ├── Pod v1
 └── Pod v1


After Update

ReplicaSet v2
 ├── Pod v2
 ├── Pod v2
 ├── Pod v2
 ├── Pod v2
 └── Pod v2
```

The exact transition depends on the configured rolling update parameters.

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                       |
| -------- | ------------------------------------------------------------------------------------------------ |
| **T2.1** | Delete one Pod manually. Kubernetes should auto-recreate it. Time how fast it recovers.          |
| **T2.2** | Roll back to the previous version with `kubectl rollout undo`. Verify the old image is restored. |
| **T2.3** | Deploy a second app (`httpd:2.4`) in the same `dev` Namespace. Both should coexist.              |

---

## T2.1 — Test Kubernetes Self-Healing

List Pods:

```bash
kubectl get pods -n dev
```

Select one Pod and delete it:

```bash
kubectl delete pod <pod-name> -n dev
```

Immediately monitor the Deployment:

```bash
kubectl get pods -n dev -w
```

Kubernetes should create a replacement Pod.

The desired state remains:

```text
Desired replicas: 5
```

Even though you manually removed one Pod, the ReplicaSet attempts to restore the desired count.

> 💡 **Key Concept:** Kubernetes controllers continuously compare the **desired state** with the **actual state** and take corrective action when they differ.

---

## T2.2 — Roll Back the Deployment

View rollout history:

```bash
kubectl rollout history deployment/nginx-app -n dev
```

Roll back:

```bash
kubectl rollout undo deployment/nginx-app -n dev
```

Monitor the rollback:

```bash
kubectl rollout status deployment/nginx-app -n dev
```

Check the Pods:

```bash
kubectl get pods -n dev
```

Verify the image:

```bash
kubectl describe deployment nginx-app -n dev
```

You can also inspect the image directly:

```bash
kubectl get deployment nginx-app -n dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

The previous image should be restored.

---

# 🕘 Deployment Rollout History

Check history:

```bash
kubectl rollout history deployment/nginx-app -n dev
```

You can inspect a specific revision:

```bash
kubectl rollout history deployment/nginx-app --revision=<revision-number> -n dev
```

Rollback to a specific revision:

```bash
kubectl rollout undo deployment/nginx-app \
  --to-revision=<revision-number> \
  -n dev
```

---

## T2.3 — Deploy a Second Application

Create a second Deployment for Apache HTTP Server.

Create:

```text
httpd-deployment.yaml
```

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: httpd-app
  namespace: dev

spec:
  replicas: 2

  selector:
    matchLabels:
      app: httpd

  template:
    metadata:
      labels:
        app: httpd

    spec:
      containers:
        - name: httpd
          image: httpd:2.4
          ports:
            - containerPort: 80
```

Apply it:

```bash
kubectl apply -f httpd-deployment.yaml
```

Verify both applications:

```bash
kubectl get deployments -n dev
```

Check all Pods:

```bash
kubectl get pods -n dev
```

The namespace should now contain both:

```text
nginx-app
httpd-app
```

Conceptually:

```text
Namespace: dev
│
├── nginx-app
│   ├── Pod
│   ├── Pod
│   └── ...
│
└── httpd-app
    ├── Pod
    └── Pod
```

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                      |
| -------- | ----------------------------------------------------------------------------------------------- |
| **T3.1** | Set up HPA: `kubectl autoscale deployment nginx-app --min=2 --max=10 --cpu-percent=50`          |
| **T3.2** | Simulate Pod failure: set wrong image name. Observe `ImagePullBackOff`. Fix it.                 |
| **T3.3** | Add liveness and readiness probes to the Nginx Deployment YAML. Apply and verify health checks. |

---

# T3.1 — Configure Horizontal Pod Autoscaling

Create an HPA for the Nginx Deployment:

```bash
kubectl autoscale deployment nginx-app \
  --min=2 \
  --max=10 \
  --cpu-percent=50 \
  -n dev
```

Verify the HPA:

```bash
kubectl get hpa -n dev
```

For more details:

```bash
kubectl describe hpa nginx-app -n dev
```

The target architecture is:

```text
                    HPA
                     │
             Monitors CPU usage
                     │
                     ▼
                Deployment
                     │
            ┌────────┼────────┐
            ▼        ▼        ▼
           Pod      Pod      Pod
```

The HPA can adjust the number of replicas between:

```text
Minimum: 2 Pods
Maximum: 10 Pods
```

based on CPU utilization.

> ⚠️ **Important:** HPA requires resource metrics, typically provided through Metrics Server. Ensure Metrics Server is installed and functioning before testing autoscaling.

> 💡 **Important:** Because the Deployment already specifies CPU requests, Kubernetes has the baseline needed for percentage-based CPU utilization calculations.

---

# T3.2 — Simulate an Image Failure

Change the Nginx image to an invalid image name.

For example:

```bash
kubectl set image deployment/nginx-app \
  nginx=nginx:invalid-version \
  -n dev
```

Monitor the Pods:

```bash
kubectl get pods -n dev -w
```

You may see:

```text
ErrImagePull
```

and eventually:

```text
ImagePullBackOff
```

Inspect the failing Pod:

```bash
kubectl describe pod <pod-name> -n dev
```

Look at the:

```text
Events
```

section.

This commonly provides useful information about why the image could not be pulled.

---

## Fix the Image

Restore a valid image:

```bash
kubectl set image deployment/nginx-app \
  nginx=nginx:1.26 \
  -n dev
```

Monitor the rollout:

```bash
kubectl rollout status deployment/nginx-app -n dev
```

Verify:

```bash
kubectl get pods -n dev
```

All expected Pods should eventually return to:

```text
Running
```

---

# T3.3 — Add Liveness and Readiness Probes

Add health checks to the Nginx container.

Update the container definition:

```yaml
containers:
  - name: nginx
    image: nginx:1.26

    ports:
      - containerPort: 80

    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"

      limits:
        cpu: "200m"
        memory: "128Mi"

    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10

    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

---

# ❤️ Liveness Probe

The liveness probe determines whether the container is still functioning.

```yaml
livenessProbe:
  httpGet:
    path: /
    port: 80
```

If the container repeatedly fails its liveness check, Kubernetes can restart the container.

Conceptually:

```text
Container
    │
    ▼
Liveness Check
    │
    ├── Healthy → Continue
    │
    └── Failed → Kubernetes may restart container
```

---

# 🟢 Readiness Probe

The readiness probe determines whether the application is ready to receive traffic.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 80
```

If the readiness check fails, Kubernetes can mark the Pod as not ready.

This is particularly important when a Service routes traffic to multiple Pods.

```text
                 Service
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
        Ready     Ready     Not Ready
         Pod       Pod         Pod
          │         │           ✕
          └─────────┴───────────┘
              Traffic
```

> 💡 **Key Difference:**
> **Liveness** asks: *"Should this container be restarted?"*
> **Readiness** asks: *"Should this Pod receive application traffic?"*

---

# 🚀 Apply the Updated Deployment

Apply the YAML:

```bash
kubectl apply -f nginx-deployment.yaml
```

Check Pod status:

```bash
kubectl get pods -n dev
```

Check detailed probe configuration:

```bash
kubectl describe pod <pod-name> -n dev
```

Look for:

```text
Liveness
Readiness
```

---

# 🔍 Useful Deployment Commands

## List Deployments

```bash
kubectl get deployments -n dev
```

## List ReplicaSets

```bash
kubectl get replicasets -n dev
```

## List Pods

```bash
kubectl get pods -n dev
```

## Show Pods with Node Information

```bash
kubectl get pods -n dev -o wide
```

## Describe Deployment

```bash
kubectl describe deployment nginx-app -n dev
```

## Scale Deployment

```bash
kubectl scale deployment nginx-app --replicas=5 -n dev
```

## Change Container Image

```bash
kubectl set image deployment/nginx-app \
  nginx=nginx:1.26 \
  -n dev
```

## Monitor Rollout

```bash
kubectl rollout status deployment/nginx-app -n dev
```

## View Rollout History

```bash
kubectl rollout history deployment/nginx-app -n dev
```

## Roll Back

```bash
kubectl rollout undo deployment/nginx-app -n dev
```

## Restart Deployment

```bash
kubectl rollout restart deployment/nginx-app -n dev
```

## Delete Deployment

```bash
kubectl delete deployment nginx-app -n dev
```

---

# 🧠 Desired State vs. Actual State

One of the most important Kubernetes concepts in this lab is **desired state management**.

Suppose the Deployment specifies:

```yaml
replicas: 5
```

Kubernetes continuously attempts to maintain:

```text
Desired State
     │
     ▼
5 Pods
```

If one Pod fails:

```text
Desired: 5
Actual:  4
```

The ReplicaSet detects the difference and creates a replacement:

```text
Desired: 5
Actual:  5
```

This controller-based architecture enables Kubernetes to provide **self-healing behavior**.

---

# 🔄 Deployment Lifecycle

The complete workflow practiced in this lab is:

```text
Create YAML
    │
    ▼
kubectl apply
    │
    ▼
Deployment
    │
    ▼
ReplicaSet
    │
    ▼
Pods
    │
    ├── Scale
    │
    ├── Update
    │
    ├── Monitor
    │
    ├── Self-Heal
    │
    └── Rollback
```

---

# 🛡️ Best-Practice Tips

* Use **Deployments** to manage long-running applications.
* Do not manually create individual Pods for production workloads.
* Define appropriate CPU and memory **requests**.
* Define reasonable CPU and memory **limits**.
* Use **readiness probes** to prevent traffic from reaching unready applications.
* Use **liveness probes** to detect unhealthy containers.
* Use rolling updates to reduce application downtime.
* Maintain a rollback strategy for failed deployments.
* Avoid using the `latest` image tag for production workloads.
* Prefer explicit and controlled image versions such as `nginx:1.26`.
* Use meaningful labels and selectors.
* Monitor rollout status after every application update.
* Test failure scenarios before deploying to production.
* Use HPA when workloads require dynamic scaling.
* Ensure Metrics Server is available before testing HPA.
* Use namespaces to separate application environments.

---

# 🧪 Troubleshooting Quick Reference

### Deployment not progressing

```bash
kubectl describe deployment nginx-app -n dev
```

### Pod is not running

```bash
kubectl get pods -n dev
kubectl describe pod <pod-name> -n dev
```

### View Pod events

```bash
kubectl describe pod <pod-name> -n dev
```

### Check container logs

```bash
kubectl logs <pod-name> -n dev
```

### Follow container logs

```bash
kubectl logs -f <pod-name> -n dev
```

### Check ReplicaSets

```bash
kubectl get rs -n dev
```

### Check rollout status

```bash
kubectl rollout status deployment/nginx-app -n dev
```

### Check rollout history

```bash
kubectl rollout history deployment/nginx-app -n dev
```

### Roll back

```bash
kubectl rollout undo deployment/nginx-app -n dev
```

---

# 🧹 Lab Cleanup

Remove the Nginx Deployment:

```bash
kubectl delete deployment nginx-app -n dev
```

Remove the Apache Deployment:

```bash
kubectl delete deployment httpd-app -n dev
```

Remove the HPA if created:

```bash
kubectl delete hpa nginx-app -n dev
```

Verify:

```bash
kubectl get deployments -n dev
kubectl get pods -n dev
kubectl get hpa -n dev
```

> ⚠️ **Caution:** Do not delete the `dev` Namespace if you intend to use it for subsequent labs.

---

# ✅ Lab 08 Completion Checklist

* [ ] Create `nginx-deployment.yaml`.
* [ ] Deploy Nginx using a Kubernetes Deployment.
* [ ] Verify three initial replicas.
* [ ] Understand the Deployment → ReplicaSet → Pod relationship.
* [ ] Scale the Deployment from 3 to 5 replicas.
* [ ] Observe new Pods being created.
* [ ] Perform a rolling update from `nginx:1.25` to `nginx:1.26`.
* [ ] Monitor the rolling update using `kubectl rollout status`.
* [ ] Delete a Pod manually.
* [ ] Observe Kubernetes automatically recreate the Pod.
* [ ] View rollout history.
* [ ] Perform a Deployment rollback.
* [ ] Verify the previous image version.
* [ ] Deploy `httpd:2.4` in the `dev` Namespace.
* [ ] Run Nginx and Apache workloads simultaneously.
* [ ] Configure HPA.
* [ ] Verify HPA configuration.
* [ ] Simulate an invalid container image.
* [ ] Observe `ErrImagePull` / `ImagePullBackOff`.
* [ ] Correct the failed image deployment.
* [ ] Configure a liveness probe.
* [ ] Configure a readiness probe.
* [ ] Verify health checks using `kubectl describe`.
* [ ] Understand Kubernetes desired-state reconciliation.
* [ ] Understand Deployment self-healing and rollback.
