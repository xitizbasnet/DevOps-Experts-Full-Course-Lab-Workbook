# Final Project — Building a Full-Scale DevOps Pipeline

> [!NOTE]
> **Project Type:** Multi-phase, end-to-end DevOps implementation
> **Scope:** This rolling project connects all **15 sessions** into one progressively built DevOps platform.
>
> **Approach:** Build the project progressively, validating each phase before moving to the next.

---

# Project Overview

The **Final Project — Building a Full-Scale DevOps Pipeline** connects all 15 sessions into a single production-like DevOps implementation.

The project progresses through four major phases:

```text
Phase 1
Containerize Python Flask App
        ↓
Phase 2
Deploy to Kubernetes
        ↓
Phase 3
CI/CD with GitHub Actions + Helm
        ↓
Phase 4
GitOps + Monitoring
```

The final architecture will provide:

* 🐍 Python Flask application
* 🐳 Docker containerization
* 🔴 Redis integration
* 🌐 Nginx reverse proxy
* ☸️ Kubernetes / Amazon EKS
* 🔐 ConfigMaps and Secrets
* 📈 Horizontal Pod Autoscaling (HPA)
* 🔒 HTTPS with cert-manager
* 🔄 GitHub Actions CI/CD
* 📦 Helm packaging
* 🚀 ArgoCD GitOps
* 📊 Prometheus and Grafana monitoring
* 🚨 Alertmanager and Slack alerting
* 🔍 Trivy container security scanning

---

# Phase 1 — Containerize Python Flask App

## Objective

Create and containerize a Python Flask application, test the application locally, publish the container image, and scan it for security vulnerabilities.

---

## 1.1 Create Flask Application

Create a Python Flask application with a `/health` endpoint.

The application should provide a health-check endpoint that can be used by Docker, Kubernetes, load balancers, and monitoring systems.

### Required Endpoint

```text
GET /health
```

### Expected Flow

```text
Client
  ↓
Flask Application
  ↓
/health
  ↓
Health Response
```

---

## 1.2 Create a Multi-Stage Dockerfile

Create a **multi-stage Dockerfile** for the Flask application.

The Dockerfile should be used to:

* Build the application image.
* Separate build dependencies from runtime dependencies.
* Produce a clean runtime image.
* Minimize the final image size.

```text
Application Source
        ↓
Multi-Stage Docker Build
        ↓
Final Runtime Image
```

---

## 1.3 Build and Push to Docker Hub

Build the Docker image and push it to **Docker Hub**.

```text
Flask Application
       ↓
Docker Build
       ↓
Docker Image
       ↓
Docker Hub
```

> [!IMPORTANT]
> Verify that the image can be successfully pulled from Docker Hub before continuing to the next step.

---

## 1.4 Create docker-compose.yml

Create a `docker-compose.yml` file containing:

* Flask
* Redis
* Nginx

### Local Architecture

```text
                 ┌─────────────┐
                 │    Nginx    │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │    Flask    │
                 │ Application │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │    Redis    │
                 └─────────────┘
```

### Test Locally

Use Docker Compose to start the complete application stack.

Verify:

* Flask starts successfully.
* Redis is reachable.
* Nginx is reachable.
* Flask `/health` endpoint responds successfully.

---

## 1.5 Push Image to Amazon ECR

After validating the container locally, push the image to **Amazon Elastic Container Registry (ECR)**.

```text
Local Docker Image
       ↓
Amazon ECR
```

Verify that the image appears in the appropriate ECR repository.

---

## 1.6 Scan Image with Trivy

Scan the ECR/container image using **Trivy**.

```text
Docker Image
     ↓
   Trivy
     ↓
Vulnerability Scan
```

### Security Requirement

Fix any:

* **HIGH** vulnerabilities
* **CRITICAL** vulnerabilities

> [!WARNING]
> Do not proceed with known HIGH/CRITICAL vulnerabilities without addressing them according to the project's security requirements.

### Phase 1 Completion Criteria

* [ ] Flask application created.
* [ ] `/health` endpoint implemented.
* [ ] Multi-stage Dockerfile created.
* [ ] Image successfully built.
* [ ] Image pushed to Docker Hub.
* [ ] `docker-compose.yml` created.
* [ ] Flask + Redis + Nginx tested locally.
* [ ] Image pushed to Amazon ECR.
* [ ] Trivy scan completed.
* [ ] HIGH/CRITICAL CVEs fixed.

