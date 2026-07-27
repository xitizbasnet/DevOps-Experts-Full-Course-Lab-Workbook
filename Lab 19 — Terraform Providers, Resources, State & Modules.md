# Session 11 — Terraform: Infrastructure as Code

> **🎯 Objective:** Write Terraform configuration to provision AWS infrastructure such as VPCs, EC2 instances, and Security Groups. Learn Terraform providers, resources, state management, modules, workspaces, remote state, and infrastructure testing.

---

## 📘 Lab 19 — Terraform Providers, Resources, State & Modules

Terraform is an **Infrastructure as Code (IaC)** tool that allows DevOps engineers to define infrastructure using declarative configuration files.

In this lab, you will work with:

* **Terraform Providers**
* **AWS Resources**
* **Data Sources**
* **Terraform State**
* **Remote State**
* **S3 Backend**
* **DynamoDB State Locking**
* **Terraform Modules**
* **Terraform Workspaces**
* **Terratest**
* **Terraform Plan / Apply / Destroy**

---

# 🎯 Learning Objectives

By completing this lab, you will learn how to:

* Configure the AWS Terraform provider.
* Define AWS infrastructure as code.
* Dynamically discover an Ubuntu AMI.
* Create EC2 instances with Terraform.
* Use Terraform variables and outputs.
* Understand Terraform state.
* Configure remote state in Amazon S3.
* Configure state locking.
* Use community Terraform modules.
* Build reusable infrastructure.
* Use Terraform workspaces for different environments.
* Deploy complete AWS infrastructure using Terraform.
* Test Terraform infrastructure using Terratest.

---

# 🏗️ Terraform Workflow

```text
                    Terraform Configuration
                              │
                              ▼
                         terraform init
                              │
                              ▼
                        terraform validate
                              │
                              ▼
                          terraform plan
                              │
                              ▼
                         terraform apply
                              │
                              ▼
                       AWS Infrastructure
                              │
                              ▼
                       Terraform State
                              │
                              ▼
                         terraform show
```

---

# 📁 Recommended Project Structure

A simple starting structure:

```text
terraform-lab/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── .gitignore
```

For a larger project:

```text
terraform-lab/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── terraform.tfvars
├── modules/
│   ├── network/
│   ├── compute/
│   └── security/
└── tests/
```

> 🔐 **Security Note:** Never commit `terraform.tfstate`, `.tfstate.backup`, AWS credentials, or secrets to Git.

---

# ⚙️ Main Terraform Configuration

The following configuration defines the Terraform version, AWS provider, S3 backend, provider region, AMI data source, EC2 instance, and output.

## `main.tf`

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "my-tf-state-bucket"
    key            = "devops/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "tf-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.region
}

variable "region" {
  default = "ap-south-1"
}

# Data source — dynamic AMI lookup
data "aws_ami" "ubuntu" {
  most_recent = true

  owners = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  tags = {
    Name        = "tf-web"
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}

output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

> ⚠️ **Remote State Prerequisite:** The S3 bucket and state-locking mechanism referenced in the backend must exist and be configured before using this backend configuration. Also ensure the AWS account has the required permissions.

---

# 🔍 Understanding the Configuration

## Terraform Block

```hcl
terraform {
  required_version = ">= 1.5"
}
```

This specifies the minimum Terraform version supported by the configuration.

---

## AWS Provider

```hcl
required_providers {
  aws = {
    source  = "hashicorp/aws"
    version = "~> 5.0"
  }
}
```

The AWS provider allows Terraform to communicate with AWS APIs.

---

## AWS Region

```hcl
provider "aws" {
  region = var.region
}
```

The region is supplied through a Terraform variable.

Default:

```hcl
variable "region" {
  default = "ap-south-1"
}
```

---

# 🗃️ Terraform Backend

The configuration uses Amazon S3 for remote Terraform state:

```hcl
backend "s3" {
  bucket         = "my-tf-state-bucket"
  key            = "devops/terraform.tfstate"
  region         = "ap-south-1"
  dynamodb_table = "tf-state-lock"
  encrypt        = true
}
```

Conceptually:

```text
                 Terraform
                     │
                     ▼
              Remote State
                     │
                     ▼
              Amazon S3 Bucket
                     │
                     ├── terraform.tfstate
                     │
                     ▼
               State Locking
```

> 💡 **Important:** Terraform's current S3 backend supports native S3 state locking through the `use_lockfile` setting. The workbook's `dynamodb_table` approach represents the traditional DynamoDB locking pattern and may be encountered in existing environments. For new implementations, consult the Terraform version's current S3 backend documentation and choose the supported locking approach.

---

# 🔎 Data Source — Dynamic AMI Lookup

Instead of hardcoding an AMI ID, Terraform dynamically searches for the latest matching Ubuntu image:

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true

  owners = ["099720109477"]

  filter {
    name   = "name"

    values = [
      "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
    ]
  }
}
```

The EC2 resource then uses:

```hcl
ami = data.aws_ami.ubuntu.id
```

This is useful because AMI IDs vary by:

* AWS Region
* Operating system
* Architecture
* Image release

---

# 🖥️ EC2 Resource

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  tags = {
    Name        = "tf-web"
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}
```

