# Session 15F — Implement E2E DevOps Architecture

## Lab 23 — End-to-End Pipeline: Code → Build → Deploy → Monitor

> [!NOTE]
> **Lab focus:** Connect all tools into a full DevOps pipeline for a production-like deployment.

---

## Objective

Connect all tools into a full DevOps pipeline for a **production-like deployment**.

---

## Full Architecture

The End-to-End DevOps architecture consists of the following components:

### Source Control

**GitHub**

* Git
* Branching
* Pull Requests (PRs)

### Continuous Integration (CI)

**GitHub Actions**

```text
Test → Build → Scan → Push to ECR
```

### Continuous Delivery (CD)

**ArgoCD**

GitOps pull-based deployment to Amazon EKS.

```text
GitHub → ArgoCD → EKS
```

### Infrastructure

**Terraform**

Terraform manages:

* VPC
* EKS
* RDS
* ALB

### Configuration Management

**Ansible**

Ansible is used for:

* Bootstrap EC2 nodes
* K8s node setup

### Secrets Management

**AWS Secrets Manager → External Secrets Operator in K8s**

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes
```

### Monitoring and Alerting

**Prometheus + Grafana + Alertmanager → Slack**

```text
Prometheus
    ↓
Grafana
    ↓
Alertmanager
    ↓
Slack
```

### Packaging

**Helm**

Helm charts are used for all Kubernetes deployments.

---

# Pipeline Flow

The complete pipeline follows this sequence:

```text
Developer pushes code
        ↓
GitHub Actions triggers
        ↓
1. Run tests (pytest)
        ↓
2. SonarQube quality gate
        ↓
3. Docker build
        ↓
4. Trivy image scan
   └─ Fail if CRITICAL CVE
        ↓
5. Push to ECR with git SHA tag
        ↓
6. Update image tag in Helm values.yaml
   └─ In infra repo
        ↓
7. Push to infra repo
   └─ Triggers ArgoCD
        ↓
ArgoCD detects change in infra repo
        ↓
8. Auto-sync: helm upgrade in EKS
        ↓
9. Rolling update — zero downtime
        ↓
10. Health check passes — sync HEALTHY
        ↓
Prometheus scrapes new pods
        ↓
11. Grafana dashboards update
        ↓
12. Alert rules evaluated
        ↓
13. On issue: Slack alert fires
```

---

## CI/CD Pipeline — Detailed Process

### Step 1 — Run Tests

GitHub Actions runs the application's automated tests using **pytest**.

```text
pytest
```

---

### Step 2 — SonarQube Quality Gate

The code is analyzed using **SonarQube**.

The pipeline must pass the configured SonarQube quality gate before continuing.

```text
GitHub Actions
      ↓
SonarQube
      ↓
Quality Gate
```

---

### Step 3 — Docker Build

The application is packaged into a Docker image.

```text
Application Source
        ↓
   Docker Build
        ↓
   Docker Image
```

---

### Step 4 — Trivy Image Scan

The Docker image is scanned using **Trivy**.

> [!WARNING]
> The pipeline must **fail if a CRITICAL CVE is detected**.

```text
Docker Image
     ↓
Trivy Image Scan
     ↓
CRITICAL CVE?
 ┌───────┴───────┐
 │               │
Yes              No
 │                │
Fail             Continue
Pipeline         Pipeline
```

---

### Step 5 — Push Image to ECR

The successfully validated Docker image is pushed to **Amazon Elastic Container Registry (ECR)**.

The image is tagged using the **Git SHA**.

```text
Docker Image
     ↓
Git SHA Tag
     ↓
Amazon ECR
```

---

### Step 6 — Update Helm Image Tag

The image tag in the Helm `values.yaml` file is updated.

The `values.yaml` file is located in the **infra repository**.

```text
infra repo
└── Helm chart
    └── values.yaml
        └── Image tag → Git SHA
```

---

### Step 7 — Push to Infrastructure Repository

The updated Helm configuration is pushed to the infrastructure repository.

This change triggers **ArgoCD**.

```text
GitHub Actions
      ↓
Infra Repository
      ↓
ArgoCD Trigger
```

---

# ArgoCD Deployment

ArgoCD detects the change in the infrastructure repository.

### Step 8 — Auto-Sync

ArgoCD automatically synchronizes the changes and performs the Helm upgrade in Amazon EKS.

```text
Infra Repository
       ↓
     ArgoCD
       ↓
helm upgrade
       ↓
      EKS
