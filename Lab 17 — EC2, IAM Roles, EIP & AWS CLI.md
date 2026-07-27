# Session 10 — AWS: EC2, IAM, EIP & Load Balancers

> **🎯 Objective:** Launch and manage Amazon EC2 instances, assign IAM roles, configure Elastic IP addresses, and automate AWS infrastructure tasks using the AWS CLI.

---

## 📘 Lab 17 — EC2, IAM Roles, EIP & AWS CLI

This lab introduces essential AWS infrastructure skills used by DevOps engineers, including:

* **Amazon EC2** — Compute instances
* **AWS IAM** — Identity and access management
* **IAM Roles** — Secure permissions for AWS resources
* **Elastic IP (EIP)** — Persistent public IPv4 address
* **AWS CLI** — Command-line automation
* **EC2 Metadata Service** — Instance and role information
* **Launch Templates** — Reusable EC2 configurations
* **AMI** — Custom EC2 machine images
* **Auto Scaling Groups** — Automated instance scaling

---

# 🏗️ Lab Architecture

```text
                         AWS Account
                              │
                              ▼
                           IAM Role
                     EC2-DevOps-Role
                              │
             ┌────────────────┴────────────────┐
             │                                 │
             ▼                                 ▼
        Amazon EC2                       Amazon S3
             │
      ┌──────┴──────┐
      │             │
      ▼             ▼
 Security Group   Elastic IP
      │             │
      └──────┬──────┘
             ▼
        Ubuntu EC2
             │
             ▼
         AWS CLI
             │
       ┌─────┴─────┐
       ▼           ▼
    AWS APIs    Instance Metadata
```

---

# 🎯 Learning Objectives

By completing this lab, you will learn how to:

* Create an IAM role for EC2.
* Attach AWS managed policies to an IAM role.
* Create and use an EC2 key pair.
* Configure an EC2 security group.
* Launch an Ubuntu EC2 instance.
* Associate an Elastic IP with an instance.
* Connect to EC2 using SSH.
* Use AWS CLI from an EC2 instance.
* Verify IAM role credentials without storing access keys.
* Query EC2 instance metadata.
* Create Launch Templates.
* Create custom AMIs.
* Build Auto Scaling Groups.
* Use EC2 User Data for automation.
* Identify and stop non-production EC2 instances using AWS CLI.

---

# 🔐 IAM Setup

## 1. Create an IAM Role

Navigate to:

```text
AWS Console
→ IAM
→ Roles
→ Create Role
```

Select:

```text
Trusted entity type: AWS service
Use case: EC2
```

Attach the following policies:

```text
AmazonEC2ReadOnlyAccess
AmazonS3ReadOnlyAccess
```

Name the role:

```text
EC2-DevOps-Role
```

> 🔐 **Security Note:** The policies above are appropriate for this lab's learning objectives. In production, follow the principle of least privilege and create permissions specifically required by the workload.

---

# 🔑 Key Pair

Navigate to:

```text
EC2
→ Key Pairs
→ Create key pair
```

Create:

```text
devops-key.pem
```

On Linux/macOS, protect the private key:

```bash
chmod 400 devops-key.pem
```

> ⚠️ **Important:** Never commit `.pem` files to GitHub or another source-code repository.

---

# 🛡️ Security Group

Create a Security Group:

```text
devops-sg
```

Configure inbound access:

| Protocol | Port | Source      |
| -------- | ---: | ----------- |
| SSH      |   22 | My IP       |
| HTTP     |   80 | `0.0.0.0/0` |
| HTTPS    |  443 | `0.0.0.0/0` |

Conceptually:

```text
Internet
   │
   ├── HTTPS :443 ────────► EC2
   │
   ├── HTTP :80 ──────────► EC2
   │
   └── SSH :22 ───────────► EC2
          │
          └── Only from My IP
```

> 🛡️ **Best Practice:** Avoid exposing SSH (`22`) to `0.0.0.0/0`. Restrict administrative access to trusted IP addresses, VPNs, bastion hosts, or other approved access mechanisms.

---

# 🖥️ Launch EC2 Instance