Terraform will create an EC2 instance based on the declared configuration.

---

# 📤 Terraform Output

```hcl
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

After deployment:

```bash
terraform output
```

Example:

```text
instance_ip = "203.0.113.10"
```

---

# 🛠️ Core Terraform Commands

## Initialize

```bash
terraform init
```

Downloads:

* Terraform providers
* Required modules
* Backend configuration

---

## Validate

```bash
terraform validate
```

Checks the configuration for syntax and structural problems.

---

## Preview Changes

```bash
terraform plan
```

Shows what Terraform intends to change.

---

## Apply Infrastructure

```bash
terraform apply
```

Terraform prompts for confirmation.

---

## Apply Without Prompt

```bash
terraform apply -auto-approve
```

> ⚠️ **Production Warning:** Avoid `-auto-approve` for production deployments unless the execution process has appropriate safeguards and approval controls.

---

## View State

```bash
terraform show
```

Displays the current Terraform-managed infrastructure state.

---

## View Outputs

```bash
terraform output
```

---

## Destroy Infrastructure

```bash
terraform destroy
```

> 🛑 **Warning:** `terraform destroy` can permanently delete infrastructure. Always review the plan and confirm the target workspace/environment before execution.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                                |
| -------- | --------------------------------------------------------------------------------------------------------- |
| **T1.1** | Create `main.tf` with EC2 resource. Run `init → validate → plan → apply`. Note instance ID.               |
| **T1.2** | `terraform show` — see the full state. `terraform output` — see `instance_ip`.                            |
| **T1.3** | Add a tag to EC2 in `.tf` file. `terraform plan` — see `~update`. `terraform apply` — tag added in-place. |

---

# T1.1 — Create Your First EC2 Instance

Create:

```text
main.tf
```

Add the Terraform configuration.

Initialize:

```bash
terraform init
```

Validate:

```bash
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

Generate a plan:

```bash
terraform plan
```

Apply:

```bash
terraform apply
```

Confirm when prompted.

Verify the instance:

```bash
terraform show
```

You can also query the AWS CLI:

```bash
aws ec2 describe-instances \
  --region ap-south-1
```

---

# T1.2 — Inspect Terraform State

Run:

```bash
terraform show
```

This displays resources Terraform currently knows about.

Then:

```bash
terraform output
```

You should see:

```text
instance_ip = "..."
```

---

# T1.3 — Modify EC2 Tags

Update:

```hcl
tags = {
  Name        = "tf-web"
  Environment = "dev"
  ManagedBy   = "terraform"
  Owner       = "DevOps"
}
```

Run:

```bash
terraform plan
```

Terraform should show an in-place modification.

The plan notation:

```text
~ update in-place
```

means the resource can be modified without being destroyed and recreated.

Apply:

```bash
terraform apply
```

Verify:

```bash
terraform show
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                              |
| -------- | ------------------------------------------------------------------------------------------------------- |
| **T2.1** | Add S3 bucket, SG, and key pair resources to same `.tf` file. Apply all at once.                        |
| **T2.2** | Remote state: create S3 bucket + DynamoDB table. Add backend config. `terraform init` to migrate state. |
| **T2.3** | Use Terraform module: call `terraform-aws-modules/vpc/aws`. Pass CIDR, AZs, subnets as input vars.      |

---

# T2.1 — Add Additional AWS Resources

Extend your Terraform configuration to include:

```text
EC2
S3 Bucket
Security Group
Key Pair
```

Conceptually:

```text
                  Terraform
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
         EC2         S3          SG
          │
          ▼
       Key Pair
