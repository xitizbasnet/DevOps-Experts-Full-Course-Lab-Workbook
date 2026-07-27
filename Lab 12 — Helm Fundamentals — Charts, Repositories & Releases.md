# Session 5 — Package Management & Helm

## Lab 12 — Helm Fundamentals — Charts, Repositories & Releases

> **🎯 Objective:** Install Helm, use community charts, and understand Helm chart structure and release management.

---

## 🧭 Overview

In this lab, you will learn how **Helm** simplifies Kubernetes application deployment and package management.

You will work with:

* Helm 3 installation
* Helm chart repositories
* Community charts
* Helm releases
* Chart structure
* `values.yaml`
* Helm upgrades and rollbacks
* Custom Helm charts
* Chart dependencies
* ChartMuseum

### 🧠 What is Helm?

**Helm** is a package manager for Kubernetes. It allows you to define, install, upgrade, and manage Kubernetes applications using reusable **Charts**.

A useful mental model is:

```text
Helm Chart
    │
    ├── Templates
    ├── Default Values
    ├── Metadata
    └── Dependencies
          │
          ▼
     Helm Release
          │
          ▼
     Kubernetes Resources
          │
          ├── Deployment
          ├── Service
          ├── ConfigMap
          ├── Secret
          └── Other Resources
```

---

# ⚙️ Installation & Setup

## Install Helm v3

Run:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify the installation:

```bash
helm version
```

---

## 📦 Add Chart Repositories

Add the Stable repository:

```bash
helm repo add stable https://charts.helm.sh/stable
```

Add the Bitnami repository:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Add the Prometheus Community repository:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
```

Update your local repository index:

```bash
helm repo update
```

> 💡 **Note:** Helm repositories and chart availability can change over time. If a repository or chart used in this lab is no longer maintained, use the current official repository recommended by the chart publisher.

---

# 🔎 Search Helm Charts

Search the locally configured repositories:

```bash
helm search repo nginx
```

Search Helm Hub:

```bash
helm search hub wordpress
```

---

# 🚀 Install a Helm Release

Install an Nginx release:

```bash
helm install my-nginx bitnami/nginx
```

Install WordPress into a dedicated namespace:

```bash
helm install my-wp bitnami/wordpress \
  --namespace wordpress \
  --create-namespace
```

> **💡 Key Concept:** `my-nginx` and `my-wp` are **release names**. A release is an installed instance of a Helm chart.

---

# 📋 Manage Helm Releases

List releases across all namespaces:

```bash
helm list -A
```

Check the status of a release:

```bash
helm status my-nginx
```

Remove a release:

```bash
helm uninstall my-nginx
```

---

# 🗂️ Helm Chart Structure

A typical Helm chart has the following structure:

```text
my-app/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl
└── charts/
```

### `Chart.yaml`

Contains chart metadata.

```text
Chart metadata
    │
    ├── Chart name
    ├── Version
    ├── Description
    └── Dependencies
```

### `values.yaml`

Contains default configuration values used by the templates.

For example:

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: latest

service:
  type: ClusterIP
  port: 80
```

### `templates/`

Contains Kubernetes resource templates.

Typical files include:

```text
templates/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
└── _helpers.tpl
```

### `charts/`

Contains packaged or downloaded chart dependencies.

---

# 🧩 Helm Template Model

Helm combines:

```text
values.yaml
      +
templates/
      +
Chart.yaml
      │
      ▼
 Helm Template Engine
      │
      ▼
Kubernetes YAML
      │
      ▼
Kubernetes API
```

This makes it possible to reuse the same application chart with different configuration values.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                                     |
| -------- | -------------------------------------------------------------------------------------------------------------- |
| **T1.1** | Add the Bitnami repository. Run `helm install my-nginx bitnami/nginx`. Check: `helm list && kubectl get pods`. |
| **T1.2** | Get release notes: `helm status my-nginx`. Follow instructions to get the service URL.                         |
| **T1.3** | Uninstall: `helm uninstall my-nginx`. Verify all Kubernetes resources are removed.                             |

---

## T1.1 — Install Nginx Using Helm

Add the Bitnami repository:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Update:

```bash
helm repo update
```

Install Nginx:

```bash
helm install my-nginx bitnami/nginx
```

Check Helm releases:

```bash
helm list
```

Check Kubernetes Pods:

```bash
kubectl get pods
```

Check all resources:

```bash
kubectl get all
```

You should see resources created by the Helm release.

---

## T1.2 — Check Release Status

Run:

```bash
helm status my-nginx
```

Helm will display information about the release, including:

* Release name
* Namespace
* Deployment status
* Revision
* Chart
* Application version
* Notes

