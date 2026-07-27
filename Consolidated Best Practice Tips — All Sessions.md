# Consolidated Best Practice Tips — All Sessions

> [!NOTE]
> This section consolidates the key **IT, DevOps, Cloud, Infrastructure, Security, and Automation best practices** covered across all sessions.
>
> These recommendations are intended to improve **security, reliability, maintainability, reproducibility, automation, and operational efficiency** across development and production environments.

---

# Linux & Shell

Apply the following practices when working with Linux systems and Bash scripts.

### 🐧 Shell Scripting Best Practices

* Always use `set -euo pipefail` at the top of every Bash script.
* Quote all variables:

```bash
"$VAR"
```

This prevents word-splitting bugs.

* Never run production workloads as `root` — create dedicated service users.
* Use `logrotate` for `/var/log` management — prevents disk-full incidents.
* Add the following aliases to `~/.bashrc` for improved readability:

```bash
alias grep='grep --color=auto'
alias ls='ls --color=auto'
```

> [!IMPORTANT]
> Running production workloads with unnecessary root privileges increases the potential impact of a security compromise.

---

# Docker

### 🐳 Container Image Best Practices

* Pin exact image versions:

```text
nginx:1.25.3
```

Never use:

```text
latest
```

This ensures reproducibility.

* Use **multi-stage builds** — reduces final image size by **60–80%**.
* Always run as a non-root `USER` in the Dockerfile — a security requirement in most organizations.
* Add a `HEALTHCHECK` to every Dockerfile — ECS/K8s uses this for container management.
* Use `.dockerignore` to exclude unnecessary files from the build context:

```text
.git
node_modules
__pycache__
*.env
```

* Scan images before pushing:

```bash
trivy image myapp:1.0
```

* Block **CRITICAL CVEs** in CI.

> [!WARNING]
> Never rely on the `latest` tag for production container deployments. Pin versions to ensure consistent and reproducible deployments.

---

# Kubernetes

### ☸️ Kubernetes Best Practices

* Always set **resource requests AND limits** — prevents noisy-neighbour pod starvation.
* Use the `RollingUpdate` strategy with:

```yaml
maxUnavailable: 0
```

for zero-downtime deployments.

* Add **liveness AND readiness probes** to **ALL production pods**.
* Use **Namespaces** to isolate:

```text
dev
staging
prod
```

Apply RBAC per namespace.

* Never use the `latest` image tag in Kubernetes manifests.

Use:

* Git SHA
* Semantic versioning (`semver`)

Example:

```text
myapp:8f3a21c
myapp:1.4.2
```

* Back up `etcd` regularly, or use managed Kubernetes such as **EKS** to offload this responsibility.

> [!TIP]
> Resource requests help Kubernetes schedule workloads appropriately, while limits prevent a workload from consuming excessive resources.

---

# Helm

### ⎈ Helm Best Practices

* Use the **Helm Diff plugin** before upgrades to preview changes in production.

* Pin chart versions in CI:

```bash
helm install myapp bitnami/nginx --version 15.3.0
```

* Use the `--atomic` flag:

```bash
helm upgrade --atomic
```

This enables automatic rollback on failure.

* Store Helm values in Git — treat `values.yaml` changes as deployments.
* Use the **Helm Secrets plugin** to encrypt sensitive `values.yaml` fields.

> [!IMPORTANT]
> Helm configuration stored in Git should be treated as deployment configuration and managed using the same review and change-control process as application code.

---

# Git

### 🔀 Git Best Practices

Use **Conventional Commits** for commit messages:

```text
feat:
fix:
chore:
docs:
ci:
```

### Branch Protection

* Never force-push to `main` or `develop` branches.
* Use Pull Requests with required reviews.
* Prefer:

```bash
git revert
```

over:

```bash
git reset
```

for shared branches — keeps history clean.

* Protect the `main` branch.

* Require:

  * **1+ PR review**
  * CI pass
  * No direct push

* Use `.gitignore` before the first commit — fixing it later is painful.

### Example Conventional Commits

```text
feat: add Redis health check
fix: resolve Flask startup issue
chore: update Docker base image
docs: update deployment guide
ci: add Trivy security scan
```

---

# GitHub Actions

### ⚙️ CI/CD Best Practices

* Cache dependencies such as:

```text
pip
npm
maven
```

between runs — can make builds approximately **50% faster**.

* Store **ALL credentials as GitHub Secrets**.

Never store credentials directly in workflow YAML files.

* Use environment protection rules.

Production should require **manual approval**.

* Run matrix tests using multiple Python/Node versions in parallel for greater test coverage.

* Use **reusable workflows** to extract common jobs and avoid duplication across repositories.

### Example Matrix Concept

```text
GitHub Actions
      │
      ├── Python 3.x
      ├── Python 3.x
      └── Python 3.x
             │
             ▼
       Parallel Tests
```

> [!WARNING]
> Never hard-code passwords, API keys, access tokens, or other credentials into GitHub Actions workflow files.

---

# GitOps / ArgoCD