```

Run:

```bash
terraform fmt
```

Then:

```bash
terraform validate
```

Generate the plan:

```bash
terraform plan
```

Apply:

```bash
terraform apply
```

> ⚠️ **S3 Naming:** S3 bucket names must be globally unique.

---

# T2.2 — Configure Remote State

Terraform state should generally not be managed as a local file when multiple engineers or automation systems need to work on the same infrastructure.

Use:

```text
S3 Bucket
+
State Locking
```

Example:

```hcl
backend "s3" {
  bucket         = "my-tf-state-bucket"
  key            = "devops/terraform.tfstate"
  region         = "ap-south-1"
  dynamodb_table = "tf-state-lock"
  encrypt        = true
}
```

Initialize:

```bash
terraform init
```

If Terraform detects that an existing local state needs migration, follow the migration prompt carefully.

Conceptually:

```text
Local State
terraform.tfstate
       │
       │ terraform init
       ▼
Remote Backend
       │
       ▼
S3
       │
       ▼
Shared Terraform State
```

> 🔐 **Security Note:** Terraform state can contain sensitive values. Encrypt remote state, restrict access through IAM, and never expose state files publicly.

---

# T2.3 — Use a Terraform VPC Module

Terraform Registry provides reusable modules.

Example:

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = "devops-vpc"

  cidr = "10.0.0.0/16"

  azs = [
    "ap-south-1a",
    "ap-south-1b"
  ]

  private_subnets = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]

  public_subnets = [
    "10.0.101.0/24",
    "10.0.102.0/24"
  ]

  enable_nat_gateway = true
  single_nat_gateway = true
}
```

Initialize:

```bash
terraform init
```

Then:

```bash
terraform plan
```

Apply:

```bash
terraform apply
```

> 💰 **Cost Note:** NAT Gateways can incur AWS charges. For a learning environment, remove unnecessary resources when the lab is complete.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                       |
| -------- | ------------------------------------------------------------------------------------------------ |
| **T3.1** | Full infrastructure: VPC + Subnets + IGW + SG + EC2 + EIP + ALB + ASG. All in Terraform.         |
| **T3.2** | `terraform workspace`: create dev and prod workspaces. Different instance sizes per workspace.   |
| **T3.3** | Terratest: write a Go test that validates EC2 is running and accessible after `terraform apply`. |

---

# T3.1 — Full AWS Infrastructure

Build the complete infrastructure:

```text
VPC
 │
 ├── Public Subnet
 │      │
 │      ├── Internet Gateway
 │      │
 │      └── ALB
 │
 ├── Public Subnet
 │      │
 │      └── ALB
 │
 ├── Security Group
 │
 ├── EC2
 │
 ├── Elastic IP
 │
 └── Auto Scaling Group
```

A production-style architecture could look like:

```text
                           Internet
                              │
                              ▼
                     Internet Gateway
                              │
                              ▼
                    ┌─────────────────┐
                    │       VPC       │
                    │  10.0.0.0/16    │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
             Public Subnet       Public Subnet
             ap-south-1a         ap-south-1b
                    │                 │
                    └────────┬────────┘
                             ▼
                         ALB
                             │
                             ▼
                      Target Group
                             │
                 ┌───────────┴───────────┐
                 ▼                       ▼
              EC2 / ASG               EC2 / ASG
                 │                       │
                 └───────────┬───────────┘
                             ▼
                       Application
```

Terraform resources may include:

```text
aws_vpc
aws_subnet
aws_internet_gateway
aws_route_table
aws_route_table_association
aws_security_group
aws_instance
aws_eip
aws_lb
aws_lb_target_group
aws_lb_listener
aws_autoscaling_group
aws_launch_template
```

> 💡 **Best Practice:** For a maintainable project, split networking, compute, security, and load-balancing resources into reusable modules rather than placing the entire infrastructure in one file.

---

# T3.2 — Terraform Workspaces

Terraform workspaces can be used to maintain separate state instances for different environments.

Create development workspace:

```bash
terraform workspace new dev
```

Create production workspace:

```bash
terraform workspace new prod
```

List workspaces:

```bash
terraform workspace list
```

Switch to development:

```bash
terraform workspace select dev
```

Switch to production:

```bash
terraform workspace select prod
```

Check current workspace:

```bash
terraform workspace show
```

---

## Different Instance Sizes by Workspace

Example:

```hcl
variable "instance_type" {
  default = "t2.micro"
}
```

You can use workspace-aware logic:

```hcl
locals {
  instance_type = terraform.workspace == "prod" ? "t3.medium" : "t2.micro"
}
```

Then:

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type

  tags = {
    Name        = "tf-web-${terraform.workspace}"
    Environment = terraform.workspace
    ManagedBy   = "terraform"
  }
}
```

Development:

```text
dev → t2.micro
```

Production:

```text
prod → t3.medium
```

> ⚠️ **Production Recommendation:** For larger organizations, separate AWS accounts and/or separate state configurations are often preferable to relying solely on Terraform workspaces for strong environment isolation.

---

# T3.3 — Terratest

**Terratest** is a Go-based testing framework commonly used to validate infrastructure.

Conceptually:

```text
Terraform
    │
    ▼
