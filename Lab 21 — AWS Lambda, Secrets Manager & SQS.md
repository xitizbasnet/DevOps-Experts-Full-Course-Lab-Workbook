# SESSION 13 — AWS: Secrets Manager, Lambda & SQS

## Lab 21 — AWS Lambda, Secrets Manager & SQS

> **Objective:** Build serverless functions, manage secrets securely, and connect services using an Amazon SQS queue.

---

## 🎯 Objective

In this lab, you will:

* Create and invoke an AWS Lambda function.
* Manage application secrets using **AWS Secrets Manager**.
* Grant Lambda permissions to retrieve secrets.
* Monitor Lambda execution through **Amazon CloudWatch Logs**.
* Create an **Amazon SQS** queue.
* Configure SQS as a Lambda trigger.
* Explore event-driven serverless architectures.
* Extend the solution using S3, EventBridge, API Gateway, SNS, Slack, ECR, and k3s.

---

# 🏗️ Architecture Overview

The core lab architecture is:

```text
                         ┌─────────────────────┐
                         │   AWS Secrets       │
                         │      Manager        │
                         │                     │
                         │ prod/app/db         │
                         └──────────┬──────────┘
                                    │
                                    │ Get Secret
                                    ▼
┌──────────────┐          ┌─────────────────────┐
│   SQS Queue  │─────────▶│   AWS Lambda       │
│              │ Trigger  │   devops-hello     │
│ devops-queue │          │   Python 3.11      │
└──────────────┘          └──────────┬──────────┘
                                    │
                                    ▼
                          ┌─────────────────────┐
                          │  CloudWatch Logs    │
                          │                     │
                          │ Lambda execution    │
                          │ and errors          │
                          └─────────────────────┘
```

### 🔄 Core Workflow

```text
Message
   │
   ▼
Amazon SQS
   │
   │ Lambda Trigger
   ▼
AWS Lambda
   │
   ├──▶ AWS Secrets Manager
   │       │
   │       └──▶ Retrieve application secret
   │
   └──▶ Amazon CloudWatch Logs
```

---

# ⚙️ Lambda Setup

## 1. Create the Lambda Function

Navigate to:

**AWS Console → Lambda → Create Function → Author from Scratch**

Configure the function:

```text
Name: devops-hello
Runtime: Python 3.11
Architecture: x86_64
```

For the execution role:

```text
Role: Create new
Permission: Basic Lambda execution role
```

---

## 2. Lambda Function Code

Use the following function code:

```python
import json
import boto3
import os

def lambda_handler(event, context):
    # Get secret from Secrets Manager
    client = boto3.client(
        'secretsmanager',
        region_name='ap-south-1'
    )

    secret = client.get_secret_value(
        SecretId='prod/app/db'
    )

    secret_data = json.loads(
        secret['SecretString']
    )

    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Hello from Lambda!',
            'db_host': secret_data.get('host', 'found'),
            'event': event
        })
    }
```

> 🔐 **Security Note:** The function retrieves the secret at runtime rather than hard-coding the sensitive value directly into the Lambda source code.

---

# 🔐 AWS Secrets Manager

## 3. Create the Secret

Navigate to:

**AWS Console → Secrets Manager → Store a new secret**

Select:

```text
Secret type: Other
```

Create the following key/value pairs:

```text
host     = mydb.example.com
password = Secret123
```

Name the secret:

```text
prod/app/db
```

### Secret Structure

```text
Secret Name: prod/app/db

host:
    mydb.example.com

password:
    Secret123
```

> ⚠️ **Security Best Practice:** The supplied `Secret123` value is suitable as a lab example only. Do not use it as a production credential.

---

# 🔑 Grant Lambda Access to Secrets Manager

The Lambda execution role must have permission to retrieve the secret.

Add the appropriate **Secrets Manager permission** to the Lambda execution role.

The Lambda function requires permission to call:

```text
secretsmanager:GetSecretValue
```

> 🔐 **Best Practice:** Follow the principle of least privilege. Grant the Lambda role access only to the specific secret(s) it actually needs.

---

# 🧪 Test the Lambda Function

## 4. Create a Test Event

In the Lambda console:

**Test → Create test event**

Create an appropriate test event and invoke the function.

Verify:

* Lambda execution succeeds.
* Response contains the expected message.
* Secret retrieval works.
* Event information is returned.
* No unexpected errors appear.

---

# 📊 CloudWatch Monitoring

Check Lambda logs in:

**AWS Console → CloudWatch → Logs → Log groups**

Locate the Lambda function's log group and inspect the invocation.

You should be able to review:

* Invocation details
* Execution status
* Errors
* Execution duration
* Log output

---

# 📬 Amazon SQS

