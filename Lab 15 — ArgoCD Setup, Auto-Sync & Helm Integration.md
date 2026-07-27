# Session 8 — GitOps with ArgoCD

## Lab 15 — ArgoCD Setup, Auto-Sync & Helm Integration

> **🎯 Objective:** Install ArgoCD, connect it to GitHub, configure GitOps-based deployments, enable automatic synchronization, and integrate Helm-based application deployments.

---

## 🧭 Overview

ArgoCD is a declarative, GitOps continuous delivery tool for Kubernetes.

In this lab, you will learn how to:

* Install ArgoCD in Kubernetes
* Access the ArgoCD web interface
* Retrieve and change the ArgoCD administrator password
* Connect ArgoCD to a GitHub repository
* Create an ArgoCD `Application`
* Deploy Kubernetes manifests from Git
* Enable automated synchronization
* Enable pruning of resources removed from Git
* Enable self-healing for manually changed resources
* Deploy Helm charts through ArgoCD
* Implement multi-environment GitOps
* Use ArgoCD Image Updater
* Implement the App of Apps pattern

---

# 🔄 GitOps Architecture

The core GitOps model used in this lab is:

```text
                  Developer
                      │
                      ▼
               GitHub Repository
                      │
                      │ Git changes
                      ▼
                    ArgoCD
                      │
             ┌────────┴────────┐
             │                 │
          Sync                  │
             │                 │
             ▼                 ▼
       Kubernetes API      Git State
             │
             ▼
       Kubernetes Cluster
             │
             ▼
        Application Pods
```

The Git repository becomes the **source of truth** for the desired Kubernetes configuration.

---

# 📁 Example Git Repository Structure

The lab assumes a GitHub repository named:

```text
devops-journey
```

A possible structure is:

```text
devops-journey/
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── namespace.yaml
│
├── helm/
│   └── myapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│
└── README.md
```

ArgoCD will monitor the repository and compare the desired state stored in Git with the actual state of the Kubernetes cluster.

---

# ⚙️ Install & Configure ArgoCD

## 1. Create the ArgoCD Namespace

Create a dedicated namespace:

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get namespace argocd
```

---

## 2. Install ArgoCD

Install ArgoCD using the official manifest:

```bash
kubectl apply -n argocd -f \
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify the resources:

```bash
kubectl get pods -n argocd
```

Wait until the ArgoCD components are running.

You can also monitor continuously:

```bash
kubectl get pods -n argocd -w
```

> 💡 **Note:** The first startup may take some time while Kubernetes downloads images and starts the ArgoCD components.

---

# 🔐 Retrieve the ArgoCD Admin Password

Retrieve the initial administrator password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d; echo
```

The username is:

```text
admin
```

> 🔐 **Security:** The initial administrator credential should be changed after the first login.

---

# 🌐 Access the ArgoCD Web Interface

Create a local port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Log in using:

```text
Username: admin
Password: <retrieved-password>
```

---

# 🖥️ ArgoCD CLI

Log in using the ArgoCD CLI:

```bash
argocd login localhost:8080 \
--username admin \
--insecure
```

List applications:

```bash
argocd app list
```

> 💡 **Note:** `--insecure` is useful for the local port-forward lab environment. Production environments should use properly configured TLS.

---

# 🔑 Change the ArgoCD Admin Password

After logging in, change the administrator password using:

```bash
argocd account update-password
```

Follow the prompts to enter:

1. Current password
2. New password
3. New password confirmation

Verify available accounts:

```bash
argocd account list
```

---

# 📄 ArgoCD Application Manifest

Create:

```text
argocd-application.yaml
```

Use the following configuration:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: myapp
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/YOUR_USER/devops-journey.git
    targetRevision: main
    path: k8s/

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Replace:

```text
YOUR_USER
```

with your actual GitHub username.

---

# 🧩 Understanding the ArgoCD Application

## API Version

```yaml
apiVersion: argoproj.io/v1alpha1
```

This identifies the ArgoCD `Application` custom resource.

---

## Application Name

```yaml
metadata:
  name: myapp
```

This creates an ArgoCD application named:

```text
myapp
```

---

## ArgoCD Namespace

```yaml
namespace: argocd
```

The ArgoCD `Application` resource itself resides in the `argocd` namespace.

---

## Project

```yaml
project: default
```

The application uses the default ArgoCD project.

---

# 📦 Git Source

```yaml
source:
  repoURL: https://github.com/YOUR_USER/devops-journey.git
  targetRevision: main
  path: k8s/