```

---

### Step 9 — Rolling Update

Kubernetes performs a **rolling update**.

The objective is:

> **Zero downtime**

```text
Existing Pods
      ↓
New Pods Start
      ↓
Health Checks
      ↓
Traffic Continues
      ↓
Old Pods Replaced
```

---

### Step 10 — Health Check

After deployment, Kubernetes health checks verify the new application pods.

The expected successful state is:

```text
Health Check → PASS
Sync Status  → HEALTHY
```

> [!IMPORTANT]
> The deployment should reach a **HEALTHY** synchronization state before the release is considered successful.

---

# Monitoring and Alerting

After the new pods are deployed, **Prometheus** scrapes metrics from the application.

### Step 11 — Grafana Dashboards Update

Prometheus collects metrics from the new pods.

Grafana uses those metrics to update dashboards.

```text
New Pods
   ↓
Prometheus
   ↓
Grafana
   ↓
Dashboards Update
```

---

### Step 12 — Alert Rules Evaluated

Configured alert rules are evaluated against the collected metrics.

```text
Metrics
   ↓
Prometheus
   ↓
Alert Rules
   ↓
Evaluation
```

---

### Step 13 — Slack Alert

When an issue is detected, an alert is sent through the alerting pipeline to Slack.

```text
Prometheus
    ↓
Alertmanager
    ↓
Slack
    ↓
Alert Fires
```

> [!TIP]
> The complete workflow is:
>
> **Code → Build → Deploy → Monitor → Alert**

---

# Task Set 1 — Guided

The Guided tasks focus on establishing the basic End-to-End DevOps pipeline.

| Task     | What to do                                                                              |
| -------- | --------------------------------------------------------------------------------------- |
| **T1.1** | Wire GitHub → GitHub Actions → ECR push. Verify new image appears in ECR after push.    |
| **T1.2** | Configure ArgoCD to watch your Helm chart in GitHub. Push change → watch deploy in K8s. |
| **T1.3** | Check Grafana: does your app appear as a scrape target? Can you see HTTP metrics?       |

---

## T1.1 — Wire GitHub → GitHub Actions → ECR Push

### Objective

Configure the pipeline so that code pushed to GitHub triggers GitHub Actions and results in a Docker image being pushed to Amazon ECR.

### Workflow

```text
GitHub
   ↓
GitHub Actions
   ↓
Docker Build
   ↓
Docker Image
   ↓
Amazon ECR
```

### Verification

After pushing code:

1. Verify that GitHub Actions is triggered.
2. Verify that the workflow completes successfully.
3. Open the ECR repository.
4. Verify that the new image appears.
5. Verify that the image uses the expected Git SHA tag.

> [!NOTE]
> **Expected result:** The new image should appear in ECR after the push.

---

## T1.2 — Configure ArgoCD to Watch Your Helm Chart

### Objective

Configure ArgoCD to monitor the Helm chart stored in GitHub.

### Task

1. Configure ArgoCD to watch your Helm chart in GitHub.
2. Push a change to the Helm chart.
3. Monitor ArgoCD.
4. Verify that ArgoCD detects the change.
5. Watch the deployment occur in Kubernetes.

### Workflow

```text
GitHub
  ↓
Helm Chart
  ↓
ArgoCD
  ↓
EKS
  ↓
Kubernetes Deployment
```

> [!TIP]
> Use the ArgoCD application status to monitor synchronization and deployment health.

---

## T1.3 — Check Grafana

### Objective

Verify that the application is being monitored by Prometheus and that its metrics are available in Grafana.

### Questions

Check Grafana:

* Does your app appear as a **scrape target**?
* Can you see **HTTP metrics**?

### Monitoring Flow

```text
Application Pods
       ↓
   Prometheus
       ↓
     Grafana
       ↓
  HTTP Metrics
```

---

# Task Set 2 — Practice

The Practice tasks introduce intentional failures, production incidents, and rollback operations.

| Task     | What to do                                                                                 |
| -------- | ------------------------------------------------------------------------------------------ |
| **T2.1** | Break the pipeline intentionally: introduce a failing test. Pipeline should block deploy.  |
| **T2.2** | Simulate production incident: OOM kill a pod. Watch Prometheus alert fire to Slack.        |
| **T2.3** | Implement rollback: push bad code → pipeline fails → ArgoCD auto rollback to last healthy. |

---

## T2.1 — Intentionally Break the Pipeline

### Objective

Verify that the CI/CD pipeline prevents failed code from being deployed.

### Task

Introduce a **failing test** intentionally.

```text
Failing Test
     ↓
