# Session 7 — CI/CD with GitHub Actions

## Lab 14 — GitHub Actions Workflows — Build, Test & Deploy

> **🎯 Objective:** Create complete CI/CD workflows triggered by `push` and Pull Request events, including automated testing, Docker image builds, Amazon ECR publishing, and Amazon ECS deployment.

---

## 🧭 Overview

GitHub Actions provides a native automation platform for implementing CI/CD directly within GitHub repositories.

In this lab, you will build a CI/CD pipeline that can:

* Trigger automatically on `push` and Pull Request events
* Check out source code
* Configure Python
* Install application dependencies
* Run automated tests
* Build Docker images
* Authenticate with AWS
* Push images to Amazon ECR
* Deploy applications to Amazon ECS
* Use GitHub Secrets for credentials
* Test across multiple Python versions
* Upload test artifacts
* Cache dependencies
* Integrate SonarCloud quality checks
* Create reusable GitHub Actions workflows

---

# 🔄 CI/CD Pipeline Overview

The target workflow is:

```text
Developer
    │
    ▼
GitHub Repository
    │
    ├── Push
    │
    └── Pull Request
          │
          ▼
     GitHub Actions
          │
          ▼
       Checkout
          │
          ▼
      Install Python
          │
          ▼
     Install Dependencies
          │
          ▼
       Run Tests
          │
          ▼
      Build Docker Image
          │
          ▼
     Push Image to ECR
          │
          ▼
      Deploy to ECS
          │
          ▼
     Production 🚀
```

---

# 📁 Workflow File Location

GitHub Actions workflow files are stored inside:

```text
.github/
└── workflows/
    └── ci-cd.yml
```

> 💡 **Note:** GitHub automatically detects workflow files stored under `.github/workflows/`.

---

# ⚙️ Workflow Structure

Create:

```text
.github/workflows/ci-cd.yml
```

The following workflow implements the CI/CD pipeline:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  AWS_REGION: ap-south-1
  ECR_REPO: my-app

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install deps & Test
        run: |
          pip install -r requirements.txt
          pytest tests/ -v

  build-push:
    name: Build & Push to ECR
    needs: test
    runs-on: ubuntu-latest

    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - uses: aws-actions/amazon-ecr-login@v2

      - name: Build & Push
        run: |
          docker build -t $ECR_REPO:${{ github.sha }} .
          docker push $ECR_REPO:${{ github.sha }}

  deploy:
    name: Deploy to ECS
    needs: build-push
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Update ECS Service
        run: |
          aws ecs update-service \
            --cluster prod-cluster \
            --service my-app \
            --force-new-deployment
```

---

# 🧩 Workflow Components

## `name`

Defines the workflow name displayed in GitHub Actions:

```yaml
name: CI/CD Pipeline
```

---

## `on`

Defines the events that trigger the workflow:

```yaml
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
```

This workflow runs when:

* Code is pushed to `main`
* Code is pushed to `develop`
* A Pull Request targets `main`

---

## Environment Variables

```yaml
env:
  AWS_REGION: ap-south-1
  ECR_REPO: my-app
```

These values can be reused throughout the workflow.

---

# 🧪 Test Job

The test job runs on:

```yaml
runs-on: ubuntu-latest
```

Checkout the repository:

```yaml
- uses: actions/checkout@v4
```

Configure Python:

```yaml
- uses: actions/setup-python@v4
  with:
    python-version: '3.11'
```

Install dependencies and execute tests:

```yaml
- name: Install deps & Test
  run: |
    pip install -r requirements.txt
    pytest tests/ -v
```

---

# 🐳 Build & Push Job

The build job depends on successful testing:

```yaml
needs: test
```

This creates the dependency:

```text
test
  │
  │ successful
  ▼
build-push
```

The job is restricted to `main`:

```yaml
if: github.ref == 'refs/heads/main'
```

---

# 🔐 AWS Authentication

Configure AWS credentials:

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ env.AWS_REGION }}
```

> 🛡️ **Security:** AWS credentials should be stored as GitHub Secrets rather than directly inside workflow files.

---

# 📦 Amazon ECR Login

Authenticate Docker with Amazon ECR:

```yaml
- uses: aws-actions/amazon-ecr-login@v2
```

---

# 🏗️ Build and Push the Docker Image

Build the image:

```bash
docker build -t $ECR_REPO:${{ github.sha }} .
```

Push the image:

```bash
docker push $ECR_REPO:${{ github.sha }}
```

The `${{ github.sha }}` value provides a unique Git commit identifier that can be used as an image tag.

Example:

```text
my-app:9f8a7c6d...
```

> 💡 **Best Practice:** Immutable commit-based image tags make it easier to identify exactly which source revision produced a deployed container image.

---

# 🚀 Deploy to Amazon ECS