```

This tells ArgoCD:

```text
Repository:
https://github.com/YOUR_USER/devops-journey.git

Branch:
main

Kubernetes manifests:
k8s/
```

The desired state is therefore maintained in Git.

---

# 🎯 Kubernetes Destination

```yaml
destination:
  server: https://kubernetes.default.svc
  namespace: dev
```

This instructs ArgoCD to deploy the application into:

```text
Namespace: dev
```

on the Kubernetes cluster where ArgoCD is running.

---

# 🔄 Automated Synchronization

Enable automatic synchronization:

```yaml
syncPolicy:
  automated:
```

This allows ArgoCD to automatically synchronize the Kubernetes cluster with Git.

---

## 🧹 Automatic Pruning

```yaml
prune: true
```

If a Kubernetes resource is removed from Git, ArgoCD can remove the corresponding resource from the cluster.

Example:

```text
Git
 │
 │ Deployment exists
 ▼
Kubernetes
 │
 ▼
Deployment exists
```

If the Deployment is removed from Git:

```text
Git
 │
 │ Deployment removed
 ▼
ArgoCD
 │
 ▼
Kubernetes Deployment removed
```

> ⚠️ **Warning:** Automatic pruning can delete resources. Use it carefully, particularly in production environments.

---

# 🩹 Self-Healing

```yaml
selfHeal: true
```

Self-healing allows ArgoCD to correct manual changes made directly to Kubernetes.

Example:

```text
Git:
replicas: 3

        ↓

Kubernetes:
replicas: 5

        ↓

ArgoCD detects drift

        ↓

ArgoCD restores:

replicas: 3
```

This reinforces Git as the desired-state source of truth.

---

# 🏗️ Create Namespace Automatically

```yaml
syncOptions:
  - CreateNamespace=true
```

ArgoCD will create the destination namespace if it does not already exist.

---

# 🚀 Apply the ArgoCD Application

Apply the application manifest:

```bash
kubectl apply -f argocd-application.yaml
```

Verify:

```bash
kubectl get applications -n argocd
```

Or:

```bash
argocd app list
```

Inspect the application:

```bash
argocd app get myapp
```

---

# 🔄 Manual Synchronization

Synchronize from the CLI:

```bash
argocd app sync myapp
```

Check application status:

```bash
argocd app get myapp
```

Check Kubernetes resources:

```bash
kubectl get all -n dev
```

You can also perform the synchronization through the ArgoCD web interface.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                   |
| -------- | ---------------------------------------------------------------------------- |
| **T1.1** | Install ArgoCD. Access UI. Change admin password. Explore the web interface. |
| **T1.2** | Add GitHub repo to ArgoCD: Settings → Repositories → Connect Repo (HTTPS).   |
| **T1.3** | Apply `argocd-application.yaml`. Sync manually from UI. Watch pods deploy.   |

---

## T1.1 — Install and Explore ArgoCD

Install:

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f \
https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check:

```bash
kubectl get pods -n argocd
```

Retrieve the password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d; echo
```

Start port forwarding:

```bash
kubectl port-forward svc/argocd-server \
-n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Log in as:

```text
admin
```

Change the password:

```bash
argocd account update-password
```

Explore the ArgoCD interface, including:

* Applications
* Projects
* Settings
* Repositories
* Clusters
* Application health
* Application sync status
* Application history
* Resource trees

---

## T1.2 — Connect GitHub Repository

From the ArgoCD UI, navigate to:

```text
Settings
   ↓
Repositories
   ↓
Connect Repo
```

Select:

```text
HTTPS
```

Provide your repository information.

Example:

```text
Repository URL:
https://github.com/YOUR_USER/devops-journey.git
```

For a public repository, authentication may not be required.

For a private repository, configure the appropriate GitHub authentication credentials.

Verify the repository connection from the ArgoCD UI.

---

## T1.3 — Deploy the Application

Apply:

```bash
kubectl apply -f argocd-application.yaml
```

Check:

```bash
argocd app list
```

Open the ArgoCD UI and locate:

```text
myapp
```

Perform a manual synchronization.

Monitor:

```bash
kubectl get pods -n dev -w
```

Verify the application:

```bash
kubectl get all -n dev
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                        |
| -------- | --------------------------------------------------------------------------------- |
| **T2.1** | Enable auto-sync. Change replica count in Git. Push. Watch ArgoCD auto-deploy.    |
| **T2.2** | Manually change a deployment via `kubectl`. Watch ArgoCD self-heal and revert it. |
| **T2.3** | Set up ArgoCD to deploy a Helm chart from GitHub (Helm source type).              |