## 5. Create the SQS Queue

Navigate to:

**AWS Console → SQS → Create Queue**

Configure:

```text
Type: Standard
Name: devops-queue
```

Create the queue.

---

## 6. Configure SQS as a Lambda Trigger

Add an SQS trigger to the Lambda function.

Configure:

```text
SQS Queue:
devops-queue
```

The resulting architecture is:

```text
Producer
   │
   ▼
┌───────────────────┐
│ Amazon SQS        │
│ devops-queue      │
└─────────┬─────────┘
          │
          │ Event Source
          ▼
┌───────────────────┐
│ AWS Lambda        │
│ devops-hello      │
└─────────┬─────────┘
          │
          ▼
   CloudWatch Logs
```

Send a test message to the queue.

The Lambda function should automatically process the message.

---

# 🧪 TASK SET 1 — Guided

| Task     | What to do                                                                                    |
| -------- | --------------------------------------------------------------------------------------------- |
| **T1.1** | Create Lambda function. Invoke manually with test event. View response and CloudWatch logs.   |
| **T1.2** | Create secret in Secrets Manager. Grant Lambda access. Lambda reads and returns secret value. |
| **T1.3** | Create SQS queue. Set as Lambda trigger. Send message via Console → Lambda auto-processes.    |

---

## T1.1 — Create and Test Lambda

Create:

```text
devops-hello
```

Use:

```text
Python 3.11
x86_64
```

Invoke the function manually with a test event.

Verify:

* Response
* Execution status
* CloudWatch logs
* Lambda duration

---

## T1.2 — Integrate Secrets Manager

Create:

```text
prod/app/db
```

with:

```text
host     = mydb.example.com
password = Secret123
```

Grant the Lambda execution role permission to retrieve the secret.

Invoke Lambda and verify that the function can retrieve the required secret information.

---

## T1.3 — Integrate SQS

Create:

```text
devops-queue
```

Configure it as the Lambda trigger.

Send a message using the AWS Console.

Verify:

```text
SQS Message
    │
    ▼
Lambda Invocation
    │
    ▼
CloudWatch Logs
```

---

# 🔧 TASK SET 2 — Practice

| Task     | What to do                                                                                    |
| -------- | --------------------------------------------------------------------------------------------- |
| **T2.1** | S3-triggered Lambda: upload file to S3 → Lambda resizes image and saves to another S3 bucket. |
| **T2.2** | Scheduled Lambda: EventBridge rule triggers Lambda every 5 min to check EC2 health.           |
| **T2.3** | Implement Lambda with API Gateway: create REST API that invokes Lambda and returns JSON.      |

---

## T2.1 — S3-Triggered Lambda

Create an event-driven workflow:

```text
S3 Upload
   │
   ▼
Lambda
   │
   │ Resize image
   ▼
Destination S3 Bucket
```

Upload an image to the source S3 bucket.

The Lambda function should:

1. Receive the S3 event.
2. Retrieve the uploaded object.
3. Resize the image.
4. Save the processed image to another S3 bucket.

Verify that the resized image is available in the destination bucket.

---

## T2.2 — Scheduled Lambda with EventBridge

Create an **Amazon EventBridge** rule that triggers Lambda every 5 minutes.

Architecture:

```text
EventBridge
     │
     │ Every 5 minutes
     ▼
AWS Lambda
     │
     ▼
Check EC2 Health
```

The Lambda function should check the health/status of EC2 instances.

---

## T2.3 — Lambda with API Gateway

Create a REST API using **Amazon API Gateway**.

Configure:

```text
Client
  │
  ▼
API Gateway
  │
  ▼
Lambda
  │
  ▼
JSON Response
```

Test the API endpoint and verify that it invokes Lambda and returns JSON.

---

# 🚀 TASK SET 3 — Challenge

| Task     | What to do                                                                                |
| -------- | ----------------------------------------------------------------------------------------- |
| **T3.1** | Serverless CI/CD notification: CodePipeline failure → SNS → SQS → Lambda → Slack webhook. |
| **T3.2** | Lambda container image: package Lambda as Docker image, push to ECR, deploy from ECR.     |
| **T3.3** | k3s cluster on EC2: install k3s single-node. Deploy same app in k3s. Compare to EKS.      |

---

## T3.1 — Serverless CI/CD Notification

Build the following event-driven notification pipeline:

```text
CodePipeline Failure
        │
        ▼
       SNS
        │
        ▼
       SQS
        │
        ▼
      Lambda
        │
        ▼
 Slack Webhook
```

### Expected Workflow

1. CodePipeline experiences a failure.
2. SNS publishes a notification.
3. SQS receives the notification.
4. Lambda is triggered by SQS.
5. Lambda processes the event.
6. Lambda sends a notification to Slack using a webhook.

