# Lab 18 — Application Load Balancer (ALB)

> **🎯 Objective:** Create an AWS Application Load Balancer (ALB) that distributes HTTP traffic across multiple EC2 instances and learn how to implement health checks, path-based routing, HTTPS, access logging, weighted traffic shifting, AWS WAF, and Lambda targets.

---

## 📘 Lab Overview

In this lab, you will build a highly available web application architecture using:

* **Amazon EC2** — Backend web servers
* **Application Load Balancer (ALB)** — Distributes incoming HTTP/HTTPS traffic
* **Target Groups** — Define backend targets
* **Health Checks** — Detect unhealthy EC2 instances
* **AWS Certificate Manager (ACM)** — TLS/SSL certificates
* **Amazon S3** — ALB access-log storage
* **Amazon Athena** — Log analysis
* **AWS WAF** — Web application protection
* **AWS Lambda** — Serverless ALB target

---

# 🏗️ Architecture

```text
                         Internet
                            │
                            │ HTTP / HTTPS
                            ▼
                 ┌─────────────────────┐
                 │  Application Load   │
                 │     Balancer        │
                 │     devops-alb      │
                 └──────────┬──────────┘
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
          Target Group            Target Group
                 │                     │
                 ▼                     ▼
        ┌────────────────┐     ┌────────────────┐
        │    EC2 #1      │     │    EC2 #2      │
        │    nginx       │     │    nginx       │
        │ ap-south-1a    │     │ ap-south-1b    │
        └────────────────┘     └────────────────┘
```

---

# 🎯 Learning Objectives

By completing this lab, you will learn how to:

* Launch multiple EC2 web servers.
* Install nginx using EC2 User Data.
* Create an Application Load Balancer.
* Configure an Internet-facing ALB.
* Configure multiple Availability Zones.
* Create a Target Group.
* Register EC2 instances as ALB targets.
* Configure ALB health checks.
* Test traffic distribution.
* Understand ALB failover behavior.
* Configure path-based routing.
* Configure HTTPS using ACM.
* Redirect HTTP traffic to HTTPS.
* Enable ALB access logging.
* Analyze ALB logs using Athena.
* Implement weighted blue-green traffic shifting.
* Add AWS WAF to an ALB.
* Configure Lambda as an ALB target.

---

# 🖥️ Prerequisites

Before beginning, ensure you have:

* AWS account
* Access to the AWS Console
* AWS Region: `ap-south-1` (Mumbai)
* VPC with appropriate subnets in at least two Availability Zones
* EC2 permissions
* Elastic Load Balancing permissions
* IAM permissions
* S3 permissions for access logs
* ACM permissions for certificates
* WAF permissions for the challenge
* Lambda permissions for the challenge

---

# 🟢 Task Set 1 — Guided

## T1.1 — Launch Two EC2 Instances and Create an ALB

### Step 1 — Launch EC2 Instance #1

Launch an Ubuntu EC2 instance.

Use User Data to automatically install nginx:

```bash
#!/bin/bash

apt update -y
apt install -y nginx

systemctl enable nginx
systemctl start nginx
```

---

### Step 2 — Launch EC2 Instance #2

Repeat the process for a second EC2 instance.

Use the same User Data:

```bash
#!/bin/bash

apt update -y
apt install -y nginx

systemctl enable nginx
systemctl start nginx
```

---

## 🧪 Make Each EC2 Identify Itself

To make traffic distribution visible, customize each nginx default page.

On EC2 #1:

```bash
echo "Hello from EC2-1: $(hostname)" | sudo tee /var/www/html/index.html
```

On EC2 #2:

```bash
echo "Hello from EC2-2: $(hostname)" | sudo tee /var/www/html/index.html
```

Test locally on each instance:

```bash
curl http://localhost
```

You should see different hostnames.

Example:

```text
Hello from EC2-1: ip-10-0-1-25
```

and:

```text
Hello from EC2-2: ip-10-0-2-31
```

---

# ⚖️ Create the Application Load Balancer

Navigate to:

```text
AWS Console
→ EC2
→ Load Balancers
→ Create Load Balancer
```

Select:

```text
Application Load Balancer
```

Configure:

```text
Name: devops-alb
Scheme: Internet-facing
IP address type: IPv4
```