Depending on the chart version, the output may include instructions for accessing the Nginx service.

You can also inspect the Service:

```bash
kubectl get svc
```

For detailed information:

```bash
kubectl describe svc my-nginx
```

> 💡 **Tip:** The exact Service name and access instructions can vary between chart versions. Always use `helm status` and `kubectl get svc` to determine the current deployment details.

---

## T1.3 — Uninstall the Release

Remove the release:

```bash
helm uninstall my-nginx
```

Verify:

```bash
helm list
```

Check Kubernetes resources:

```bash
kubectl get all
```

You can also check:

```bash
kubectl get pods
kubectl get svc
kubectl get deployments
```

> ⚠️ **Important:** Helm uninstall removes resources managed by the release, but some resources or externally provisioned infrastructure may require separate cleanup depending on the chart and configuration.

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                    |
| -------- | --------------------------------------------------------------------------------------------- |
| **T2.1** | Install with custom values: `helm install my-nginx bitnami/nginx --set service.type=NodePort` |
| **T2.2** | Upgrade a release: `helm upgrade my-nginx bitnami/nginx --set replicaCount=3`                 |
| **T2.3** | Roll back: `helm rollback my-nginx 1`. Verify the previous replica count is restored.         |

---

## T2.1 — Customize the Service Type

Install Nginx using a NodePort Service:

```bash
helm install my-nginx bitnami/nginx \
  --set service.type=NodePort
```

Verify the release:

```bash
helm list
```

Check the Service:

```bash
kubectl get svc
```

You should see:

```text
TYPE
NodePort
```

Get more information:

```bash
kubectl describe svc my-nginx
```

Check the assigned NodePort:

```bash
kubectl get svc my-nginx
```

---

## T2.2 — Upgrade the Release

Change the replica count to 3:

```bash
helm upgrade my-nginx bitnami/nginx \
  --set replicaCount=3
```

Check the release:

```bash
helm status my-nginx
```

Check the Pods:

```bash
kubectl get pods
```

You should observe three replicas if the chart's current values support `replicaCount` as expected.

Check the Helm revision:

```bash
helm history my-nginx
```

You should now see a new revision.

---

## T2.3 — Roll Back the Release

View the release history:

```bash
helm history my-nginx
```

Example:

```text
REVISION    STATUS
1           superseded
2           deployed
```

Roll back to revision 1:

```bash
helm rollback my-nginx 1
```

Check status:

```bash
helm status my-nginx
```

Check Pods:

```bash
kubectl get pods
```

Verify the configuration has returned to the earlier revision.

> 💡 **Key Concept:** Helm keeps release history, making it possible to return to a previous known-good configuration.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                          |
| -------- | --------------------------------------------------------------------------------------------------- |
| **T3.1** | Create your own Helm chart: `helm create myapp`. Modify `values.yaml` to deploy your Flask app.     |
| **T3.2** | Add chart dependencies in `Chart.yaml` (for example, Redis). Run `helm dependency update`. Install. |
| **T3.3** | Publish your chart to ChartMuseum. Install from the ChartMuseum repository in a second cluster.     |

---

# T3.1 — Create Your Own Helm Chart

Create a new chart:

```bash
helm create myapp
```

List the generated files:

```bash
ls -la myapp
```

Explore the chart:

```bash
cd myapp
find . -maxdepth 2 -type f
```

You should see a structure similar to:

```text
myapp/
├── Chart.yaml
├── values.yaml
├── charts/
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── ingress.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── tests/
    └── ...
```

---

## Modify `values.yaml`

Configure the chart for your Flask application.

Example:

```yaml
replicaCount: 2

image:
  repository: your-dockerhub-user/flask-app
  pullPolicy: IfNotPresent
  tag: "1.0"

service:
  type: ClusterIP
  port: 5000
```

> 🔧 Replace `your-dockerhub-user/flask-app` with the image you created in **Lab 05**.

Inspect the rendered Kubernetes manifests:

```bash
helm template myapp .
```

This is an important validation step because it lets you review the generated YAML before deploying it.

---

## Validate the Chart

Run:

```bash
helm lint .
```

Expected result:

```text
1 chart(s) linted, 0 chart(s) failed
```

Package the chart:

```bash
helm package .
```

This generates a package similar to:

```text
myapp-0.1.0.tgz
```

---

## Install Your Application

Install:

```bash
helm install myapp .
```

Check:

```bash
helm list
```

Check Kubernetes resources:

```bash
kubectl get all
```

Check Pods:

```bash
kubectl get pods
```

---

# T3.2 — Add Chart Dependencies

Helm charts can depend on other charts.

For example:

```text
myapp
 │
 ├── Flask application
 │
 └── Redis
```