---

# Phase 2 — Deploy to Kubernetes with ConfigMaps, Secrets & Autoscaling

## Objective

Deploy the Flask application to **Amazon EKS** using Kubernetes resources, configuration management, secrets, autoscaling, ingress, and HTTPS.

---

## 2.1 Create Kubernetes Manifests

Create Kubernetes YAML manifests for:

* Deployment
* Service
* HPA

### Kubernetes Architecture

```text
                    ┌──────────────┐
                    │ Kubernetes   │
                    │   Service    │
                    └──────┬───────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │    Deployment   │
                  │                 │
                  │  Flask Pods     │
                  └────────┬────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
                ConfigMap       Secret
```

---

## 2.2 Configure ConfigMap

Use a Kubernetes **ConfigMap** for application configuration.

```text
ConfigMap
    ↓
Application Pod
    ↓
Application Configuration
```

> [!NOTE]
> ConfigMaps should contain application configuration that does not require secret protection.

---

## 2.3 Configure Kubernetes Secret

Use a Kubernetes **Secret** for database credentials.

```text
Kubernetes Secret
       ↓
Database Credentials
       ↓
Application Pod
```

> [!WARNING]
> Do not commit plaintext credentials to Git repositories.

---

## 2.4 Deploy to Amazon EKS

Deploy the Kubernetes resources to **Amazon EKS**.

Verify:

* Deployment is available.
* Pods are running.
* Service is available.
* ConfigMap is applied.
* Secret is available.
* Application is responding correctly.

```text
kubectl
   ↓
Amazon EKS
   ↓
Kubernetes Resources
   ↓
Flask Application
```

---

## 2.5 Configure Horizontal Pod Autoscaler

Configure an **HPA** for the Flask application.

### Scaling Requirement

The HPA should scale pods when CPU utilization exceeds:

> **50%**

```text
CPU Utilization
       ↓
    > 50%
       ↓
HPA Trigger
       ↓
Increase Pod Count
```

### Verify HPA

Generate sufficient workload and verify that Kubernetes increases the number of application pods.

```text
Low CPU
  ↓
Normal Replicas

High CPU > 50%
  ↓
HPA
  ↓
More Replicas
```

---

## 2.6 Add Ingress

Configure Kubernetes **Ingress** to expose the Flask application.

```text
Internet
    ↓
Ingress
    ↓
Kubernetes Service
    ↓
Flask Pods
```

---

## 2.7 Configure cert-manager for HTTPS

Add **cert-manager** to provide HTTPS certificates.

```text
Client
  ↓
HTTPS
  ↓
Ingress
  ↓
cert-manager Certificate
  ↓
Flask Application
```

### Testing

Test the HTTPS endpoint using:

* A real domain, or
* `nip.io`

> [!IMPORTANT]
> Verify that HTTPS is functioning correctly before considering Phase 2 complete.

### Phase 2 Completion Criteria

* [ ] Deployment YAML created.
* [ ] Service YAML created.
* [ ] HPA YAML created.
* [ ] ConfigMap implemented.
* [ ] Secret implemented for DB credentials.
* [ ] Application deployed to EKS.
* [ ] HPA verified.
* [ ] HPA scales pods when CPU > 50%.
* [ ] Ingress configured.
* [ ] cert-manager configured.
* [ ] HTTPS tested.
* [ ] Domain or `nip.io` tested successfully.

---

# Phase 3 — CI/CD with GitHub Actions + Helm

## Objective

Implement a complete CI/CD workflow using **GitHub Actions and Helm**.

The pipeline should automatically test, build, scan, publish, update infrastructure configuration, and deploy applications.

---

# 3.1 GitHub Actions Pipeline

Create a GitHub Actions workflow with the following pipeline:

```text
Code
 ↓
Test
 ↓
Build
 ↓
Push to ECR
 ↓
Update Helm values.yaml
 ↓
Push to Infra Repository
```

### Required Pipeline Steps

1. **Test**
2. **Build**
3. **Push to ECR**
4. **Update `values.yaml`**
5. **Push to infra repository**

---

## CI/CD Flow