GitHub Actions
     ↓
Pipeline Failure
     ↓
Deployment Blocked
```

### Expected Result

The pipeline should detect the failing test and **block deployment**.

> [!IMPORTANT]
> A failing test must prevent the application from progressing through the deployment pipeline.

---

## T2.2 — Simulate a Production Incident

### Objective

Simulate a production-style application incident and validate Prometheus and Slack alerting.

### Task

**OOM kill a pod.**

Then monitor the alerting workflow.

### Expected Flow

```text
Kubernetes Pod
      ↓
   OOM Kill
      ↓
  Prometheus
      ↓
   Alert Rule
      ↓
 Alertmanager
      ↓
     Slack
```

### Verification

Verify that:

1. The pod experiences an OOM kill.
2. Prometheus detects the relevant condition.
3. The Prometheus alert rule fires.
4. Alertmanager processes the alert.
5. The alert is delivered to Slack.

---

## T2.3 — Implement Rollback

### Objective

Test rollback to the last healthy application version.

### Task

Perform the following sequence:

```text
Push Bad Code
      ↓
Pipeline Fails
      ↓
ArgoCD
      ↓
Auto Rollback
      ↓
Last Healthy Version
```

### Expected Result

After the bad code is detected, **ArgoCD should automatically roll back to the last healthy version**.

> [!WARNING]
> Perform rollback testing in a controlled lab environment before applying automated rollback mechanisms to production workloads.

---

# Task Set 3 — Challenge

The Challenge tasks extend the DevOps architecture into DORA metrics, cost monitoring, and disaster recovery.

| Task     | What to do                                                                                     |
| -------- | ---------------------------------------------------------------------------------------------- |
| **T3.1** | DORA metrics dashboard: deployment frequency, lead time, MTTR, change failure rate in Grafana. |
| **T3.2** | Cost dashboard: AWS Cost Explorer + Grafana. Alert when daily spend exceeds $5.                |
| **T3.3** | Full disaster recovery test: terminate all EKS nodes. Time to full recovery via automation.    |

---

## T3.1 — DORA Metrics Dashboard

### Objective

Create a **DORA metrics dashboard** in Grafana.

### Metrics

The dashboard should include:

* **Deployment frequency**
* **Lead time**
* **MTTR**
* **Change failure rate**

### Dashboard Flow

```text
DevOps / CI-CD Data
        ↓
     Metrics
        ↓
     Grafana
        ↓
DORA Metrics Dashboard
```

---

## T3.2 — Cost Dashboard

### Objective

Create a cost dashboard using:

* **AWS Cost Explorer**
* **Grafana**

### Architecture

```text
AWS Cost Explorer
        ↓
      Cost Data
        ↓
      Grafana
        ↓
 Cost Dashboard
```

### Alert Requirement

Configure an alert when daily spending exceeds:

> **$5**

```text
Daily Spend
     ↓
 ┌───┴────┐
 │        │
≤ $5     > $5
 │        │
Normal   🚨 Alert
```

> [!IMPORTANT]
> **Daily spending threshold:** $5.

---

## T3.3 — Full Disaster Recovery Test

### Objective

Perform a full disaster recovery test by terminating all EKS nodes and measuring the time required for complete recovery through automation.

### Task

Terminate **all EKS nodes**.

### Recovery Flow

```text
Terminate All EKS Nodes
          ↓
    Recovery Process
          ↓
       Automation
          ↓
      EKS Recovery
          ↓
   Environment Restored
```

### Measure Recovery Time

Record the:

> **Time to full recovery via automation.**

### Expected Outcome

The environment should recover through automation, and the complete recovery time should be measured and documented.

---

# End-to-End Architecture

The complete architecture can be represented as follows:

```text
┌──────────────────────┐
│      Developer       │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│       GitHub         │
│  Git / Branch / PR   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│    GitHub Actions    │
│                      │
│ pytest               │
│ SonarQube             │
│ Docker Build         │
│ Trivy Scan           │
│ Push to ECR          │
│ Update Helm Tag      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Infrastructure Repo  │
│    Helm / values     │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│        ArgoCD        │
│   GitOps / AutoSync  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│       AWS EKS        │
│    Kubernetes Apps   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│      Prometheus      │
│   Metrics / Alerts   │
└──────────┬───────────┘
           │
           ├─────────────────┐
           ▼                 ▼