Add a Redis dependency to `Chart.yaml`.

Example:

```yaml
dependencies:
  - name: redis
    version: "<CHART_VERSION>"
    repository: "https://charts.bitnami.com/bitnami"
```

> 💡 **Important:** Use a currently available Redis chart version rather than blindly copying an old version from training material.

Run:

```bash
helm dependency update
```

Check:

```bash
ls -la charts/
```

You should see the downloaded dependency.

You can also inspect:

```bash
helm dependency list
```

---

## Validate the Dependency

Run:

```bash
helm lint .
```

Render the complete chart:

```bash
helm template myapp .
```

Look through the rendered output to understand how your application and Redis resources will be deployed.

Install:

```bash
helm install myapp .
```

Verify:

```bash
helm list
```

Check:

```bash
kubectl get pods
```

---

# T3.3 — Publish to ChartMuseum

**ChartMuseum** is a Helm chart repository server.

The workflow is:

```text
Developer
    │
    ▼
Helm Chart
    │
    ▼
helm package
    │
    ▼
myapp-0.1.0.tgz
    │
    ▼
ChartMuseum
    │
    ▼
Helm Repository
    │
    ├──────────────► Cluster 1
    │
    └──────────────► Cluster 2
```

Package your chart:

```bash
helm package .
```

Example output:

```text
Successfully packaged chart and saved it to:
myapp-0.1.0.tgz
```

Add your ChartMuseum repository:

```bash
helm repo add devops-charts http://<CHARTMUSEUM-HOST>:8080
```

Update:

```bash
helm repo update
```

Search for your chart:

```bash
helm search repo myapp
```

Install from ChartMuseum:

```bash
helm install myapp devops-charts/myapp
```

Verify:

```bash
helm list
kubectl get pods
```

---

## 🌐 Install the Chart in a Second Cluster

Configure your `kubectl` context for the second cluster.

List available contexts:

```bash
kubectl config get-contexts
```

Switch context:

```bash
kubectl config use-context <SECOND-CLUSTER-CONTEXT>
```

Verify:

```bash
kubectl config current-context
```

Add the ChartMuseum repository:

```bash
helm repo add devops-charts http://<CHARTMUSEUM-HOST>:8080
```

Update repositories:

```bash
helm repo update
```

Install:

```bash
helm install myapp devops-charts/myapp
```

Verify:

```bash
helm list
kubectl get pods
kubectl get svc
```

> 💡 **Key Learning:** The same Helm chart can be reused across multiple Kubernetes clusters while changing environment-specific configuration through values.

---

# 🔍 Useful Helm Commands

## List Repositories

```bash
helm repo list
```

## Update Repositories

```bash
helm repo update
```

## Search Repository

```bash
helm search repo nginx
```

## Search Helm Hub

```bash
helm search hub wordpress
```

## Install Chart

```bash
helm install my-nginx bitnami/nginx
```

## List Releases

```bash
helm list -A
```

## Check Release Status

```bash
helm status my-nginx
```

## View Release History

```bash
helm history my-nginx
```

## Upgrade Release

```bash
helm upgrade my-nginx bitnami/nginx
```

## Roll Back

```bash
helm rollback my-nginx 1
```

## Uninstall Release

```bash
helm uninstall my-nginx
```

## Create Chart

```bash
helm create myapp
```

## Validate Chart

```bash
helm lint myapp
```

## Render Templates Locally

```bash
helm template myapp ./myapp
```

## Package Chart

```bash
helm package myapp
```

## Update Dependencies

```bash
helm dependency update myapp
```

## List Dependencies

```bash
helm dependency list myapp
```

---

# 🛠️ Troubleshooting

## Helm Command Not Found

Check:

```bash
helm version
```

If Helm is not installed, repeat the installation procedure and verify that the Helm binary is available in your `PATH`.

---

## Chart Not Found

Run:

```bash
helm repo list
```

Then:

```bash
helm repo update
```

Search:

```bash
helm search repo <chart-name>
```

---

## Release Already Exists

If you receive an error indicating that the release already exists:

```bash
helm list -A
```

Check the release.

You can either upgrade it:

```bash
helm upgrade my-nginx bitnami/nginx
```

or remove it:

```bash
helm uninstall my-nginx
```

---

## Helm Deployment Is Failing

Check:

```bash
helm status my-nginx
```

Then inspect Kubernetes resources:

```bash
kubectl get all
```

Check Pods:

```bash
kubectl get pods
```

If a Pod is failing:

```bash
kubectl describe pod <pod-name>
```

View logs:

```bash
kubectl logs <pod-name>
```

---

## Inspect Rendered YAML

Before installing or upgrading, use:

```bash
helm template myapp ./myapp
```