The deployment job requires a successful build:

```yaml
needs: build-push
```

The production environment is specified with:

```yaml
environment: production
```

The ECS service is updated with:

```bash
aws ecs update-service \
  --cluster prod-cluster \
  --service my-app \
  --force-new-deployment
```

This forces ECS to start a new deployment for the service.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                         |
| -------- | -------------------------------------------------------------------------------------------------- |
| **T1.1** | Create `.github/workflows/hello.yml`: trigger on push, run `echo 'Hello Actions'`. Push and watch. |
| **T1.2** | Add GitHub secrets: Settings → Secrets → `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`.             |
| **T1.3** | Create full CI workflow: checkout → setup Python → `pip install` → `pytest`. Commit and watch run. |

---

## T1.1 — Create Your First GitHub Actions Workflow

Create:

```text
.github/workflows/hello.yml
```

Add:

```yaml
name: Hello Actions

on:
  push:

jobs:
  hello:
    runs-on: ubuntu-latest

    steps:
      - name: Hello
        run: echo 'Hello Actions'
```

Commit the workflow:

```bash
git add .github/workflows/hello.yml
```

Commit:

```bash
git commit -m "ci: add hello actions workflow"
```

Push:

```bash
git push
```

Navigate to:

```text
GitHub Repository
   │
   └── Actions
        │
        └── Hello Actions
```

Observe the workflow execution.

---

## T1.2 — Configure GitHub Secrets

Open your GitHub repository.

Navigate to:

```text
Settings
   │
   └── Secrets and variables
        │
        └── Actions
```

Create the following repository secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

The workflow references them using:

```yaml
${{ secrets.AWS_ACCESS_KEY_ID }}
```

and:

```yaml
${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

> 🔐 **Important:** Never hard-code AWS credentials directly into a workflow file.

---

## T1.3 — Create a Full CI Workflow

Create:

```text
.github/workflows/ci.yml
```

Example:

```yaml
name: Python CI

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest tests/ -v
```

Commit and push:

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add Python test workflow"
git push
```

Open:

```text
GitHub → Actions
```

Verify:

```text
Checkout
   ↓
Setup Python
   ↓
Install dependencies
   ↓
Run tests
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                               |
| -------- | ---------------------------------------------------------------------------------------- |
| **T2.1** | Add matrix build: test on Python 3.9, 3.10, 3.11 simultaneously using `strategy.matrix`. |
| **T2.2** | Add artifact upload: upload `test-results/` so they're downloadable from the Actions UI. |
| **T2.3** | Add dependency caching: cache pip packages. Compare run time before and after caching.   |

---

## T2.1 — Matrix Testing

Modify the test job:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11']

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest tests/ -v
```

The workflow now creates three test jobs:

```text
              Test
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    Python    Python    Python
     3.9       3.10      3.11
       │        │        │
       ▼        ▼        ▼
     Tests    Tests     Tests
```

---

## T2.2 — Upload Test Results

If tests generate output in:

```text
test-results/
```

you can upload the directory as an artifact.

Example:

```yaml
- name: Upload test results
  uses: actions/upload-artifact@v4
  with:
    name: test-results-${{ matrix.python-version }}
    path: test-results/
```

After the workflow finishes:

```text
GitHub
  → Actions
    → Workflow Run
      → Artifacts
```

You can download the generated test results from the Actions UI.

> 💡 **Best Practice:** Artifacts are useful for test reports, build outputs, logs, and diagnostic files that need to be retained after a workflow finishes.

---

## T2.3 — Cache Python Dependencies

Use the built-in caching support from `setup-python`:

```yaml
- name: Setup Python
  uses: actions/setup-python@v4
  with:
    python-version: ${{ matrix.python-version }}
    cache: pip
```

Then install dependencies:

```yaml
- name: Install dependencies
  run: |
    pip install -r requirements.txt
```

Compare workflow execution times before and after caching.

Expected flow:

```text
First Run
    │
    ▼
Download Dependencies
    │
    ▼
Install
    │
    ▼
Test

Later Runs
    │
    ▼
Restore Cache
    │
    ▼
Install Remaining Dependencies
    │
    ▼
Test
```

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                                             |
| -------- | ---------------------------------------------------------------------------------------------------------------------- |
| **T3.1** | Full pipeline: test → Docker build → ECR push → ECS deploy. Trigger on merge to `main` only.                           |
| **T3.2** | Add SonarQube scan step using SonarCloud GitHub Action. Fail build if quality gate fails.                              |
| **T3.3** | Implement reusable workflow: extract common steps to `.github/workflows/reusable-build.yml`. Call from multiple repos. |

---

# T3.1 — Full CI/CD Pipeline

The target architecture is:

```text
Pull Request
     │
     ▼
    Test
     │
     ▼
 PR Review
     │
     ▼
 Merge to main
     │
     ▼
 Docker Build
     │
     ▼
 Amazon ECR
     │
     ▼
 Amazon ECS
     │
     ▼
 Production 🚀
```

A complete workflow can be structured as:

```yaml
name: Full CI/CD

on:
  push:
    branches:
      - main

env:
  AWS_REGION: ap-south-1
  ECR_REPO: my-app

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: pip

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest tests/ -v

  build-push:
    name: Build & Push to ECR
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker image
        run: |
          docker build \
            -t $ECR_REPO:${{ github.sha }} .

      - name: Push Docker image
        run: |
          docker push $ECR_REPO:${{ github.sha }}

  deploy:
    name: Deploy to ECS
    needs: build-push
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster prod-cluster \
            --service my-app \
            --force-new-deployment
```

> ⚠️ **Important:** The sample ECR commands above use `$ECR_REPO` as supplied in the original workbook. In an actual AWS deployment, Docker normally needs the complete ECR registry URI, such as `<account-id>.dkr.ecr.<region>.amazonaws.com/my-app`, when tagging and pushing the image.

---

# T3.2 — Add SonarCloud Quality Checks

SonarCloud can be incorporated into the CI pipeline to analyze code quality and security.

Example:

```yaml
- name: SonarCloud Scan
  uses: SonarSource/sonarqube-scan-action@v6
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

A typical pipeline becomes:

```text
Checkout
   │
   ▼
Install Dependencies
   │
   ├───────────────┐
   ▼               ▼
Run Tests      SonarCloud
   │               │
   └───────┬───────┘
           ▼
     Quality Gate
           │
     ┌─────┴─────┐
     │           │
   PASS         FAIL
     │           │
     ▼           ▼
 Docker Build   Stop
```

> 🛡️ **Best Practice:** Do not allow production deployment when required security or quality gates fail.

---

## SonarCloud Configuration

Store the SonarCloud token as a GitHub Secret:

```text
SONAR_TOKEN
```

Use:

```yaml
env:
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Depending on the SonarCloud project configuration, you may also need project metadata such as:

```text
sonar-project.properties
```

Example:

```properties
sonar.projectKey=my-project
sonar.organization=my-organization
```

---

# T3.3 — Reusable GitHub Actions Workflow

Reusable workflows allow common CI/CD logic to be maintained centrally and called from multiple repositories.

Create:

```text
.github/workflows/reusable-build.yml
```

Example:

```yaml
name: Reusable Build

on:
  workflow_call:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: pip

      - name: Install dependencies
        run: |
          pip install -r requirements.txt

      - name: Run tests
        run: |
          pytest tests/ -v
```

A calling workflow can use:

```yaml
name: Application CI

on:
  push:
    branches:
      - main

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
```

The reusable workflow can then serve as a common foundation for multiple repositories.

---

# 🔐 GitHub Actions Security

GitHub Actions workflows frequently interact with production infrastructure. Treat workflow files as production automation code.

## Never Hard-Code Credentials

❌ Avoid:

```yaml
aws-access-key-id: AKIAxxxxxxxx
aws-secret-access-key: secret-value
```

Use GitHub Secrets instead:

```yaml
aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## Recommended AWS Authentication

For production CI/CD, consider using **GitHub Actions OIDC with AWS IAM roles** instead of long-lived AWS access keys.

Conceptually:

```text
GitHub Actions
      │
      │ OIDC Token
      ▼
AWS IAM
      │
      ▼
Assume Role
      │
      ▼
Temporary AWS Credentials
      │
      ▼
ECR / ECS / Other AWS Services
```

This reduces the need to store long-lived AWS credentials in GitHub.

---

# 🧪 Workflow Troubleshooting

## View Workflow Runs

Navigate to:

```text
GitHub Repository
   │
   └── Actions
```

Select the failed workflow and inspect the failed job.

---

## Check Individual Steps

A workflow is divided into jobs and steps:

```text
Workflow
 ├── test
 │    ├── Checkout
 │    ├── Setup Python
 │    ├── Install dependencies
 │    └── Run tests
 │
 ├── build-push
 │    ├── Checkout
 │    ├── AWS authentication
 │    ├── ECR login
 │    ├── Docker build
 │    └── Docker push
 │
 └── deploy
      └── ECS deployment