terraform apply
    │
    ▼
AWS Infrastructure
    │
    ▼
Terratest
    │
    ├── Is EC2 running?
    ├── Is network reachable?
    ├── Does application respond?
    └── Are expected resources present?
```

A test might validate:

```text
EC2 state = running
Port 80 = accessible
HTTP response = 200
Expected application content = present
```

Typical workflow:

```bash
terraform init
terraform apply
go test ./tests -v
terraform destroy
```

> 🧪 **Best Practice:** Infrastructure tests should clean up resources even when a test fails. This helps prevent unnecessary AWS resource consumption.

---

# 🧹 Terraform Formatting

Run:

```bash
terraform fmt
```

This automatically formats Terraform files according to standard Terraform formatting conventions.

For a recursive project:

```bash
terraform fmt -recursive
```

---

# 🔍 Terraform Dependency Graph

Terraform automatically builds a dependency graph.

For example:

```text
VPC
 │
 ▼
Subnet
 │
 ▼
EC2
```

Terraform understands relationships between resources and determines the appropriate creation and destruction order.

You can visualize the graph:

```bash
terraform graph
```

---

# 🔐 Terraform State Management

Terraform state maps your configuration to actual infrastructure.

Conceptually:

```text
Terraform Configuration
        │
        ▼
   Desired State
        │
        ▼
 Terraform State
        │
        ▼
 Actual AWS Resources
```

State may contain:

* Resource IDs
* Resource attributes
* Dependencies
* Outputs
* Potentially sensitive values

Therefore:

> 🔐 **Never expose Terraform state publicly.**

---

# 🚨 Terraform Drift

**Drift** occurs when infrastructure is changed outside Terraform.

Example:

```text
Terraform
   │
   ▼
EC2 instance
```

An administrator manually changes the instance outside Terraform.

Now:

```text
Terraform Configuration
        ≠
AWS Infrastructure
```

Run:

```bash
terraform plan
```

Terraform detects differences between the declared configuration and infrastructure/state.

This is one of the major advantages of Infrastructure as Code.

---

# 🛡️ Terraform Security Best Practices

* Never commit Terraform state files to Git.
* Never hardcode AWS access keys.
* Use IAM roles where possible.
* Use remote state with encryption.
* Restrict S3 state-bucket access.
* Enable appropriate state locking.
* Review `terraform plan` before production changes.
* Avoid unnecessary `-auto-approve`.
* Store secrets in AWS Secrets Manager or Parameter Store.
* Use variables instead of hardcoded environment-specific values.
* Use `.gitignore`.
* Apply least-privilege IAM policies.
* Use separate state for production environments.
* Use CI/CD approval gates for production.
* Pin provider and module versions appropriately.
* Run security scanning tools such as Checkov or tfsec where appropriate.

---

# 📄 Recommended `.gitignore`

Create:

```text
.gitignore
```

Example:

```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
crash.*.log

# Variable files that may contain secrets
*.tfvars
*.tfvars.json

# Terraform override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI configuration
.terraformrc
terraform.rc

# OS files
.DS_Store
Thumbs.db
```

> 💡 **Note:** Whether `*.tfvars` should be ignored depends on the contents. Public, non-sensitive example variable files can be committed if your team explicitly permits it. Never commit files containing secrets.

---

# 🧪 Terraform Validation Workflow

Before applying infrastructure:

```bash
terraform fmt -check
```

Then:

```bash
terraform init
```

Then:

```bash
terraform validate
```

Then:

```bash
terraform plan
```

Finally:

```bash
terraform apply
```

Recommended workflow:

```text
              Developer
                  │
                  ▼
             terraform fmt
                  │
                  ▼
           terraform validate
                  │
                  ▼
            terraform plan
                  │
                  ▼
           Code Review / PR
                  │
                  ▼
          Approval / CI Checks
                  │
                  ▼
          terraform apply
                  │
                  ▼
            AWS Resources