┌─────────────────┐  ┌─────────────────┐
│     Grafana     │  │  Alertmanager   │
│    Dashboard    │  │     Alerts      │
└─────────────────┘  └────────┬────────┘
                               │
                               ▼
                       ┌───────────────┐
                       │     Slack     │
                       │    Alerts     │
                       └───────────────┘
```

---

# Supporting Architecture Components

| Component                         | Technology                | Purpose                                |
| --------------------------------- | ------------------------- | -------------------------------------- |
| **Source**                        | GitHub                    | Git, branching, PRs                    |
| **CI**                            | GitHub Actions            | Test, build, scan, and push to ECR     |
| **Code Quality**                  | SonarQube                 | Quality gate                           |
| **Container Build**               | Docker                    | Build application images               |
| **Image Security**                | Trivy                     | Container image vulnerability scanning |
| **Container Registry**            | Amazon ECR                | Store container images                 |
| **CD**                            | ArgoCD                    | GitOps pull-based deployment           |
| **Kubernetes**                    | Amazon EKS                | Application deployment platform        |
| **Infrastructure**                | Terraform                 | VPC, EKS, RDS, ALB                     |
| **Configuration Management**      | Ansible                   | Bootstrap EC2 nodes / K8s node setup   |
| **Secrets**                       | AWS Secrets Manager       | Secure secret storage                  |
| **Kubernetes Secret Integration** | External Secrets Operator | Integrate secrets into K8s             |
| **Packaging**                     | Helm                      | Kubernetes application packaging       |
| **Monitoring**                    | Prometheus                | Metrics collection                     |
| **Visualization**                 | Grafana                   | Dashboards and metrics visualization   |
| **Alerting**                      | Alertmanager              | Alert processing                       |
| **Notifications**                 | Slack                     | Alert notifications                    |

---

# Lab Completion Checklist

## Task Set 1 — Guided

* [ ] **T1.1** Wire GitHub → GitHub Actions → ECR push.
* [ ] **T1.1** Verify that the new image appears in ECR after push.
* [ ] **T1.2** Configure ArgoCD to watch the Helm chart in GitHub.
* [ ] **T1.2** Push a change and verify deployment in Kubernetes.
* [ ] **T1.3** Verify that the application appears as a Prometheus scrape target.
* [ ] **T1.3** Verify that HTTP metrics are visible in Grafana.

## Task Set 2 — Practice

* [ ] **T2.1** Introduce a failing test intentionally.
* [ ] **T2.1** Verify that the pipeline blocks deployment.
* [ ] **T2.2** OOM kill a pod.
* [ ] **T2.2** Verify that Prometheus detects the incident.
* [ ] **T2.2** Verify that the alert reaches Slack.
* [ ] **T2.3** Push bad code.
* [ ] **T2.3** Verify pipeline failure.
* [ ] **T2.3** Verify ArgoCD auto rollback to the last healthy version.

## Task Set 3 — Challenge

* [ ] **T3.1** Create DORA metrics dashboard in Grafana.
* [ ] **T3.1** Add deployment frequency.
* [ ] **T3.1** Add lead time.
* [ ] **T3.1** Add MTTR.
* [ ] **T3.1** Add change failure rate.
* [ ] **T3.2** Integrate AWS Cost Explorer with Grafana.
* [ ] **T3.2** Create the cost dashboard.
* [ ] **T3.2** Configure an alert when daily spend exceeds **$5**.
* [ ] **T3.3** Terminate all EKS nodes.
* [ ] **T3.3** Recover the environment using automation.
* [ ] **T3.3** Record the time to full recovery.

---

# Final DevOps Workflow

```text
                    CODE
                      ↓
                   GitHub
                      ↓
                    TEST
                      ↓
             SonarQube Quality Gate
                      ↓
                    BUILD
                      ↓
                    Docker
                      ↓
                    SCAN
                      ↓
                    Trivy
                      ↓
                    PUSH
                      ↓
                Amazon ECR
                      ↓
              Update Helm Tag
                      ↓
               Infra Repository
                      ↓
                    ArgoCD
                      ↓
                  GitOps CD
                      ↓
                  Amazon EKS
                      ↓
            Rolling Update
             Zero Downtime
                      ↓
               Health Check
                      ↓
                   HEALTHY
                      ↓
                Prometheus
                      ↓
                   Grafana
                      ↓
              Alert Evaluation
                      ↓
                Alertmanager
                      ↓
                    Slack
```