```text
Developer
    ↓
GitHub
    ↓
GitHub Actions
    ↓
Test
    ↓
Build Docker Image
    ↓
Push Image to ECR
    ↓
Update Helm values.yaml
    ↓
Push to Infra Repository
```

> [!TIP]
> Use immutable image identifiers such as Git commit SHA values for traceability between source code and deployed container images.

---

# 3.2 Create Helm Chart

Create a Helm chart for the Flask application.

Parameterize the following:

* Image
* Replicas
* Resources

### Helm Architecture

```text
Helm Chart
├── Image
├── Replicas
└── Resources
```

The Helm chart should allow deployment configuration to be changed without modifying the core Kubernetes manifests.

---

## Helm Deployment

Deploy the application using:

```text
helm upgrade
```

### Deployment Flow

```text
Helm Chart
    ↓
helm upgrade
    ↓
Amazon EKS
    ↓
Kubernetes
    ↓
Flask Application
```

---

# 3.3 Branch Strategy

Implement the following branch strategy:

```text
Feature Branch
      ↓
Pull Request
      ↓
develop
      ↓
Testing / Validation
      ↓
main
      ↓
Production Deployment
```

### Required Behavior

* PRs target `develop`.
* Merging to `main` triggers the production deployment.
* Production deployment requires **manual approval**.

### Production Flow

```text
Pull Request
     ↓
develop
     ↓
Validation
     ↓
Merge to main
     ↓
Production Deployment
     ↓
Manual Approval
     ↓
Deploy
```

> [!IMPORTANT]
> The production deployment must include a **manual approval gate**.

### Phase 3 Completion Criteria

* [ ] GitHub Actions workflow created.
* [ ] Automated tests configured.
* [ ] Docker build configured.
* [ ] ECR push configured.
* [ ] Helm `values.yaml` update automated.
* [ ] Infra repository update automated.
* [ ] Helm chart created.
* [ ] Image parameterized.
* [ ] Replica count parameterized.
* [ ] Resources parameterized.
* [ ] `helm upgrade` deployment tested.
* [ ] PRs target `develop`.
* [ ] Merge to `main` triggers production deployment.
* [ ] Production deployment requires manual approval.

---

# Phase 4 — GitOps + Monitoring

## Objective

Implement GitOps deployment using **ArgoCD** and add complete application monitoring using **Prometheus, Grafana, and Alertmanager**.

---

# 4.1 Configure ArgoCD

Configure ArgoCD to watch the **infra repository**.

```text
Infra Repository
       ↓
     ArgoCD
       ↓
     Amazon EKS
```

ArgoCD should automatically deploy changes when the Helm `values.yaml` changes.

### GitOps Flow

```text
GitHub
  ↓
Infra Repository
  ↓
values.yaml Changes
  ↓
ArgoCD Detects Change
  ↓
Auto-Deploy
  ↓
Amazon EKS
```

> [!IMPORTANT]
> ArgoCD should automatically deploy changes when `Helm values.yaml` changes.

---

# 4.2 Configure Prometheus

Configure Prometheus to scrape the Flask application's `/metrics` endpoint.

```text
Flask Application
       ↓
   /metrics
       ↓
  Prometheus
```

### Monitoring Flow

```text
Flask Pods
    ↓
/metrics endpoint
    ↓
Prometheus
    ↓
Metrics Database
```

---

# 4.3 Build Grafana Dashboard

Create a Grafana dashboard containing the following application metrics:

* **RPS**
* **p95 latency**
* **Error rate**

### Dashboard Architecture

```text
              Flask Application
                     │
                     ▼
                  /metrics
                     │
                     ▼
                Prometheus
                     │
                     ▼
                  Grafana
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
       RPS       p95 Latency   Error Rate
```

### Dashboard Requirements

The Grafana dashboard must display:

#### RPS

**Requests Per Second**

Used to understand application traffic volume.

#### p95 Latency

The **95th percentile latency** of application requests.

Used to identify the response time experienced by the majority of users while highlighting slower requests.

#### Error Rate

The percentage of requests resulting in errors.

---

# 4.4 Configure Alertmanager

Configure **Alertmanager** to monitor the application error rate.

### Alert Condition

Trigger an alert when:

```text
Error Rate > 1%
for 5 minutes
```

### Alert Flow