For debugging an installation:

```bash
helm install myapp ./myapp \
  --dry-run \
  --debug
```

This can reveal:

* Invalid YAML
* Incorrect values
* Template errors
* Missing configuration
* Incorrect resource definitions

---

# 🧹 Lab Cleanup

Uninstall the Nginx release:

```bash
helm uninstall my-nginx
```

Uninstall your application:

```bash
helm uninstall myapp
```

If you installed WordPress:

```bash
helm uninstall my-wp -n wordpress
```

Optionally remove the namespace:

```bash
kubectl delete namespace wordpress
```

Remove Helm repositories if they are no longer required:

```bash
helm repo remove stable
helm repo remove bitnami
helm repo remove prometheus-community
```

> ⚠️ **Note:** Remove only repositories and Kubernetes resources created for this lab. Do not delete shared resources used by other workloads.

---

# 🛡️ Helm Best Practices

* Use Helm 3 for modern Kubernetes deployments.
* Keep `values.yaml` clean and environment-oriented.
* Avoid hard-coding environment-specific configuration into templates.
* Use separate values files for different environments.
* Validate charts with `helm lint`.
* Render templates with `helm template` before deployment.
* Use `helm --dry-run --debug` when troubleshooting.
* Keep chart versions under source control.
* Pin dependency versions where appropriate.
* Review third-party charts before deploying them.
* Keep sensitive credentials out of `values.yaml`.
* Use Kubernetes Secrets or an external secret-management solution for sensitive values.
* Use release history to support controlled rollback.
* Use meaningful release names.
* Test Helm upgrades before applying them to production.
* Avoid using `latest` for production container images.
* Use versioned container image tags.
* Document required values and dependencies.
* Keep reusable charts modular and maintainable.

---

# 📋 Lab 12 Completion Checklist

* [ ] Install Helm v3.
* [ ] Verify Helm installation.
* [ ] Add Helm repositories.
* [ ] Update Helm repository indexes.
* [ ] Search for community charts.
* [ ] Install a Bitnami Nginx chart.
* [ ] Understand the concept of a Helm release.
* [ ] Check Helm release status.
* [ ] Access the deployed Service.
* [ ] Uninstall a Helm release.
* [ ] Install a chart using custom values.
* [ ] Change a Service to NodePort.
* [ ] Upgrade a Helm release.
* [ ] Change the replica count.
* [ ] View Helm release history.
* [ ] Roll back a Helm release.
* [ ] Create a custom Helm chart.
* [ ] Understand `Chart.yaml`.
* [ ] Understand `values.yaml`.
* [ ] Understand Helm templates.
* [ ] Run `helm lint`.
* [ ] Render templates with `helm template`.
* [ ] Package a Helm chart.
* [ ] Add chart dependencies.
* [ ] Run `helm dependency update`.
* [ ] Install a chart with dependencies.
* [ ] Understand ChartMuseum.
* [ ] Publish a chart to ChartMuseum.
* [ ] Add ChartMuseum as a Helm repository.
* [ ] Install a chart from ChartMuseum.
* [ ] Deploy the same chart to a second Kubernetes cluster.
* [ ] Apply Helm security and version-management best practices.

---

# 🧠 Lab 12 — Key Takeaways

| Component           | Purpose                                  |
| ------------------- | ---------------------------------------- |
| **Helm**            | Kubernetes package manager               |
| **Chart**           | Reusable Kubernetes application package  |
| **Chart.yaml**      | Chart metadata and dependencies          |
| **values.yaml**     | Default configuration values             |
| **templates/**      | Kubernetes resource templates            |
| **charts/**         | Chart dependencies                       |
| **Release**         | Installed instance of a Helm chart       |
| **Repository**      | Location where Helm charts are published |
| **`helm install`**  | Install a release                        |
| **`helm upgrade`**  | Update an existing release               |
| **`helm rollback`** | Return to an earlier release revision    |
| **ChartMuseum**     | Self-hosted Helm chart repository        |

### 🔄 The Helm Workflow

```text
                  Helm Repository
                        │
                        ▼
                  helm search repo
                        │
                        ▼
                   Helm Chart
                        │
                        ▼
                 helm install
                        │
                        ▼
                 Helm Release
                        │
                        ▼
              Kubernetes Resources
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
       helm upgrade          helm rollback
             │                     │
             └──────────┬──────────┘
                        ▼
                Managed Workload
```

> **🎓 Lab Outcome:** By completing Lab 12, you should be able to install and manage community Helm charts, customize deployments using values, perform upgrades and rollbacks, build your own Helm chart, manage dependencies, and understand how a private Helm repository such as ChartMuseum can support reusable deployments across multiple Kubernetes clusters.