### 🚀 GitOps Best Practices

* **Git is the single source of truth.**
* Ban `kubectl apply` in production.
* Separate:

  * **Application repository** — code
  * **Infrastructure repository** — manifests

This provides a clean GitOps pattern.

* Enable:

```yaml
selfHeal: true
```

ArgoCD automatically reverts manual `kubectl` changes.

* Use the **App of Apps** pattern to manage many services from one ArgoCD application.
* Configure RBAC in ArgoCD:

| Role           | Access      |
| -------------- | ----------- |
| Developers     | Read + Sync |
| Administrators | Full access |

### GitOps Flow

```text
Git
 ↓
Infra Repository
 ↓
ArgoCD
 ↓
Kubernetes
```

> [!IMPORTANT]
> In a GitOps environment, changes should be made through Git rather than directly modifying production Kubernetes resources.

---

# Prometheus & Grafana

### 📊 Monitoring Best Practices

* Set storage retention:

```text
--storage.tsdb.retention.time=30d
```

This helps prevent disk-full incidents.

* Use **recording rules** for expensive PromQL queries — pre-compute metrics at scrape time.
* Alert on **symptoms**, such as:

```text
High latency
High error rate
```

rather than only causes such as:

```text
High CPU
```

* Version-control Grafana dashboards as JSON in Git — provides reproducibility.
* Test alerts using:

```bash
amtool
```

Fire test alerts before real incidents happen.

### Recommended Monitoring Model

```text
Application
     ↓
  Metrics
     ↓
 Prometheus
     ↓
  Grafana
     ↓
Dashboards

Prometheus
     ↓
Alert Rules
     ↓
Alertmanager
     ↓
Notification
```

> [!TIP]
> Monitoring should focus on user-visible symptoms such as latency, availability, throughput, and error rates rather than relying exclusively on infrastructure metrics.

---

# AWS

### ☁️ AWS Best Practices

* Never use the **root account** for routine administration.
* Create an IAM administrator user with MFA on Day 1.
* Tag every resource with:

```text
Environment
Owner
Project
CostCenter
```

* Enable the following security services in every account:

```text
CloudTrail
AWS Config
GuardDuty
```

These form the security baseline.

* Use **IAM Roles**, rather than Access Keys, for:

```text
EC2
Lambda
```

Keys require rotation, while roles don't expire.

* Use **Multi-AZ** for all stateful services such as:

```text
RDS
ElastiCache
```

Single-AZ is not highly available.

* Set billing alerts at:

```text
$1
$5
$10
```

This helps identify cost patterns early.

> [!IMPORTANT]
> AWS resource tagging is essential for cost allocation, ownership tracking, inventory management, and operational governance.

---

# Terraform

### 🏗️ Infrastructure as Code Best Practices

* Use remote state in:

```text
S3 + DynamoDB locking
```

This is mandatory for team use.

* Always run:

```bash
terraform validate
terraform plan
```

before:

```bash
terraform apply
```

### Protect Production Resources

Use:

```hcl
lifecycle {
  prevent_destroy = true
}
```

for:

* RDS

* S3

* Production resources

* Use community modules such as:

```text
terraform-aws-modules
```

These are battle-tested and avoid reinventing infrastructure modules.

* Never commit the following to Git:

```text
.tfstate
.terraform/
```

Add them to `.gitignore`.

### Recommended Terraform Workflow

```text
Terraform Code
      ↓
terraform validate
      ↓
terraform plan
      ↓
Code Review
      ↓
terraform apply
      ↓
AWS Infrastructure
```

> [!WARNING]
> Terraform state can contain sensitive information. Never commit Terraform state files to a source-control repository.

---

# Ansible

### 🔧 Ansible Best Practices

* Maintain **idempotency**.

Run the playbook twice:

```text
First Run  → Changes
Second Run → 0 changes
```

The second run should show **0 changes** when the target system already matches the desired state.

* Use:

```bash
ansible-playbook --check --diff
```

before the first production apply.

This allows you to preview changes.

* Run `ansible-lint` against playbooks in CI — catches issues before they reach production.
* Use **Ansible Vault** for all secrets.

Never store plaintext passwords in inventory files.

* Use dynamic inventory for cloud environments:

```text
aws_ec2 plugin
```

This automatically discovers instances using tags.

### Ansible Workflow

```text
Playbook
   ↓
ansible-lint
   ↓
--check --diff
   ↓
Review
   ↓
Apply
   ↓
Verify Idempotency
```

---

# Security — General

### 🔐 Security Best Practices

* **Secrets never belong in code or Git.**

Use:

```text
AWS Secrets Manager
Vault
Kubernetes Secrets
```

* Follow the **Principle of Least Privilege**.

Every IAM role and policy should have the minimum permissions required.

* Rotate credentials:

```text
IAM Access Keys → Maximum 90 days
DB Passwords    → Quarterly
```

* Scan containers using:

```text
Trivy in CI
ECR scan on push
```

Block **CRITICAL CVEs**.

* Implement network segmentation.