```text
Flask Application
       ↓
   /metrics
       ↓
   Prometheus
       ↓
Error Rate > 1%
for 5 minutes
       ↓
 Alertmanager
       ↓
      Slack
```

> [!WARNING]
> The alert must remain above the **1% error-rate threshold for 5 minutes** before triggering.

---

# 4.5 Test Alert End-to-End

Test the complete monitoring and alerting pipeline.

### Test Flow

```text
Generate Application Errors
          ↓
      Flask /metrics
          ↓
       Prometheus
          ↓
 Error Rate > 1%
   for 5 minutes
          ↓
     Alertmanager
          ↓
         Slack
          ↓
   Alert Received
```

### Verification

Verify that:

1. Flask exposes `/metrics`.
2. Prometheus successfully scrapes the application.
3. Grafana displays application metrics.
4. Error rate exceeds 1%.
5. The condition remains for 5 minutes.
6. Prometheus triggers the alert.
7. Alertmanager processes the alert.
8. Slack receives the alert.

---

# Phase 4 Completion Criteria

* [ ] ArgoCD configured.
* [ ] ArgoCD watches the infra repository.
* [ ] Helm `values.yaml` changes trigger deployment.
* [ ] Prometheus configured.
* [ ] Flask `/metrics` endpoint configured.
* [ ] Prometheus successfully scrapes Flask metrics.
* [ ] Grafana configured.
* [ ] RPS dashboard created.
* [ ] p95 latency dashboard created.
* [ ] Error-rate dashboard created.
* [ ] Alertmanager configured.
* [ ] Error-rate alert configured for > 1% for 5 minutes.
* [ ] Slack integration configured.
* [ ] End-to-end alert tested successfully.

---

# Final Project Architecture

The completed project should provide the following end-to-end architecture:

```text
                              ┌─────────────────┐
                              │    Developer    │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │     GitHub      │
                              │  Git / PR / SCM │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ GitHub Actions  │
                              │                 │
                              │ Test            │
                              │ Build           │
                              │ Push to ECR     │
                              │ Update Helm     │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │   Amazon ECR    │
                              │ Container Image  │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ Infra Repository│
                              │  Helm values    │
                              └────────┬────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │     ArgoCD      │
                              │     GitOps      │
                              └────────┬────────┘
                                       │
                                       ▼
                         ┌───────────────────────────┐
                         │        Amazon EKS         │
                         │                           │
                         │  ┌─────────────────────┐  │
                         │  │      Ingress        │  │
                         │  │   HTTPS / TLS       │  │
                         │  └──────────┬──────────┘  │
                         │             │             │
                         │             ▼             │
                         │  ┌─────────────────────┐  │
                         │  │      Service        │  │
                         │  └──────────┬──────────┘  │
                         │             │             │
                         │             ▼             │
                         │  ┌─────────────────────┐  │
                         │  │   Flask Deployment  │  │
                         │  │                     │  │
                         │  │  Pod ←→ Pod ←→ Pod │  │
                         │  └──────────┬──────────┘  │
                         │             │             │
                         │       ┌─────┴─────┐       │
                         │       ▼           ▼       │
                         │  ConfigMap      Secret   │
                         │                           │
                         │          HPA              │
                         │       CPU > 50%           │
                         └─────────────┬─────────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │    Prometheus   │
                              │     /metrics    │
                              └────────┬────────┘
                                       │
                         ┌─────────────┴─────────────┐
                         ▼                           ▼
                ┌─────────────────┐         ┌─────────────────┐
                │     Grafana     │         │  Alertmanager   │
                │                 │         │                 │
                │ RPS             │         │ Error > 1%      │
                │ p95 Latency     │         │ for 5 minutes   │
                │ Error Rate      │         └────────┬────────┘
                └─────────────────┘                  │
                                                     ▼
                                            ┌─────────────────┐
                                            │      Slack      │
                                            │     Alert       │
                                            └─────────────────┘
```

---

# Final Project Workflow

The complete project connects all four phases into one DevOps lifecycle:

```text
┌───────────────────────────────────────────────────────────────┐
│                         SOURCE                                │
│                         GitHub                                │
└───────────────────────────────┬───────────────────────────────┘
                                ↓
┌───────────────────────────────────────────────────────────────┐
│                         CI/CD                                 │
│                      GitHub Actions                            │
│                                                               │
│ Test → Build → Push ECR → Update Helm values.yaml             │
└───────────────────────────────┬───────────────────────────────┘
                                ↓
┌───────────────────────────────────────────────────────────────┐
│                         GITOPS                                │
│                         ArgoCD                                │
│                                                               │
│             Infra Repository → Amazon EKS                     │
└───────────────────────────────┬───────────────────────────────┘
                                ↓
┌───────────────────────────────────────────────────────────────┐
│                      KUBERNETES                               │
│                         EKS                                   │
│                                                               │
│ Ingress → Service → Flask Pods → ConfigMap / Secret           │
│                          ↑                                    │
│                          │                                    │
│                         HPA                                   │
└───────────────────────────────┬───────────────────────────────┘
                                ↓
┌───────────────────────────────────────────────────────────────┐
│                       MONITORING                              │
│                                                               │
│ Flask /metrics → Prometheus → Grafana                         │
│                         │                                     │
│                         ↓                                     │
│                    Alertmanager                               │
│                         ↓                                     │
│                       Slack                                   │
└───────────────────────────────────────────────────────────────┘
```

---

# Final Project Validation Checklist

## Phase 1 — Containerization

* [ ] Flask application created.
* [ ] `/health` endpoint working.
* [ ] Multi-stage Dockerfile created.
* [ ] Docker image built.
* [ ] Image pushed to Docker Hub.
* [ ] `docker-compose.yml` created.
* [ ] Flask + Redis + Nginx tested locally.
* [ ] Image pushed to ECR.
* [ ] Trivy scan completed.
* [ ] HIGH/CRITICAL CVEs fixed.

## Phase 2 — Kubernetes

* [ ] Deployment YAML created.
* [ ] Service YAML created.
* [ ] HPA YAML created.
* [ ] ConfigMap configured.
* [ ] Secret configured.
* [ ] Application deployed to EKS.
* [ ] HPA verified.
* [ ] CPU > 50% scaling tested.
* [ ] Ingress configured.
* [ ] cert-manager configured.
* [ ] HTTPS tested.
* [ ] Domain or `nip.io` tested.

## Phase 3 — CI/CD

* [ ] GitHub Actions configured.
* [ ] Test stage implemented.
* [ ] Docker build implemented.
* [ ] ECR push implemented.
* [ ] Helm `values.yaml` update implemented.
* [ ] Infra repository update implemented.
* [ ] Helm chart created.
* [ ] Image parameterized.
* [ ] Replicas parameterized.
* [ ] Resources parameterized.
* [ ] `helm upgrade` tested.
* [ ] PRs configured for `develop`.
* [ ] `main` configured for production deployment.
* [ ] Manual production approval configured.

## Phase 4 — GitOps and Monitoring

* [ ] ArgoCD configured.
* [ ] Infra repository connected.
* [ ] `values.yaml` changes trigger deployment.
* [ ] Prometheus configured.
* [ ] Flask `/metrics` endpoint configured.
* [ ] Prometheus scraping verified.
* [ ] Grafana configured.
* [ ] RPS dashboard created.
* [ ] p95 latency dashboard created.
* [ ] Error-rate dashboard created.
* [ ] Alertmanager configured.
* [ ] Error rate > 1% for 5 minutes alert configured.
* [ ] Slack integration configured.
* [ ] End-to-end alert successfully tested.

---

# Final Success Criteria

The **Final Project — Building a Full-Scale DevOps Pipeline** is complete when the application can successfully progress through the complete lifecycle:

```text
Developer
    ↓
GitHub
    ↓
GitHub Actions
    ↓
Test
    ↓
Build
    ↓
Security Scan
    ↓
Amazon ECR
    ↓
Helm
    ↓
Infra Repository
    ↓
ArgoCD
    ↓
Amazon EKS
    ↓
Ingress + HTTPS
    ↓
Flask Application
    ↓
HPA Autoscaling
    ↓
Prometheus
    ↓
Grafana
    ↓
Alertmanager
    ↓
Slack
```

> [!TIP]
> **Final Project Goal:** Build a production-like DevOps platform that demonstrates the complete software delivery lifecycle:
>
> **Develop → Test → Build → Secure → Package → Deploy → Scale → Monitor → Alert**
>
> This final project progressively combines the technologies and concepts covered across **all 15 sessions** into one full-scale DevOps pipeline.