```

---

# 🩺 Troubleshooting

## `terraform init` Fails

Check:

* AWS credentials or IAM role.
* AWS region.
* Provider version.
* S3 backend configuration.
* S3 bucket existence.
* Backend permissions.
* Network connectivity.

Run:

```bash
terraform init
```

---

## `terraform plan` Shows Unexpected Changes

Check:

```bash
terraform plan
```

Then inspect:

```bash
terraform show
```

Look for:

* Manual AWS changes
* Changed variables
* Changed provider versions
* Changed resource configuration
* Resource drift

---

## EC2 AMI Is Not Found

Verify the AMI filter:

```hcl
filter {
  name   = "name"
  values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
}
```

Also verify:

```text
AWS Region = ap-south-1
```

The Ubuntu owner ID and image naming conventions must match the image you intend to deploy.

---

## Terraform Wants to Destroy a Resource

Do not immediately apply.

Run:

```bash
terraform plan
```

Carefully examine the proposed change.

Look for:

```text
-/+
```

This indicates Terraform intends to destroy and recreate a resource.

Possible causes include:

* Immutable resource attributes changed.
* Resource moved or renamed.
* Provider behavior changed.
* Configuration changed.
* State drift.

---

# 📋 Lab 19 Completion Checklist

* [ ] Install Terraform.
* [ ] Configure AWS credentials or an appropriate IAM role.
* [ ] Create `main.tf`.
* [ ] Configure AWS provider.
* [ ] Configure Terraform version.
* [ ] Configure provider version.
* [ ] Configure AWS region.
* [ ] Create Ubuntu AMI data source.
* [ ] Create EC2 instance.
* [ ] Add EC2 tags.
* [ ] Create Terraform output.
* [ ] Run `terraform init`.
* [ ] Run `terraform validate`.
* [ ] Run `terraform plan`.
* [ ] Run `terraform apply`.
* [ ] Run `terraform show`.
* [ ] Run `terraform output`.
* [ ] Modify EC2 tags.
* [ ] Observe in-place update in `terraform plan`.
* [ ] Add S3 resource.
* [ ] Add Security Group.
* [ ] Add key pair resource.
* [ ] Configure remote state.
* [ ] Configure state locking using the approach appropriate to your Terraform version.
* [ ] Use the VPC community module.
* [ ] Create a complete VPC.
* [ ] Create subnets.
* [ ] Create Internet Gateway.
* [ ] Create Security Group.
* [ ] Create EC2.
* [ ] Create EIP.
* [ ] Create ALB.
* [ ] Create ASG.
* [ ] Create `dev` workspace.
* [ ] Create `prod` workspace.
* [ ] Configure different instance sizes.
* [ ] Write a Terratest test.
* [ ] Validate infrastructure.
* [ ] Run `terraform destroy` after completing the lab.

---

# 🧠 Lab 19 — Key Takeaways

| Terraform Concept | Purpose                                                  |
| ----------------- | -------------------------------------------------------- |
| **Provider**      | Connects Terraform to infrastructure APIs such as AWS    |
| **Resource**      | Defines infrastructure Terraform creates/manages         |
| **Data Source**   | Retrieves existing or dynamically discovered information |
| **Variable**      | Makes configuration reusable and configurable            |
| **Output**        | Exposes useful Terraform values                          |
| **State**         | Tracks infrastructure managed by Terraform               |
| **Backend**       | Defines where Terraform state is stored                  |
| **S3 Backend**    | Provides centralized remote state storage                |
| **State Locking** | Prevents conflicting state modifications                 |
| **Module**        | Packages reusable infrastructure configuration           |
| **Workspace**     | Provides separate Terraform state instances              |
| **Plan**          | Previews infrastructure changes                          |
| **Apply**         | Creates or modifies infrastructure                       |
| **Destroy**       | Removes Terraform-managed infrastructure                 |
| **Terratest**     | Tests infrastructure using Go                            |
| **Drift**         | Difference between declared and actual infrastructure    |

---

# 🚀 Complete Terraform DevOps Architecture

```text
                         Git Repository
                              │
                              ▼
                       Terraform Code
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
             Variables                  Modules
                 │                         │
                 └────────────┬────────────┘
                              ▼
                       Terraform Plan
                              │
                              ▼
                         Code Review
                              │
                              ▼
                      Terraform Apply
                              │
                              ▼
                    ┌─────────────────┐
                    │       AWS       │
                    └────────┬────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
       ▼                     ▼                     ▼
      VPC                   EC2                   ALB
       │                     │                     │
       ├── Subnets           ├── EIP               └── Target Group
       ├── IGW               └── SG
       └── Routes
                              │
                              ▼
                       Auto Scaling Group
                              │
                              ▼
                        Applications

                              ▲
                              │
                        Terraform State
                              │
                              ▼
                        Amazon S3
                              │
                              ▼
                       State Locking
```

> **🎓 Lab Outcome:** By completing Lab 19, you should be able to define AWS infrastructure using Terraform, manage EC2 and networking resources, use dynamic AMI discovery, understand and protect Terraform state, implement remote state, consume reusable modules, separate environments with workspaces, provision complete AWS infrastructure, and validate deployments through automated infrastructure testing.