Use:

```text
Private Subnets
    ↓
Databases
```

and:

```text
NAT
 ↓
Outbound Internet Access
```

### Recommended Network Architecture

```text
                     Internet
                         │
                         ▼
                    Public Layer
                         │
                         ▼
                   Load Balancer
                         │
                         ▼
                 Private Application
                      Subnets
                         │
                         ▼
                  Private Database
                      Subnets
```

> [!IMPORTANT]
> Databases should reside in private subnets and should not be directly exposed to the public Internet. Outbound access should be controlled through appropriate network components such as NAT.

---

# Cross-Platform Best Practice Summary

The recommendations across all sessions can be summarized into the following operational principles:

| Area                  | Core Best Practice                                                                   |
| --------------------- | ------------------------------------------------------------------------------------ |
| 🐧 Linux              | Secure shell scripts and avoid running workloads as root                             |
| 🐳 Docker             | Use pinned versions, multi-stage builds, health checks, and non-root users           |
| ☸️ Kubernetes         | Define resources, probes, namespaces, RBAC, and controlled image versions            |
| ⎈ Helm                | Version charts, preview changes, use atomic upgrades, and store configuration in Git |
| 🔀 Git                | Use Conventional Commits, PRs, branch protection, and controlled history             |
| ⚙️ GitHub Actions     | Secure secrets, cache dependencies, use approvals, and reusable workflows            |
| 🚀 ArgoCD             | Git as the single source of truth with automated reconciliation                      |
| 📊 Prometheus/Grafana | Monitor symptoms, retain metrics appropriately, version dashboards, and test alerts  |
| ☁️ AWS                | Use IAM, MFA, tagging, security services, Multi-AZ, and cost alerts                  |
| 🏗️ Terraform         | Remote state, plan before apply, protected resources, and no state in Git            |
| 🔧 Ansible            | Idempotency, linting, dry runs, Vault, and dynamic inventory                         |
| 🔐 Security           | Least privilege, secret management, vulnerability scanning, and network segmentation |

---

# Golden Rules for Production

> [!IMPORTANT]
> The following rules should be treated as the core operational standards across the entire DevOps environment.

### 1. 🔐 Never expose secrets

```text
Secrets → Secret Manager / Vault
Never → Git / Source Code
```

### 2. 🛡️ Never deploy known critical vulnerabilities

```text
Build
 ↓
Scan
 ↓
CRITICAL CVE
 ↓
BLOCK
```

### 3. 🔄 Git should be the source of truth

```text
Git
 ↓
CI/CD / GitOps
 ↓
Production
```

### 4. 👤 Never use unnecessary root privileges

Use dedicated service accounts and least-privilege IAM roles.

### 5. 📦 Pin versions

Avoid:

```text
latest
```

Prefer:

```text
Git SHA
Semantic Version
Exact Image Version
Exact Helm Chart Version
```

### 6. 📊 Monitor user-impacting symptoms

Prioritize:

```text
Availability
Latency
Error Rate
Traffic
```

### 7. 🧪 Test failure and recovery

Regularly test:

```text
Pipeline Failures
Rollback
Alerts
Disaster Recovery
```

### 8. 🔍 Review before production changes

Use:

```text
Git PR
CI
terraform plan
helm diff
ansible --check --diff
Manual Approval
```

### 9. 🏷️ Tag everything

Use consistent AWS tags:

```text
Environment
Owner
Project
CostCenter
```

### 10. 🔁 Automate repeatable operations

The ultimate objective is:

```text
Manual Work
    ↓
Automation
    ↓
Repeatability
    ↓
Consistency
    ↓
Reliability
```

---

# Overall DevOps Best Practice Model

```text
                         ┌──────────────────┐
                         │      SECURITY    │
                         │ Secrets / IAM /  │
                         │ Vulnerability    │
                         └────────┬─────────┘
                                  │
                                  ▼
┌──────────┐     ┌──────────┐   ┌──────────┐    ┌──────────┐
│   CODE   │ ──► │   BUILD  │ ─►│   TEST   │ ──►│   SCAN   │
└──────────┘     └──────────┘   └──────────┘    └────┬─────┘
                                                      │
                                                      ▼
                                               ┌──────────┐
                                               │  DEPLOY  │
                                               └────┬─────┘
                                                    │
                                                    ▼
                                               ┌──────────┐
                                               │  SCALE   │
                                               └────┬─────┘
                                                    │
                                                    ▼
                                               ┌──────────┐
                                               │ MONITOR  │
                                               └────┬─────┘
                                                    │
                                                    ▼
                                               ┌──────────┐
                                               │  ALERT   │
                                               └────┬─────┘
                                                    │
                                                    ▼
                                               ┌──────────┐
                                               │ RECOVER  │
                                               └────┬─────┘
                                                    │
                                                    └──────►
                                                     Improve
```

> [!TIP]
> **The central principle across all sessions is simple:**
>
> **Secure it → Version it → Automate it → Test it → Monitor it → Recover it → Improve it.**