---

# T2.1 — Enable Auto-Sync

The Application configuration should contain:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Apply the updated configuration:

```bash
kubectl apply -f argocd-application.yaml
```

Check:

```bash
argocd app get myapp
```

Now modify the Deployment stored in Git.

For example:

```yaml
spec:
  replicas: 5
```

Commit:

```bash
git add .
git commit -m "feat: scale application to five replicas"
```

Push:

```bash
git push origin main
```

ArgoCD detects the Git change and synchronizes the Kubernetes environment.

Monitor:

```bash
kubectl get pods -n dev -w
```

Verify:

```bash
kubectl get deployment -n dev
```

---

# T2.2 — Test ArgoCD Self-Healing

First verify the desired state in Git.

For example:

```yaml
replicas: 3
```

Then manually modify Kubernetes:

```bash
kubectl scale deployment myapp \
--replicas=5 \
-n dev
```

Check:

```bash
kubectl get deployment myapp -n dev
```

ArgoCD should detect that the live cluster state differs from the Git state.

Conceptually:

```text
Git Desired State
replicas: 3
       │
       ▼
   ArgoCD
       │
       │ Detect drift
       ▼
Kubernetes Live State
replicas: 5
       │
       ▼
   Self-Heal
       │
       ▼
replicas: 3
```

Verify:

```bash
kubectl get deployment myapp -n dev
```

> 🛡️ **GitOps Principle:** Manual changes to managed resources should generally be made through Git rather than directly with `kubectl`.

---

# T2.3 — Deploy a Helm Chart from GitHub

A Git repository can contain a Helm chart:

```text
devops-journey/
└── helm/
    └── myapp/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            └── service.yaml
```

Configure the ArgoCD source:

```yaml
source:
  repoURL: https://github.com/YOUR_USER/devops-journey.git
  targetRevision: main
  path: helm/myapp
  helm:
    valueFiles:
      - values.yaml
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: myapp-helm
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/YOUR_USER/devops-journey.git
    targetRevision: main
    path: helm/myapp

    helm:
      valueFiles:
        - values.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Apply:

```bash
kubectl apply -f argocd-application.yaml
```

Check:

```bash
argocd app list
```

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                             |
| -------- | ------------------------------------------------------------------------------------------------------ |
| **T3.1** | Multi-env: create ArgoCD apps for dev and prod namespaces, each deploying from different Git branches. |
| **T3.2** | ArgoCD Image Updater: auto-update image tag in Git when new ECR image is pushed.                       |
| **T3.3** | App of Apps pattern: create a parent ArgoCD app that manages multiple child apps from one Git repo.    |

---

# T3.1 — Multi-Environment GitOps

Create separate branches:

```text
main
dev
```

Or use an environment-specific structure such as:

```text
environments/
├── dev/
└── prod/
```

A branch-based model for this exercise can be:

```text
GitHub
│
├── dev branch
│     │
│     ▼
│   ArgoCD
│     │
│     ▼
│   Kubernetes
│     │
│     └── dev namespace
│
└── main branch
      │
      ▼
    ArgoCD
      │
      ▼
   Kubernetes
      │
      └── prod namespace
```

---

## Development Application

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: myapp-dev
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/YOUR_USER/devops-journey.git
    targetRevision: dev
    path: k8s/

  destination:
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

---

## Production Application

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: myapp-prod
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/YOUR_USER/devops-journey.git
    targetRevision: main
    path: k8s/

  destination:
    server: https://kubernetes.default.svc
    namespace: prod

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Verify both applications:

```bash
argocd app list
```

Expected concept:

```text
myapp-dev
    │
    └── Git: dev branch
          │
          ▼
       dev namespace


myapp-prod
    │
    └── Git: main branch
          │
          ▼
       prod namespace