---

## 🌐 Network Mapping

Select your VPC.

Select two Availability Zones:

```text
ap-south-1a
ap-south-1b
```

Ensure that the selected subnets are suitable for an Internet-facing ALB and have the required routing.

---

# 🛡️ ALB Security Group

Create or select a Security Group for the ALB.

Allow:

```text
HTTP
TCP
Port: 80
Source: 0.0.0.0/0
```

Conceptually:

```text
Internet
   │
   │ TCP/80
   ▼
ALB Security Group
   │
   ▼
Application Load Balancer
```

> 🔐 **Security Note:** Public HTTP access is appropriate for this introductory lab. In production, prefer HTTPS and redirect HTTP traffic to HTTPS.

---

# 🎯 Create the Target Group

Configure the listener:

```text
Protocol: HTTP
Port: 80
Action: Forward
Target Group: New Target Group
```

Create the Target Group:

```text
Target type: Instances
Protocol: HTTP
Port: 80
```

Register:

```text
EC2 #1
EC2 #2
```

The ALB will use the Target Group to determine where incoming traffic should be forwarded.

---

# ❤️ ALB Health Checks

The ALB periodically checks whether registered targets are healthy.

A typical health check is:

```text
Protocol: HTTP
Port: Traffic port
Path: /
```

Expected response:

```text
HTTP 200 OK
```

Target states should eventually become:

```text
healthy
healthy
```

Architecture:

```text
                 ALB
                  │
          ┌───────┴───────┐
          │               │
       Health           Health
       Check             Check
          │               │
          ▼               ▼
       EC2 #1           EC2 #2
       Healthy          Healthy
```

---

# 🚀 Create the ALB

Select:

```text
Create Load Balancer
```

Wait until the ALB state becomes:

```text
Active
```

Copy the generated DNS name.

Example:

```text
devops-alb-123456.ap-south-1.elb.amazonaws.com
```

Open:

```text
http://devops-alb-123456.ap-south-1.elb.amazonaws.com
```

You should receive a response from one of your EC2 instances.

---

# ⚠️ Important — EC2 Security Group Configuration

The EC2 instances should not generally allow HTTP traffic from the entire Internet when they are behind an ALB.

A better production architecture is:

```text
Internet
   │
   ▼
ALB Security Group
   │
   │ HTTP/HTTPS
   ▼
EC2 Security Group
```

The EC2 Security Group should allow HTTP/HTTPS **from the ALB Security Group**, rather than:

```text
0.0.0.0/0
```

This prevents users from bypassing the ALB and connecting directly to backend instances.

---

# 🧪 Test ALB from CLI

Set the ALB DNS name:

```bash
ALB_DNS="devops-alb-123456.ap-south-1.elb.amazonaws.com"
```

Run:

```bash
for i in {1..10}; do
  curl -s http://$ALB_DNS
done
```

To specifically look for the hostname:

```bash
for i in {1..10}; do
  curl -s http://$ALB_DNS | grep hostname
done
```

You should observe responses from different EC2 instances.

> 💡 **Note:** Do not rely on strict alternating behavior. ALB distribution is not guaranteed to alternate requests one-for-one between targets.

---

# T1.2 — Test Traffic Distribution

Run:

```bash
for i in {1..10}; do
  echo "Request $i:"
  curl -s http://$ALB_DNS
  echo
done
```

Expected concept:

```text
Request 1 → EC2-1
Request 2 → EC2-2
Request 3 → EC2-2
Request 4 → EC2-1
Request 5 → EC2-2
...
```

The exact sequence may vary.

---

# T1.3 — Test ALB Health Checks and Failover

Stop one EC2 instance.

Navigate to:

```text
EC2
→ Instances
→ Select EC2
→ Instance state
→ Stop instance
```

Initially, the ALB may still show the target as healthy.

After health checks fail, the target should transition to:

```text
unhealthy
```

The ALB should stop routing new requests to that target.

Architecture after failure:

```text
                   ALB
                    │
             Target Group
                    │
            ┌───────┴───────┐
            │               │
            ▼               ▼
         EC2 #1           EC2 #2
         Healthy          Stopped
            │
            └──► Traffic
```