Navigate to:

```text
EC2
→ Instances
→ Launch Instance
```

Use:

```text
AMI: Ubuntu 22.04
Instance type: t2.micro
Security Group: devops-sg
Key Pair: devops-key
IAM Role: EC2-DevOps-Role
```

After launch, wait until the instance reaches:

```text
Instance state: Running
Status checks: 2/2 checks passed
```

---

# 🌐 Elastic IP

An Elastic IP provides a persistent public IPv4 address for an EC2 instance.

Navigate to:

```text
EC2
→ Elastic IPs
→ Allocate Elastic IP address
```

Then:

```text
Actions
→ Associate Elastic IP address
→ Select EC2 instance
```

Connect using the EIP:

```bash
ssh -i devops-key.pem ubuntu@<EIP>
```

Example:

```bash
ssh -i devops-key.pem ubuntu@203.0.113.10
```

> 💡 **Important:** An Elastic IP remains associated with your AWS account and can persist across EC2 stop/start operations when it remains associated with the instance. AWS may charge for public IPv4 addresses, including Elastic IP usage, according to current AWS pricing.

---

# ☁️ AWS CLI on EC2

One of the key DevOps practices demonstrated in this lab is accessing AWS APIs **without storing long-lived access keys on the server**.

The EC2 instance uses its attached IAM role.

---

## Verify IAM Role

Run:

```bash
aws sts get-caller-identity
```

The output should identify the AWS identity associated with the instance role.

---

## List EC2 Instances

```bash
aws ec2 describe-instances --region ap-south-1
```

---

## List S3 Buckets

```bash
aws s3 ls
```

---

# 🔎 EC2 Instance Metadata

The EC2 Instance Metadata Service can provide information about the current instance.

## Instance ID

```bash
curl http://169.254.169.254/latest/meta-data/instance-id
```

## Public IPv4 Address

```bash
curl http://169.254.169.254/latest/meta-data/public-ipv4
```

## IAM Information

```bash
curl http://169.254.169.254/latest/meta-data/iam/info
```

> 🔐 **Security Note:** In production, prefer **IMDSv2** and avoid exposing instance metadata to untrusted applications or users. Where possible, configure the instance to require IMDSv2.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                           |
| -------- | ---------------------------------------------------------------------------------------------------- |
| **T1.1** | Create IAM Role for EC2. Launch EC2, attach role. Verify: `aws sts get-caller-identity` shows role.  |
| **T1.2** | Allocate Elastic IP. Associate to EC2. Stop and start instance — verify same IP.                     |
| **T1.3** | From EC2: `aws s3 ls` — lists your S3 buckets. Create bucket: `aws s3 mb s3://my-devops-bucket-2024` |

---

## T1.1 — IAM Role Verification

Create:

```text
EC2-DevOps-Role
```

Attach it to the EC2 instance.

SSH into the instance:

```bash
ssh -i devops-key.pem ubuntu@<EIP>
```

Verify the AWS identity:

```bash
aws sts get-caller-identity
```

The result should correspond to the IAM role attached to the EC2 instance.

### Why is this important?

Instead of storing:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

on the server, the EC2 instance receives temporary credentials through its IAM role.

```text
EC2
 │
 ▼
IAM Role
 │
 ▼
Temporary Credentials
 │
 ▼
AWS API
```

This is safer than embedding long-lived access keys into scripts or configuration files.

---

# T1.2 — Elastic IP Persistence

Allocate and associate the EIP.

Record the address:

```text
<EIP>
```

Connect:

```bash
ssh -i devops-key.pem ubuntu@<EIP>
```

Stop the instance:

```text
EC2
→ Instance
→ Instance state
→ Stop instance
```

Start it again.

Verify the Elastic IP:

```text
EC2
→ Elastic IPs
```

The associated EIP should remain the same while it remains associated with the instance.

---

# T1.3 — S3 Access from EC2

Run:

```bash
aws s3 ls
```

Create the specified bucket:

```bash
aws s3 mb s3://my-devops-bucket-2024
```

Verify:

```bash
aws s3 ls
```

