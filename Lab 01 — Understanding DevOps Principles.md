# Session 1 — DevOps Foundation & Course Overview

## Lab 01 — Understanding DevOps Principles

### 🎯 Objective

Understand the **DevOps lifecycle**, key principles such as **CI/CD, Collaboration, and Automation**, and how DevOps differs from traditional IT delivery.

---

## 📚 Concepts to Know

### SDLC — Software Development Life Cycle

The DevOps lifecycle can be represented through the following eight phases:

```text
Plan → Code → Build → Test → Release → Deploy → Operate → Monitor
```

### CI — Continuous Integration

**Continuous Integration (CI)** is the practice of:

* Merging code frequently.
* Automatically testing code on every push.

### CD — Continuous Delivery / Continuous Deployment

**Continuous Delivery/Deployment (CD)** involves automatically deploying applications to:

* Staging environments
* Production environments

> **Note:** Continuous Delivery and Continuous Deployment are related but distinct practices. The difference will be explored further in **Task Set 2**.

### IaC — Infrastructure as Code

**Infrastructure as Code (IaC)** means managing infrastructure through **code rather than manual clicks**.

Examples include defining and managing:

* Servers
* Networking
* Infrastructure resources
* Cloud environments

### Shift-Left

**Shift-Left** means moving activities such as **testing and security earlier in the development pipeline**.

The goal is to identify and address issues earlier rather than waiting until later stages of delivery.

---

## ☁️ Explore in AWS Console

> **Region:** `ap-south-1` — Mumbai

Follow these steps to explore the AWS services used within the DevOps ecosystem.

### Step 1 — Sign in to AWS

Log in to:

[https://console.aws.amazon.com](https://console.aws.amazon.com)

Select the AWS Region:

```text
ap-south-1 (Mumbai)
```

### Step 2 — Explore AWS CodePipeline

Search for:

```text
CodePipeline
```

**Purpose:** AWS CodePipeline is where **CI/CD pipelines** live in AWS.

### Step 3 — Explore AWS CloudFormation

Search for:

```text
CloudFormation
```

**Purpose:** AWS CloudFormation is an **AWS-native Infrastructure as Code (IaC) tool**.

### Step 4 — Explore Amazon CloudWatch

Search for:

```text
CloudWatch
```

**Purpose:** CloudWatch provides a **monitoring and logging hub** for AWS environments.

### Step 5 — Identify the DevOps Ecosystem

Observe how these AWS services work together to form a **DevOps ecosystem within one cloud platform**.

---

# 🟢 Task Set 1 — Guided

Follow the prompts below step by step.

| Task     | What to do                                                                                              |
| -------- | ------------------------------------------------------------------------------------------------------- |
| **T1.1** | Draw the DevOps lifecycle: write the 8 phases on paper or whiteboard.                                   |
| **T1.2** | Open AWS Console → list 5 services you recognise and match each to a lifecycle phase.                   |
| **T1.3** | Create a free GitHub account at [github.com](https://github.com) — this is your code collaboration hub. |

### T1.1 — Draw the DevOps Lifecycle

Write or draw the following **8 phases** on paper or a whiteboard:

```text
Plan → Code → Build → Test → Release → Deploy → Operate → Monitor
```

### T1.2 — Map AWS Services to Lifecycle Phases

Open the **AWS Console**.

Identify **5 AWS services you recognise** and associate each service with an appropriate DevOps lifecycle phase.

### T1.3 — Create a GitHub Account

Create a free GitHub account at:

[https://github.com](https://github.com)

GitHub will serve as your **code collaboration hub** throughout the learning journey.

---

# 🟡 Task Set 2 — Practice

Try the following tasks yourself using the concepts introduced in the guided exercise.

| Task     | What to do                                                                       |
| -------- | -------------------------------------------------------------------------------- |
| **T2.1** | Research: what is the difference between CD (Delivery) vs CD (Deployment)?       |
| **T2.2** | Find the AWS region selector — switch to N.Virginia, then switch back to Mumbai. |
| **T2.3** | Create a public GitHub repo named `devops-journey` with a README file.           |

### T2.1 — Research CD Delivery vs. CD Deployment

Research the difference between:

* **Continuous Delivery**
* **Continuous Deployment**

Consider how each approach handles deployment to staging and production environments.

### T2.2 — Change the AWS Region

Locate the **AWS Region selector** in the AWS Console.

Perform the following:

1. Switch to **N. Virginia**.
2. Confirm that the selected region has changed.
3. Switch back to **Mumbai** (`ap-south-1`).

### T2.3 — Create a GitHub Repository

Create a **public GitHub repository** with the following name:

```text
devops-journey
```

Initialize the repository with a:

```text
README.md
```

---

# 🔴 Task Set 3 — Challenge

Apply your knowledge to practical, real-world scenarios.

| Task     | What to do                                                                                 |
| -------- | ------------------------------------------------------------------------------------------ |
| **T3.1** | Write a 1-page 'DevOps Transformation Plan' for a fictional company moving from waterfall. |
| **T3.2** | Map AWS services to SDLC: CodeCommit → Code, CodeBuild → Build, CodeDeploy → Deploy, etc.  |
| **T3.3** | Push a Hello World HTML file to your GitHub repo using git CLI.                            |

### T3.1 — DevOps Transformation Plan

Write a **1-page "DevOps Transformation Plan"** for a fictional company transitioning from a **Waterfall** development model to DevOps.

Your plan should consider how the organization can introduce:

* DevOps principles
* CI/CD
* Collaboration
* Automation
* Infrastructure as Code
* Shift-Left testing and security

### T3.2 — Map AWS Services to the SDLC

Map AWS services to the appropriate SDLC phases.

For example:

```text
CodeCommit → Code
CodeBuild  → Build
CodeDeploy → Deploy
```

Extend the mapping to other relevant AWS services and SDLC phases.

### T3.3 — Push a Hello World HTML File Using Git CLI

Create a simple **Hello World HTML file** and push it to your GitHub repository using the **Git CLI**.

Example filename:

```text
index.html
```

The file should contain a basic Hello World page.

Use your GitHub repository:

```text
devops-journey
```

---

## 💡 Best-Practice Reminder

As you complete this lab, keep the following DevOps principles in mind:

* **Automate** repetitive tasks wherever possible.
* **Integrate code frequently** and validate changes automatically.
* **Collaborate** through version control and shared repositories.
* **Shift testing and security left** in the development lifecycle.
* **Manage infrastructure as code** rather than relying on manual configuration.
* **Monitor applications and infrastructure** continuously.

---

## ✅ Lab 01 Completion Checklist

* [ ] Understand the 8-phase DevOps lifecycle.
* [ ] Understand CI and CD concepts.
* [ ] Understand Infrastructure as Code.
* [ ] Understand the Shift-Left approach.
* [ ] Explore CodePipeline in AWS.
* [ ] Explore CloudFormation in AWS.
* [ ] Explore CloudWatch in AWS.
* [ ] Create a GitHub account.
* [ ] Create the `devops-journey` repository.
* [ ] Practice switching AWS regions.
* [ ] Create a DevOps Transformation Plan.
* [ ] Map AWS services to SDLC phases.
* [ ] Push a Hello World HTML file using Git CLI.