> 🩺 **Key Concept:** ALB health checks allow the load balancer to automatically avoid unhealthy targets.

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                          |
| -------- | ----------------------------------------------------------------------------------- |
| **T2.1** | Add path-based routing: `/api` → TG-1 (Flask), `/` → TG-2 (nginx). Test both paths. |
| **T2.2** | Configure HTTPS listener with ACM certificate (free). Redirect HTTP → HTTPS.        |
| **T2.3** | Enable ALB access logs → S3 bucket. Analyse request patterns with Athena.           |

---

# T2.1 — Path-Based Routing

Create two Target Groups:

```text
TG-1 → Flask application
TG-2 → nginx
```

Configure ALB listener rules:

```text
IF path = /api/*
        │
        ▼
      TG-1
      Flask

IF path = /*
        │
        ▼
      TG-2
      nginx
```

Architecture:

```text
                         ALB
                          │
                HTTP Listener :80
                          │
              ┌───────────┴───────────┐
              │                       │
          /api/*                    /*
              │                       │
              ▼                       ▼
          TG-1 Flask              TG-2 nginx
```

Test:

```bash
curl http://$ALB_DNS/api/
```

and:

```bash
curl http://$ALB_DNS/
```

Verify that each request reaches the expected application.

---

# T2.2 — Configure HTTPS with ACM

Navigate to:

```text
AWS Console
→ Certificate Manager
→ Request
→ Request a public certificate
```

Request a certificate for a domain you control.

Example:

```text
devops.example.com
```

Complete domain validation.

> 🔐 **Important:** ACM public certificates are available at no additional charge when used with supported AWS services such as an Application Load Balancer. You still need to own/control the domain and complete validation.

---

## Add HTTPS Listener

Navigate to:

```text
EC2
→ Load Balancers
→ devops-alb
→ Listeners
→ Add listener
```

Configure:

```text
Protocol: HTTPS
Port: 443
Certificate: ACM certificate
Default action: Forward to Target Group
```

---

## Redirect HTTP → HTTPS

Configure the HTTP listener:

```text
HTTP :80
```

Action:

```text
Redirect to HTTPS
Port: 443
Status code: 301
```

Architecture:

```text
Client
  │
  ├── HTTP :80
  │       │
  │       ▼
  │    Redirect
  │       │
  │       ▼
  └── HTTPS :443
          │
          ▼
         ALB
          │
          ▼
      Target Group
          │
          ▼
          EC2
```

---

# T2.3 — ALB Access Logs

Enable ALB access logging.

Navigate to:

```text
EC2
→ Load Balancers
→ devops-alb
→ Attributes
→ Access logs
```

Configure an S3 bucket for log storage.

Example:

```text
S3 Bucket:
devops-alb-access-logs-<unique-name>
```

Generate traffic:

```bash
for i in {1..20}; do
  curl -s http://$ALB_DNS > /dev/null
done
```

ALB access logs can contain information such as:

* Request time
* Client IP
* Target IP
* Request processing time
* Target processing time
* HTTP status codes
* Request URL
* User agent
* TLS information

---

# 🔎 Analyze Logs with Amazon Athena

Use Athena to query ALB access logs stored in S3.

Example analytical questions:

```text
Which URLs receive the most requests?
Which clients generate the most traffic?
Which requests return HTTP 4xx?
Which requests return HTTP 5xx?
Which targets have higher response times?
What is the distribution of HTTP status codes?
```

> 💡 **Best Practice:** For production environments, structure your S3 logging strategy carefully and use appropriate retention and lifecycle policies to control storage costs.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                 |
| -------- | ------------------------------------------------------------------------------------------ |
| **T3.1** | Blue-Green with ALB: weighted routing 90% blue / 10% green. Gradually shift to 100% green. |
| **T3.2** | Add WAF to ALB: block requests with SQL injection patterns.                                |
| **T3.3** | Lambda target in ALB: create a Lambda function and register as ALB target group.           |

---

# T3.1 — Blue-Green Deployment with ALB

Create two application environments:

```text
BLUE
Version 1
     │
     ▼
Target Group Blue

GREEN
Version 2
     │
     ▼
Target Group Green
```

Conceptually:

```text
                         ALB
                          │
                    Listener Rules
                          │
                ┌─────────┴─────────┐
                │                   │
              BLUE                GREEN
               90%                 10%
                │                   │
                ▼                   ▼
            TG-Blue             TG-Green
```