```

Identify which step failed before troubleshooting the entire pipeline.

---

## Common Failure Areas

### Python Dependency Installation

Check:

```bash
pip install -r requirements.txt
```

Verify that:

```text
requirements.txt
```

exists in the repository.

---

### Test Failure

Run locally:

```bash
pytest tests/ -v
```

Fix the test or application issue before pushing again.

---

### AWS Authentication Failure

Check:

* AWS credentials
* GitHub Secrets
* AWS region
* IAM permissions
* OIDC configuration if used

---

### ECR Push Failure

Check:

* ECR repository exists
* Correct AWS region
* ECR authentication
* IAM permissions
* Image tag and registry URI

---

### ECS Deployment Failure

Check:

* ECS cluster name
* ECS service name
* IAM permissions
* Task definition
* Container image
* ECS service events

---

# 🛡️ CI/CD Best Practices

* Keep workflows under `.github/workflows/`.
* Run automated tests before deployment.
* Protect the `main` branch.
* Require Pull Request reviews.
* Use GitHub Secrets for sensitive values.
* Prefer AWS OIDC over long-lived AWS access keys.
* Use immutable Docker image tags such as commit SHA.
* Scan source code and dependencies.
* Add quality gates before production deployment.
* Use reusable workflows for common processes.
* Cache dependencies to reduce build time.
* Upload useful test artifacts.
* Separate test, build, and deployment jobs.
* Use environment protection for production.
* Restrict production deployments to approved branches.
* Keep CI/CD logs free from credentials and sensitive information.
* Give AWS IAM roles only the permissions required by the pipeline.

---

# 📋 Lab 14 Completion Checklist

* [ ] Create `.github/workflows/hello.yml`.
* [ ] Trigger a workflow from a Git push.
* [ ] View workflow execution in GitHub Actions.
* [ ] Configure GitHub Actions Secrets.
* [ ] Create a Python CI workflow.
* [ ] Configure `actions/checkout`.
* [ ] Configure Python with `setup-python`.
* [ ] Install Python dependencies.
* [ ] Run `pytest`.
* [ ] Implement a Python matrix build.
* [ ] Test Python 3.9.
* [ ] Test Python 3.10.
* [ ] Test Python 3.11.
* [ ] Upload test results as artifacts.
* [ ] Enable pip dependency caching.
* [ ] Build a Docker image.
* [ ] Authenticate with AWS.
* [ ] Authenticate Docker with Amazon ECR.
* [ ] Push an image to ECR.
* [ ] Deploy to Amazon ECS.
* [ ] Restrict production deployment to `main`.
* [ ] Add SonarCloud analysis.
* [ ] Implement a quality gate.
* [ ] Create a reusable workflow.
* [ ] Call a reusable workflow.
* [ ] Understand GitHub Actions job dependencies.
* [ ] Understand GitHub Actions secrets.
* [ ] Understand CI/CD security practices.

---

# 🧠 Lab 14 — Key Takeaways

| Component             | Purpose                                                 |
| --------------------- | ------------------------------------------------------- |
| **GitHub Actions**    | Automates CI/CD workflows                               |
| **Workflow**          | Defines an automation process                           |
| **Job**               | A collection of steps executed on a runner              |
| **Step**              | Individual action or command                            |
| **Trigger**           | Event that starts a workflow                            |
| **Runner**            | Environment where a job executes                        |
| **Secrets**           | Secure storage for sensitive values                     |
| **Matrix**            | Runs jobs across multiple configurations                |
| **Artifact**          | Stores files generated by workflows                     |
| **Cache**             | Reuses dependencies to speed up workflows               |
| **Amazon ECR**        | Container image registry                                |
| **Amazon ECS**        | Container orchestration service                         |
| **SonarCloud**        | Code quality and security analysis                      |
| **Reusable Workflow** | Shared workflow logic used by multiple workflows        |
| **OIDC**              | Short-lived identity mechanism for cloud authentication |
| **CI**                | Continuous Integration                                  |
| **CD**                | Continuous Delivery/Deployment                          |

---

# 🚀 End-to-End Production CI/CD Model

```text
                         Developer
                             │
                             ▼
                       GitHub Repository
                             │
                    ┌────────┴────────┐
                    │                 │
                 Pull Request       Push
                    │                 │
                    ▼                 ▼
                 CI Tests          CI Tests
                    │                 │
                    ▼                 │
              Code Quality            │
                    │                 │
                    ▼                 │
              Security Scan            │
                    │                 │
             ┌──────┴──────┐          │
             │             │          │
           PASS           FAIL        │
             │             │          │
             │          ❌ Stop       │
             │                        │
             ▼                        │
          PR Review                   │
             │                        │
             ▼                        │
        Merge to main ◄───────────────┘
             │
             ▼
       Docker Build
             │
             ▼
        Amazon ECR
             │
             ▼
      Production Approval
             │
             ▼
        Amazon ECS
             │
             ▼
       Production 🚀
```

> **🎓 Lab Outcome:** By completing Lab 14, you should be able to build a practical GitHub Actions CI/CD pipeline that automatically tests application code, builds and publishes container images to Amazon ECR, deploys workloads to Amazon ECS, performs quality checks, securely handles credentials, and reuses common CI/CD logic across repositories.