```

> ⚠️ **Production Recommendation:** In real environments, production deployments should generally have additional approval and change-control safeguards rather than unrestricted automatic synchronization.

---

# T3.2 — ArgoCD Image Updater

ArgoCD Image Updater can monitor container image repositories and update image versions used by ArgoCD-managed applications.

Example flow:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
CI/CD
    │
    ▼
Docker Build
    │
    ▼
Amazon ECR
    │
    │ New image
    ▼
ArgoCD Image Updater
    │
    ▼
Update image reference
    │
    ▼
ArgoCD
    │
    ▼
Kubernetes
```

For this challenge:

1. Deploy ArgoCD Image Updater.
2. Configure access to Amazon ECR.
3. Configure your ArgoCD application for image updates.
4. Push a new image to ECR.
5. Verify that Image Updater detects the new image.
6. Verify the desired image version is updated according to your configured strategy.
7. Verify ArgoCD synchronizes the updated workload.

> 🔐 **Security:** Use appropriate AWS IAM permissions and avoid embedding long-lived AWS credentials in Kubernetes manifests or Git repositories.

---

# T3.3 — App of Apps Pattern

The **App of Apps** pattern uses a parent ArgoCD Application to manage multiple child Applications.

Architecture:

```text
                    ArgoCD
                       │
                       ▼
                 Parent Application
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Child App    Child App    Child App
        frontend      backend       redis
          │            │            │
          ▼            ▼            ▼
       Deployment   Deployment   Deployment
```

A Git repository might contain:

```text
argocd/
├── root-app.yaml
└── applications/
    ├── frontend.yaml
    ├── backend.yaml
    └── redis.yaml
```

The parent application points to:

```text
argocd/applications/
```

ArgoCD then discovers and manages the child Application resources.

Conceptually:

```text
GitHub
│
└── argocd/
    │
    ├── root-app.yaml
    │
    └── applications/
         ├── frontend.yaml
         ├── backend.yaml
         └── redis.yaml
```

This makes it possible to manage many applications from a single GitOps entry point.

---

# 🔍 ArgoCD CLI Reference

List applications:

```bash
argocd app list
```

Get application details:

```bash
argocd app get myapp
```

Synchronize an application:

```bash
argocd app sync myapp
```

View application history:

```bash
argocd app history myapp
```

Refresh application state:

```bash
argocd app get myapp --refresh
```

Delete an application:

```bash
argocd app delete myapp
```

> ⚠️ **Warning:** Deleting an ArgoCD Application can also remove managed Kubernetes resources depending on the application's configuration and deletion behavior. Review the resources before deleting production applications.

---

# 🔎 Kubernetes Troubleshooting

Check ArgoCD pods:

```bash
kubectl get pods -n argocd
```

Check ArgoCD services:

```bash
kubectl get svc -n argocd
```

Check ArgoCD Application resources:

```bash
kubectl get applications -n argocd
```

Describe an Application:

```bash
kubectl describe application myapp -n argocd
```

Check application namespace:

```bash
kubectl get all -n dev
```

Check events:

```bash
kubectl get events -n dev --sort-by=.lastTimestamp
```

---

# 🧰 Common ArgoCD Troubleshooting

## Application Shows `OutOfSync`

Possible causes:

* Git repository changed.
* Kubernetes resources were manually modified.
* ArgoCD has not refreshed yet.
* Manifest rendering has changed.
* Helm values differ.
* The application is configured for manual synchronization.

Check:

```bash
argocd app get myapp
```

Refresh:

```bash
argocd app get myapp --refresh
```

Synchronize:

```bash
argocd app sync myapp
```

---

## Application Shows `Degraded`

Check:

```bash
argocd app get myapp
```

Then inspect Kubernetes resources:

```bash
kubectl get all -n dev
```

Inspect the affected pod:

```bash
kubectl describe pod <pod-name> -n dev
```

View logs:

```bash
kubectl logs <pod-name> -n dev
```

---

## Repository Connection Failure

Check:

* Repository URL
* Branch name
* Repository path
* GitHub credentials
* Repository visibility
* Network connectivity
* GitHub authentication configuration

---

## Pods Are Not Starting

Check:

```bash
kubectl get pods -n dev
```

Then:

```bash
kubectl describe pod <pod-name> -n dev
```

And:

```bash
kubectl logs <pod-name> -n dev
```

---

# 🛡️ GitOps Best Practices