Begin with:

```text
Blue: 90%
Green: 10%
```

Monitor:

* HTTP 4xx
* HTTP 5xx
* Response latency
* Application logs
* Target health
* Business metrics

Gradually shift:

```text
90 / 10
   ↓
75 / 25
   ↓
50 / 50
   ↓
25 / 75
   ↓
0 / 100
```

Final state:

```text
Blue: 0%
Green: 100%
```

> ⚠️ **AWS Implementation Note:** Native ALB listener forwarding supports weighted target groups, but the exact behavior and configuration depend on listener rules and target groups. Do not assume the ALB provides a generic application-deployment controller; deployment automation should manage the traffic-shift process.

---

# T3.2 — AWS WAF with ALB

AWS WAF can protect the ALB against common web attacks.

Architecture:

```text
Internet
   │
   ▼
 AWS WAF
   │
   │ Allowed
   ▼
  ALB
   │
   ▼
Target Groups
   │
   ▼
 EC2
```

Navigate to:

```text
AWS WAF & Shield
→ Web ACLs
→ Create web ACL
```

Associate the Web ACL with:

```text
devops-alb
```

Add AWS Managed Rules, including the appropriate rule group for common threats such as SQL injection.

Test requests against the protected ALB.

> 🛡️ **Security Note:** Use managed rules carefully and monitor false positives before enabling aggressive blocking in production. Start in a controlled testing mode where appropriate.

---

# T3.3 — Lambda as an ALB Target

An Application Load Balancer can invoke Lambda functions through a Lambda target group.

Architecture:

```text
                    ALB
                     │
                     ▼
              Listener Rule
                     │
                     ▼
              Lambda Target
                     │
                     ▼
              Lambda Function
                     │
                     ▼
                 Response
```

Create a Lambda function.

Example response concept:

```json
{
  "statusCode": 200,
  "statusDescription": "200 OK",
  "isBase64Encoded": false,
  "headers": {
    "Content-Type": "text/plain"
  },
  "body": "Hello from Lambda behind ALB"
}
```

Create the target group:

```text
Target type: Lambda
```

Register the Lambda function.

Then create an ALB listener rule that forwards traffic to the Lambda target group.

Test:

```bash
curl http://$ALB_DNS/lambda
```

---

# 🧪 Troubleshooting

## ALB Shows Targets as Unhealthy

Check:

```bash
curl http://localhost
```

on the EC2 instance.

Then check nginx:

```bash
sudo systemctl status nginx
```

Check listening ports:

```bash
sudo ss -tuln | grep :80
```

Check the EC2 Security Group.

Ensure:

```text
ALB Security Group
       │
       ▼
EC2 Security Group
       │
       ▼
TCP/80
```

---

## ALB DNS Does Not Respond

Check:

* ALB state is `Active`.
* ALB has subnets in appropriate Availability Zones.
* ALB Security Group allows HTTP/HTTPS.
* Target Group contains registered targets.
* Targets are healthy.
* Route tables are correct.
* Internet Gateway is attached where required.
* DNS name is copied correctly.

---

## Only One EC2 Receives Traffic

Check:

```text
Target Group
→ Targets
```

Both instances should show:

```text
healthy
```

If one is unhealthy, investigate its:

* nginx service
* port `80`
* Security Group
* Network ACL
* health-check path
* application response

---

# 🔐 ALB Security Best Practices

* Prefer HTTPS for production traffic.
* Redirect HTTP to HTTPS.
* Use ACM for TLS certificates.
* Keep EC2 instances private where architecture permits.
* Allow backend traffic from the ALB Security Group rather than the Internet.
* Use AWS WAF for appropriate web application protection.
* Enable ALB access logs.
* Monitor HTTP 4xx and 5xx responses.
* Monitor target response latency.
* Configure appropriate health checks.
* Use multiple Availability Zones.
* Apply least-privilege IAM permissions.
* Use S3 lifecycle policies for old access logs.
* Avoid exposing unnecessary backend ports.
* Monitor ALB and target health using CloudWatch.

---

# 📊 Key ALB Metrics

Important CloudWatch metrics include:

| Metric                      | Purpose                             |
| --------------------------- | ----------------------------------- |
| `RequestCount`              | Number of requests processed        |
| `HTTPCode_ELB_4XX_Count`    | Client-side errors generated by ALB |
| `HTTPCode_ELB_5XX_Count`    | ALB-side errors                     |
| `HTTPCode_Target_4XX_Count` | 4xx responses from targets          |
| `HTTPCode_Target_5XX_Count` | 5xx responses from targets          |
| `TargetResponseTime`        | Target response latency             |
| `HealthyHostCount`          | Number of healthy targets           |
| `UnHealthyHostCount`        | Number of unhealthy targets         |
| `NewConnectionCount`        | New connections established         |
| `ActiveConnectionCount`     | Active connections                  |

---

# 📋 Lab 18 Completion Checklist

* [ ] Launch EC2 #1.
* [ ] Launch EC2 #2.
* [ ] Install nginx on both instances.
* [ ] Customize each nginx page with its hostname.
* [ ] Create `devops-alb`.
* [ ] Configure Internet-facing IPv4.
* [ ] Select `ap-south-1a` and `ap-south-1b`.
* [ ] Configure ALB Security Group.
* [ ] Create Target Group.
* [ ] Register both EC2 instances.
* [ ] Configure HTTP listener on port `80`.
* [ ] Verify both targets become healthy.
* [ ] Test ALB DNS.
* [ ] Send multiple requests through the ALB.
* [ ] Observe traffic distribution.
* [ ] Stop one EC2 instance.
* [ ] Verify health-check failure.
* [ ] Verify traffic is routed only to healthy targets.
* [ ] Configure path-based routing.
* [ ] Create Flask and nginx Target Groups.
* [ ] Configure ACM certificate.
* [ ] Configure HTTPS on port `443`.
* [ ] Redirect HTTP to HTTPS.
* [ ] Enable ALB access logs.
* [ ] Store logs in S3.
* [ ] Analyze ALB logs with Athena.
* [ ] Implement blue-green traffic shifting.
* [ ] Configure AWS WAF.
* [ ] Test WAF protection.
* [ ] Configure Lambda as an ALB target.

---

# 🧠 Lab 18 — Key Takeaways

| Component                     | Purpose                                              |
| ----------------------------- | ---------------------------------------------------- |
| **Application Load Balancer** | Distributes HTTP/HTTPS traffic                       |
| **Target Group**              | Defines backend destinations                         |
| **Health Check**              | Determines target availability                       |
| **Listener**                  | Accepts incoming connections                         |
| **Listener Rule**             | Controls traffic-routing decisions                   |
| **Path-Based Routing**        | Routes different URL paths to different applications |
| **ACM**                       | Provides managed TLS certificates                    |
| **S3 Access Logs**            | Stores ALB request logs                              |
| **Athena**                    | Queries ALB logs in S3                               |
| **AWS WAF**                   | Protects web applications from common attacks        |
| **Weighted Target Groups**    | Supports controlled traffic distribution             |
| **Lambda Target Group**       | Allows ALB to invoke Lambda functions                |
| **Availability Zones**        | Improve application availability and resilience      |

---

# 🚀 End-to-End ALB Architecture

```text
                             INTERNET
                                │
                                ▼
                         ┌─────────────┐
                         │  AWS WAF    │
                         └──────┬──────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │ Application Load    │
                    │ Balancer            │
                    │ devops-alb           │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
             /api/*            /          /lambda
                │              │              │
                ▼              ▼              ▼
            Flask TG        nginx TG      Lambda TG
                │              │              │
         ┌──────┴──────┐       │              │
         ▼             ▼       ▼              ▼
       EC2-1         EC2-2   EC2 instances   Lambda
         │             │
         └──────┬──────┘
                │
                ▼
        Health Checks
                │
                ▼
        CloudWatch Metrics
                │
                ▼
          S3 Access Logs
                │
                ▼
             Athena
```

> **🎓 Lab Outcome:** By completing Lab 18, you should be able to deploy an Application Load Balancer across multiple Availability Zones, register EC2 targets, validate health checks and failover, implement path-based routing and HTTPS, analyze ALB access logs, perform controlled blue-green traffic shifts, protect applications with AWS WAF, and integrate Lambda functions as ALB targets.