> ⚠️ **AWS Naming Note:** S3 bucket names must be globally unique. If `my-devops-bucket-2024` is already taken, use another unique bucket name.

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                  |
| -------- | ------------------------------------------------------------------------------------------- |
| **T2.1** | Create Launch Template from your EC2 config. Launch 2 more instances from template.         |
| **T2.2** | Tag all EC2 instances: `Environment=Dev`, `Owner=Vinod`. Use CLI: `aws ec2 create-tags ...` |
| **T2.3** | Create AMI from your running EC2: `EC2 → Actions → Image → Create Image`. Launch from AMI.  |

---

# T2.1 — Create a Launch Template

A Launch Template provides a reusable EC2 configuration.

It can contain:

```text
AMI
Instance type
Key pair
Security group
IAM instance profile
Storage configuration
User Data
Tags
```

Navigate to:

```text
EC2
→ Launch Templates
→ Create launch template
```

Use your existing EC2 configuration.

Launch two additional instances using the template.

Verify:

```bash
aws ec2 describe-instances --region ap-south-1
```

---

# T2.2 — Tag EC2 Instances

Use:

```text
Environment=Dev
Owner=Vinod
```

Example AWS CLI:

```bash
aws ec2 create-tags \
  --resources <INSTANCE-ID> \
  --tags Key=Environment,Value=Dev Key=Owner,Value=Vinod \
  --region ap-south-1
```

Verify tags:

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --query 'Reservations[].Instances[].{ID:InstanceId,Tags:Tags}'
```

---

# T2.3 — Create an AMI

Navigate to:

```text
EC2
→ Instances
→ Select instance
→ Actions
→ Image and templates
→ Create image
```

Create an image from the running EC2 instance.

The resulting AMI can be used to launch additional instances with the same configured environment.

Conceptually:

```text
Configured EC2
      │
      ▼
     AMI
      │
 ┌────┴────┐
 ▼         ▼
EC2 #1    EC2 #2
```

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                               |
| -------- | -------------------------------------------------------------------------------------------------------- |
| **T3.1** | Auto Scaling Group: `min=2`, `max=5`, target CPU=`50%`. Simulate load with stress tool. Watch ASG scale. |
| **T3.2** | User Data: launch EC2 with startup script that auto-installs Docker and runs nginx container.            |
| **T3.3** | Cost optimization: identify all running EC2 by tag, stop non-production ones with AWS CLI script.        |

---

# T3.1 — Auto Scaling Group

Create an Auto Scaling Group with:

```text
Minimum capacity: 2
Maximum capacity: 5
Target CPU utilization: 50%
```

Architecture:

```text
                    Load Balancer
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
           EC2         EC2         EC2
             │           │           │
             └───────────┼───────────┘
                         │
                         ▼
                 Auto Scaling Group
                         │
                  CPU Target = 50%
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
         Scale Out              Scale In
          Max = 5                Min = 2
```

Install a stress-testing utility on an Ubuntu test instance if needed:

```bash
sudo apt update
sudo apt install -y stress
```

Generate CPU load:

```bash
stress --cpu 2 --timeout 300
```

Monitor the ASG:

```text
EC2
→ Auto Scaling Groups
→ Select ASG
→ Activity
```

Monitor:

* Desired capacity
* Current capacity
* CPU utilization
* Scaling activities
* Instance health

> 💡 **Best Practice:** For production workloads, configure health checks, cooldown behavior, scaling policies, and load balancing appropriately rather than relying solely on CPU utilization.

---

# T3.2 — EC2 User Data + Docker + Nginx

EC2 User Data allows commands to run during instance initialization.

Example:

```bash
#!/bin/bash

apt update -y
apt install -y docker.io

systemctl enable docker
systemctl start docker

docker run -d \
  --name nginx \
  -p 80:80 \
  nginx