> 💡 **Challenge:** Include useful information in the Slack notification such as pipeline name, execution status, failure details, and timestamp.

---

## T3.2 — Deploy Lambda Using a Container Image

Package the Lambda application as a Docker image.

Expected workflow:

```text
Lambda Application
       │
       ▼
 Dockerfile
       │
       ▼
 Docker Image
       │
       ▼
 Amazon ECR
       │
       ▼
 AWS Lambda
```

Complete the following:

1. Create the Lambda Docker image.
2. Build the image.
3. Authenticate Docker with Amazon ECR.
4. Push the image to ECR.
5. Create or update Lambda using the ECR image.
6. Invoke the Lambda function.
7. Verify execution through CloudWatch.

---

## T3.3 — Deploy the Application to k3s

Create a **single-node k3s cluster on EC2**.

Deploy the same application to k3s.

Compare the deployment with EKS.

### Comparison Areas

Evaluate:

* Installation
* Cluster management
* Control plane
* Networking
* Storage
* Application deployment
* Scaling
* Monitoring
* Security
* Operational overhead
* AWS integration

Architecture:

```text
                    AWS EC2
                       │
                       ▼
                  k3s Cluster
                       │
                       ▼
                 Application
```

Compare this with:

```text
                    AWS
                     │
                     ▼
                    EKS
                     │
                     ▼
                Kubernetes
                     │
                     ▼
                Application
```

---

# 🔐 Security & Best Practices

### Secrets Manager

* [ ] Do not hard-code production passwords in source code.
* [ ] Store sensitive values in Secrets Manager.
* [ ] Use IAM policies to control secret access.
* [ ] Apply least-privilege permissions.
* [ ] Consider secret rotation for production credentials.

### Lambda

* [ ] Use an execution role with only required permissions.
* [ ] Monitor Lambda execution using CloudWatch.
* [ ] Handle errors and retries appropriately.
* [ ] Avoid logging sensitive information.

### SQS

* [ ] Configure appropriate visibility timeout.
* [ ] Consider a Dead-Letter Queue (DLQ) for failed processing.
* [ ] Monitor queue depth and message failures.
* [ ] Use appropriate message retention settings.

### Event-Driven Architecture

* [ ] Keep producers and consumers loosely coupled.
* [ ] Design Lambda functions to handle repeated event processing safely.
* [ ] Monitor failures across the entire event chain.

---

# ✅ Lab 21 — Completion Checklist

### AWS Lambda

* [ ] Lambda function `devops-hello` created
* [ ] Python 3.11 runtime configured
* [ ] x86_64 architecture configured
* [ ] Lambda execution role created
* [ ] Function invoked successfully
* [ ] CloudWatch logs reviewed

### Secrets Manager

* [ ] Secret `prod/app/db` created
* [ ] `host` key configured
* [ ] `password` key configured
* [ ] Lambda execution role granted Secrets Manager access
* [ ] Lambda successfully retrieved the secret

### SQS

* [ ] Standard SQS queue `devops-queue` created
* [ ] SQS configured as Lambda trigger
* [ ] Test message sent
* [ ] Lambda automatically processed the message

### Practice

* [ ] S3-triggered Lambda implemented
* [ ] EventBridge scheduled Lambda implemented
* [ ] API Gateway + Lambda implemented

### Challenge

* [ ] CodePipeline → SNS → SQS → Lambda → Slack workflow implemented
* [ ] Lambda container image built
* [ ] Image pushed to ECR
* [ ] Lambda deployed from ECR
* [ ] Single-node k3s installed on EC2
* [ ] Application deployed to k3s
* [ ] k3s compared with EKS

---

# 💡 Key Takeaways

After completing **Lab 21**, you should understand how to:

1. Build and invoke AWS Lambda functions.
2. Secure application credentials with **AWS Secrets Manager**.
3. Use IAM roles to provide Lambda with controlled AWS access.
4. Process asynchronous workloads with **Amazon SQS**.
5. Build event-driven architectures using S3, EventBridge, SNS, and SQS.
6. Expose Lambda functions through **API Gateway**.
7. Monitor serverless applications using **CloudWatch**.
8. Package Lambda functions as container images and deploy them from **Amazon ECR**.
9. Build CI/CD notification workflows using AWS serverless services.
10. Deploy applications to **k3s** and compare the experience with **Amazon EKS**.

> **🏁 Lab Outcome:** You have progressed from a basic Lambda function to event-driven, secure, and containerized serverless architectures using AWS Lambda, Secrets Manager, SQS, SNS, EventBridge, API Gateway, ECR, and k3s.