* Treat Git as the source of truth.
* Avoid manual production changes with `kubectl`.
* Use Pull Requests for configuration changes.
* Review Kubernetes manifests before merging.
* Use protected branches for production.
* Use separate environments for development and production.
* Use ArgoCD Projects to restrict application permissions.
* Use least-privilege repository and cluster access.
* Protect GitHub credentials and Kubernetes secrets.
* Use automated synchronization carefully.
* Understand the impact of `prune: true`.
* Use `selfHeal: true` where appropriate.
* Use approval controls for sensitive production deployments.
* Use Helm values carefully and keep environment-specific configuration clear.
* Monitor ArgoCD application health.
* Keep application and infrastructure changes auditable through Git history.
* Avoid storing plaintext credentials in Git.
* Use AWS IAM and appropriate identity mechanisms for ECR access.
* Regularly review ArgoCD and Kubernetes permissions.

---

# 📋 Lab 15 Completion Checklist

* [ ] Create the `argocd` namespace.
* [ ] Install ArgoCD.
* [ ] Verify ArgoCD pods are running.
* [ ] Retrieve the initial admin password.
* [ ] Access the ArgoCD web interface.
* [ ] Change the admin password.
* [ ] Install and configure the ArgoCD CLI.
* [ ] Connect a GitHub repository.
* [ ] Create an ArgoCD `Application`.
* [ ] Configure Git as the application source.
* [ ] Configure Kubernetes as the destination.
* [ ] Perform a manual synchronization.
* [ ] Enable automated synchronization.
* [ ] Test automatic deployment after a Git change.
* [ ] Enable automatic pruning.
* [ ] Test ArgoCD self-healing.
* [ ] Manually modify a Kubernetes Deployment.
* [ ] Verify ArgoCD restores the Git-defined state.
* [ ] Deploy a Helm chart through ArgoCD.
* [ ] Create separate dev and production applications.
* [ ] Use different Git branches for environments.
* [ ] Explore ArgoCD Image Updater.
* [ ] Configure ECR image update automation.
* [ ] Understand the App of Apps pattern.
* [ ] Create a parent ArgoCD application.
* [ ] Manage multiple child applications.
* [ ] Practice ArgoCD troubleshooting.
* [ ] Review GitOps security and governance practices.

---

# 🧠 Lab 15 — Key Takeaways

| Concept           | Purpose                                                           |
| ----------------- | ----------------------------------------------------------------- |
| **ArgoCD**        | GitOps continuous delivery tool for Kubernetes                    |
| **GitOps**        | Uses Git as the declarative source of truth                       |
| **Application**   | Defines the relationship between Git and Kubernetes               |
| **Sync**          | Applies desired Git state to Kubernetes                           |
| **Auto-Sync**     | Automatically synchronizes detected changes                       |
| **Prune**         | Removes resources no longer defined in Git                        |
| **Self-Heal**     | Corrects manual drift from desired state                          |
| **Helm**          | Kubernetes package and templating system                          |
| **Image Updater** | Automates container image update workflows                        |
| **App of Apps**   | Manages multiple ArgoCD applications through a parent application |
| **OutOfSync**     | Live Kubernetes state differs from Git                            |
| **Healthy**       | Application resources are operating as expected                   |
| **Degraded**      | One or more application resources have health problems            |
| **Git Source**    | Repository containing the desired configuration                   |
| **Destination**   | Kubernetes cluster and namespace receiving the deployment         |

---

# 🚀 End-to-End GitOps Workflow

```text
                         Developer
                             │
                             ▼
                       GitHub Repository
                             │
                             │ Pull Request
                             ▼
                     Code Review + CI
                             │
                             ▼
                       Merge to Git
                             │
                             ▼
                    Git Desired State
                             │
                             ▼
                          ArgoCD
                             │
                  ┌──────────┴──────────┐
                  │                     │
             Compare State          Detect Drift
                  │                     │
                  ▼                     ▼
              Synchronize           Self-Heal
                  │
                  ▼
              Kubernetes
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Frontend   Backend     Database
       │          │          │
       └──────────┼──────────┘
                  ▼
             Running App 🚀
```

> **🎓 Lab Outcome:** By completing Lab 15, you should be able to install and operate ArgoCD, connect Kubernetes workloads to Git repositories, perform manual and automated synchronization, detect and correct configuration drift, deploy Helm applications, manage multiple environments, understand image-update automation, and implement the App of Apps GitOps pattern.