```

Launch an Ubuntu EC2 instance with this script in:

```text
Advanced details
→ User data
```

After the instance starts, verify Docker:

```bash
docker ps
```

Expected container:

```text
nginx
```

Test:

```bash
curl http://localhost
```

Or open:

```text
http://<EC2-PUBLIC-IP>
```

The Security Group must permit HTTP traffic on port `80`.

---

# T3.3 — Cost Optimization with AWS CLI

First identify running instances by tag.

For example, list instances tagged:

```text
Environment=Dev
```

Use:

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --filters \
    "Name=instance-state-name,Values=running" \
    "Name=tag:Environment,Values=Dev" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Name:Tags[?Key==`Name`]|[0].Value,Environment:Tags[?Key==`Environment`]|[0].Value}'
```

Before stopping instances, review the list carefully.

Stop selected instances:

```bash
aws ec2 stop-instances \
  --instance-ids <INSTANCE-ID> \
  --region ap-south-1
```

For multiple instances:

```bash
aws ec2 stop-instances \
  --instance-ids <INSTANCE-ID-1> <INSTANCE-ID-2> \
  --region ap-south-1
```

> ⚠️ **Production Safety:** Do not automatically stop instances solely because they are tagged `Environment=Dev`. Confirm ownership, workload criticality, schedules, maintenance windows, and exceptions first.

---

# 🔧 Useful AWS CLI Commands

## List Running EC2 Instances

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --filters "Name=instance-state-name,Values=running"
```

## Get Instance IDs

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --query 'Reservations[].Instances[].InstanceId'
```

## Get Instance Public IPs

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --query 'Reservations[].Instances[].PublicIpAddress'
```

## Get Instance State

```bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name}'
```

## Stop an Instance

```bash
aws ec2 stop-instances \
  --instance-ids <INSTANCE-ID> \
  --region ap-south-1
```

## Start an Instance

```bash
aws ec2 start-instances \
  --instance-ids <INSTANCE-ID> \
  --region ap-south-1
```

## Check Current AWS Identity

```bash
aws sts get-caller-identity
```

---

# 🩺 Troubleshooting

## SSH Connection Fails

Check:

* EC2 instance is running.
* Security Group allows TCP `22`.
* Source IP is correct.
* Private key permissions are correct.
* Correct username is being used.
* Correct Elastic IP/public IP is being used.
* Network ACLs and routing are not blocking traffic.

Example:

```bash
chmod 400 devops-key.pem
```

Then:

```bash
ssh -i devops-key.pem ubuntu@<EIP>
```

---

## AWS CLI Returns `AccessDenied`

Check the attached IAM role:

```bash
aws sts get-caller-identity
```

Then review:

```text
IAM
→ Roles
→ EC2-DevOps-Role
→ Permissions
```

Confirm that the required action is permitted.

---

## EC2 Cannot Access S3

Verify:

1. IAM role is attached.
2. IAM policy permits the required S3 operation.
3. AWS CLI is using the expected region.
4. Network connectivity exists.
5. S3 bucket name is correct.

Run:

```bash
aws sts get-caller-identity
```

Then:

```bash
aws s3 ls
```

---

## Nginx Is Not Accessible

Check the container:

```bash
docker ps
```

Check Docker logs:

```bash
docker logs nginx
```

Check port:

```bash
sudo ss -tuln | grep :80
```

Check the Security Group allows:

```text
HTTP / TCP / 80
```

Test locally:

```bash
curl http://localhost
```

---

# 🔐 AWS Security Best Practices

* Use IAM roles instead of long-lived access keys on EC2.
* Follow least-privilege IAM policies.
* Restrict SSH access to trusted sources.
* Prefer Session Manager where appropriate.
* Require IMDSv2 for EC2 instances.
* Never commit private keys to Git.
* Never store AWS secrets in source code.
* Use Secrets Manager or Systems Manager Parameter Store for application secrets.
* Use encrypted EBS volumes.
* Apply security groups using least privilege.
* Tag resources consistently.
* Monitor CloudTrail activity.
* Use AWS Config and Security Hub where appropriate.
* Regularly review unused IAM permissions.
* Remove unused Elastic IPs and other unnecessary resources.
* Shut down non-production resources according to approved schedules.

---

# 🏷️ Recommended EC2 Tagging Standard

For this lab, use:

```text
Environment=Dev
Owner=Vinod
```

For a production DevOps environment, consider extending the tagging standard:

```text
Environment=Dev
Owner=Vinod
Application=DevOpsLab
Project=DevOpsTraining
CostCenter=IT
ManagedBy=Terraform
```

Consistent tagging makes it easier to:

* Identify resources.
* Track ownership.
* Automate operations.
* Implement cost controls.
* Filter AWS CLI queries.
* Build cost reports.
* Apply governance policies.

---

# 📋 Lab 17 Completion Checklist

* [ ] Create `EC2-DevOps-Role`.
* [ ] Attach EC2 and S3 read-only policies.
* [ ] Create `devops-key.pem`.
* [ ] Secure the key with `chmod 400`.
* [ ] Create `devops-sg`.
* [ ] Configure SSH, HTTP, and HTTPS inbound rules.
* [ ] Launch Ubuntu 22.04 EC2.
* [ ] Attach `EC2-DevOps-Role`.
* [ ] Allocate an Elastic IP.
* [ ] Associate the EIP with EC2.
* [ ] Connect through SSH.
* [ ] Install/verify AWS CLI.
* [ ] Run `aws sts get-caller-identity`.
* [ ] Run `aws ec2 describe-instances`.
* [ ] Run `aws s3 ls`.
* [ ] Query EC2 metadata.
* [ ] Create an S3 bucket.
* [ ] Create a Launch Template.
* [ ] Launch two additional instances.
* [ ] Apply EC2 tags.
* [ ] Create a custom AMI.
* [ ] Launch an instance from the AMI.
* [ ] Create an Auto Scaling Group.
* [ ] Configure minimum capacity of 2.
* [ ] Configure maximum capacity of 5.
* [ ] Configure target CPU utilization of 50%.
* [ ] Test scaling behavior.
* [ ] Deploy Docker automatically through User Data.
* [ ] Run nginx in Docker.
* [ ] Identify non-production instances by tags.
* [ ] Test stopping instances using AWS CLI.
* [ ] Review AWS security best practices.

---

# 🧠 Lab 17 — Key Takeaways

| Component / Concept    | Purpose                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| **EC2**                | Provides scalable virtual compute capacity                        |
| **IAM Role**           | Grants AWS permissions without storing long-lived access keys     |
| **Security Group**     | Controls network access to EC2                                    |
| **Key Pair**           | Provides SSH authentication                                       |
| **Elastic IP**         | Provides a persistent public IPv4 address                         |
| **AWS CLI**            | Enables command-line AWS management and automation                |
| **Instance Metadata**  | Provides information and credentials-related metadata to EC2      |
| **Launch Template**    | Defines reusable EC2 launch configuration                         |
| **AMI**                | Captures a reusable EC2 machine image                             |
| **Auto Scaling Group** | Automatically manages EC2 capacity                                |
| **User Data**          | Automates EC2 initialization                                      |
| **Tags**               | Provide resource organization, ownership, and automation metadata |

---

# 🚀 End-to-End DevOps Workflow

```text
                    AWS ACCOUNT
                         │
                         ▼
                    IAM ROLE
              EC2-DevOps-Role
                         │
                         ▼
                  LAUNCH TEMPLATE
                         │
                         ▼
                    EC2 INSTANCE
                    /          \
                   /            \
                  ▼              ▼
            Security Group    Elastic IP
                  │              │
                  └──────┬───────┘
                         ▼
                    SSH / CLI
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
             EC2        S3       AWS APIs
              │
              ▼
         Docker / Apps
              │
              ▼
       Auto Scaling Group
              │
         ┌────┴────┐
         ▼         ▼
       Scale     Scale
        Out       In
```

> **🎓 Lab Outcome:** By completing Lab 17, you should be able to provision and manage EC2 infrastructure, securely assign IAM permissions through instance roles, configure networking and Elastic IPs, automate AWS operations using the CLI, create reusable EC2 configurations and AMIs, implement Auto Scaling, automate instance initialization with User Data, and apply practical AWS security and cost-optimization practices.
