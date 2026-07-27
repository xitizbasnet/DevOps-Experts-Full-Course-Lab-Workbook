# Lab 05 — Docker Images & Dockerfile

## 🎯 Objective

Build custom Docker images using a **Dockerfile**, understand **layer caching**, and push Docker images to a **container registry**.

By the end of this lab, you will be able to:

* Create a Dockerfile for a Python Flask application.
* Build custom Docker images.
* Understand Docker image layers and build caching.
* Optimize Docker builds using layer ordering.
* Use `.dockerignore`.
* Tag and push images to Docker Hub.
* Scan images for security vulnerabilities.
* Push images to **AWS Elastic Container Registry (ECR)**.

---

# 🐳 Dockerfile Example

The following example demonstrates a Dockerfile for a **Python Flask application**.

```dockerfile
# Dockerfile — Python Flask App

FROM python:3.11-slim

# Metadata
LABEL maintainer="vinod@devops.com"
LABEL version="1.0"

# Working directory
WORKDIR /app

# Copy deps first (cache optimization)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app code
COPY . .

# Non-root user
RUN useradd -m appuser && chown -R appuser /app
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:5000/ || exit 1

# Start command
CMD ["python", "app.py"]
```

---

## 🔍 Dockerfile Breakdown

### Base Image

```dockerfile
FROM python:3.11-slim
```

Uses the Python 3.11 Slim image as the base image.

### Metadata

```dockerfile
LABEL maintainer="vinod@devops.com"
LABEL version="1.0"
```

Adds metadata to the Docker image.

### Working Directory

```dockerfile
WORKDIR /app
```

Sets `/app` as the working directory inside the container.

### Dependency Installation

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

The dependency file is copied separately before the application source code.

> 💡 **Why?** Docker can reuse the dependency-installation layer when application source code changes but `requirements.txt` remains unchanged. This improves build performance through **layer caching**.

### Copy Application Code

```dockerfile
COPY . .
```

Copies the application files into the container's `/app` directory.

### Run as a Non-Root User

```dockerfile
RUN useradd -m appuser && chown -R appuser /app
USER appuser
```

Creates an `appuser` account and runs the application as that user rather than as root.

> 🛡️ **Security Best Practice:** Running applications as a non-root user reduces the potential impact of a container compromise.

### Expose Application Port

```dockerfile
EXPOSE 5000
```

Documents that the application listens on port `5000`.

### Health Check

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:5000/ || exit 1
```

Defines a health check that tests whether the Flask application is responding.

> ⚠️ **Important:** The provided `python:3.11-slim` image may not include the `curl` utility. If `curl` is unavailable, this health check will fail unless `curl` is installed or the health check is implemented using another available mechanism.

### Start Command

```dockerfile
CMD ["python", "app.py"]
```

Starts the Flask application when the container runs.

---

# 🏗️ Docker Image Build Commands

## Build an Image

```bash
docker build -t myapp:1.0 .
```

Builds the Docker image and tags it as:

```text
myapp:1.0
```

---

## Build Without Cache

```bash
docker build -t myapp:1.0 --no-cache .
```

Builds the image without using previously cached layers.

> 💡 **Use case:** A no-cache build can be useful when troubleshooting build behavior or ensuring that cached layers are not reused.

---

## List Images

```bash
docker images
```

Displays locally available Docker images.

---

## View Image Layer History

```bash
docker history myapp:1.0
```

Displays the layer history of the `myapp:1.0` image.

This helps you understand how Docker constructed the image.

---

## Tag an Image for Docker Hub

```bash
docker tag myapp:1.0 username/myapp:1.0
```

Tags the local image using the Docker Hub repository naming convention.

---

## Log in to Docker Hub

```bash
docker login
```

Authenticates with Docker Hub.

---

## Push an Image to Docker Hub

```bash
docker push username/myapp:1.0
```

Uploads the image to the specified Docker Hub repository.

---

## Remove an Image

```bash
docker rmi myapp:1.0
```

Removes the specified local image.

---

## Remove Dangling Images

```bash
docker image prune -f
```

Removes dangling Docker images.

> ⚠️ **Caution:** Always review cleanup commands before running them on shared or production systems.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                                                                   |
| -------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **T1.1** | Create `app.py` (Flask: return `'Hello Docker!'`), `requirements.txt` (flask), Dockerfile as above. Build: `docker build -t flask-app:1.0 .` |
| **T1.2** | Run: `docker run -d -p 5000:5000 flask-app:1.0`. Test: `curl localhost:5000`                                                                 |
| **T1.3** | Push to Docker Hub: `docker tag flask-app:1.0 /flask-app:1.0 && docker push`                                                                 |

---

## T1.1 — Create and Build the Flask Application

Create:

```text
app.py
requirements.txt
Dockerfile
```

### `app.py`

Create a Flask application that returns:

```text
Hello Docker!
```

### `requirements.txt`

Add:

```text
flask
```

### `Dockerfile`

Use the Dockerfile provided in this lab.

Build the image:

```bash
docker build -t flask-app:1.0 .
```

Verify the image:

```bash
docker images
```

---

## T1.2 — Run and Test the Flask Container

Run the application:

```bash
docker run -d -p 5000:5000 flask-app:1.0
```

Verify that the container is running:

```bash
docker ps
```

Test the application:

```bash
curl localhost:5000
```

The application should return:

```text
Hello Docker!
```

> ⚠️ **Note:** Flask must listen on an address reachable from outside the container, commonly `0.0.0.0`, rather than only `127.0.0.1`, when using Docker port publishing.

---

## T1.3 — Push to Docker Hub

Tag the image for Docker Hub:

```bash
docker tag flask-app:1.0 <username>/flask-app:1.0
```

Then push it:

```bash
docker push <username>/flask-app:1.0
```

> 📝 **Note:** The original command contains `/flask-app:1.0`. Replace the missing username portion with your actual Docker Hub username.

For example:

```bash
docker tag flask-app:1.0 myusername/flask-app:1.0
docker push myusername/flask-app:1.0
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                             |
| -------- | ------------------------------------------------------------------------------------------------------ |
| **T2.1** | Build same Dockerfile twice — observe cache hits on second build (much faster).                        |
| **T2.2** | Swap `FROM python:3.11-slim` to `python:3.11` — compare image sizes with `docker images`.              |
| **T2.3** | Add a `.dockerignore` file excluding `.git`, `__pycache__`, `*.pyc`. Rebuild and compare context size. |

---

## T2.1 — Observe Docker Layer Caching

Build the Dockerfile:

```bash
docker build -t flask-app:1.0 .
```

Build the same image again:

```bash
docker build -t flask-app:1.0 .
```

Observe the second build carefully.

Look for cached layers and compare the build duration.

> 💡 **Key Concept:** Docker can reuse unchanged layers. This is why placing relatively stable instructions such as dependency installation before frequently changing application code can significantly improve build performance.

---

## T2.2 — Compare Python Base Images

Change:

```dockerfile
FROM python:3.11-slim
```

to:

```dockerfile
FROM python:3.11
```

Build the image again and compare the sizes:

```bash
docker images
```

Observe the difference between:

```text
python:3.11-slim
python:3.11
```

> 💡 **Tip:** Smaller base images can reduce storage and transfer requirements, but image size should be balanced against compatibility, security, debugging requirements, and operational needs.

---

## T2.3 — Create a `.dockerignore`

Create:

```text
.dockerignore
```

Add:

```text
.git
__pycache__
*.pyc
```

The `.dockerignore` file prevents these files and directories from being sent as part of the Docker build context.

Rebuild the image:

```bash
docker build -t flask-app:1.0 .
```

Compare the build context size and behavior before and after adding `.dockerignore`.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                                |
| -------- | --------------------------------------------------------------------------------------------------------- |
| **T3.1** | Multi-stage build: Stage 1 builds Node.js app, Stage 2 copies only `dist/` to nginx. Compare image sizes. |
| **T3.2** | Scan your image: `docker scout cves flask-app:1.0` or `trivy image flask-app:1.0`. Fix any CRITICAL CVEs. |
| **T3.3** | Push to AWS ECR: create repo, authenticate, tag, push. Pull from ECR on a fresh EC2.                      |

---

## T3.1 — Build a Multi-Stage Docker Image

Create a **multi-stage Dockerfile**.

The build should contain:

### Stage 1 — Build

Use a Node.js image to:

* Install dependencies.
* Build the Node.js application.
* Generate the `dist/` directory.

### Stage 2 — Runtime

Use Nginx as the runtime image.

Copy only the generated `dist/` directory from Stage 1 into Nginx.

Conceptually:

```text
Stage 1
Node.js
   │
   ├── Source Code
   ├── Dependencies
   └── Build
        │
        ▼
      dist/
        │
        ▼
Stage 2
Nginx
   │
   └── dist/ only
```

Compare the resulting image size with a single-stage implementation.

> 💡 **Key Concept:** Multi-stage builds allow build-time dependencies and source files to remain outside the final runtime image, often resulting in a smaller and cleaner production image.

---

## T3.2 — Scan the Docker Image

Use Docker Scout:

```bash
docker scout cves flask-app:1.0
```

Or use Trivy:

```bash
trivy image flask-app:1.0
```

Review the vulnerability report.

Identify any:

```text
CRITICAL
```

vulnerabilities.

Then work to remediate the identified **CRITICAL CVEs**.

Possible remediation approaches include:

* Updating the base image.
* Updating application dependencies.
* Updating operating system packages.
* Rebuilding the image.
* Re-running the security scan.

> 🛡️ **Security Best Practice:** Do not assume that a successful image build means the image is secure. Container images should be scanned regularly throughout the software supply chain.

> ⚠️ **Important:** CVEs can originate from the base image, operating system packages, or application dependencies. Remediation should be based on the actual vulnerability report rather than blindly upgrading packages.

---

## T3.3 — Push the Image to AWS ECR

Push the Docker image to **Amazon Elastic Container Registry (ECR)**.

The workflow is:

```text
Create ECR Repository
        │
        ▼
Authenticate Docker
        │
        ▼
Tag Image
        │
        ▼
Push Image
        │
        ▼
Pull Image from Fresh EC2
```

### Step 1 — Create an ECR Repository

Create an ECR repository for the application.

For example:

```text
flask-app
```

### Step 2 — Authenticate Docker

Authenticate Docker against your AWS ECR registry using the AWS CLI.

The standard workflow is:

```bash
aws ecr get-login-password --region ap-south-1 | \
docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
```

Replace:

```text
<AWS_ACCOUNT_ID>
```

with your AWS account ID.

### Step 3 — Tag the Image

Tag the local image for ECR:

```bash
docker tag flask-app:1.0 \
<AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/flask-app:1.0
```

### Step 4 — Push the Image

Push the image:

```bash
docker push \
<AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/flask-app:1.0
```

### Step 5 — Pull from a Fresh EC2 Instance

On a fresh EC2 instance:

1. Install Docker.
2. Configure AWS credentials or an appropriate IAM role.
3. Authenticate Docker against ECR.
4. Pull the image.
5. Run the container.
6. Test the application.

Pull:

```bash
docker pull \
<AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/flask-app:1.0
```

Run:

```bash
docker run -d -p 5000:5000 \
<AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/flask-app:1.0
```

Test:

```bash
curl localhost:5000
```

Expected response:

```text
Hello Docker!
```

> 🛡️ **IAM Best Practice:** For EC2 workloads, prefer an appropriately scoped **IAM role** over storing long-lived AWS access keys directly on the server.

---

# 🧠 Docker Layer Caching

Dockerfiles are built as a sequence of layers.

For example:

```text
FROM python:3.11-slim
        │
        ▼
COPY requirements.txt
        │
        ▼
RUN pip install ...
        │
        ▼
COPY . .
        │
        ▼
USER appuser
        │
        ▼
CMD ...
```

When a Dockerfile instruction and its relevant inputs have not changed, Docker can often reuse the corresponding cached layer.

### Recommended Pattern

Place relatively stable dependencies before frequently changing source code:

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

This is generally more efficient than:

```dockerfile
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
```

because changes to application source files can invalidate later layers.

---

# 🛡️ Docker Security Best Practices

Throughout this lab, follow these practices:

* Use minimal, trusted base images.
* Keep base images and dependencies updated.
* Run applications as a non-root user.
* Use `.dockerignore`.
* Avoid embedding secrets in Dockerfiles.
* Scan images for known vulnerabilities.
* Remediate critical vulnerabilities before deployment.
* Use immutable version tags where practical.
* Avoid relying exclusively on the `latest` tag.
* Remove unnecessary packages and files from production images.
* Push production images only to trusted registries.
* Use IAM roles rather than long-lived AWS credentials where possible.

---

# 🧹 Lab Cleanup

Review running containers:

```bash
docker ps
```

Review all containers:

```bash
docker ps -a
```

Stop the Flask container if it is running:

```bash
docker stop <container-id-or-name>
```

Remove it:

```bash
docker rm <container-id-or-name>
```

Review images:

```bash
docker images
```

Remove the lab image if no longer required:

```bash
docker rmi flask-app:1.0
```

Clean up dangling images:

```bash
docker image prune -f
```

> ⚠️ **Caution:** Do not remove images or containers that are being used by other applications or lab exercises.

---

# ✅ Lab 05 Completion Checklist

* [ ] Understand Docker images and Dockerfiles.
* [ ] Create a Python Flask application.
* [ ] Create `requirements.txt`.
* [ ] Create a Dockerfile.
* [ ] Build a custom Docker image.
* [ ] Run the Flask application in a container.
* [ ] Test the application using `curl`.
* [ ] Understand Docker layer caching.
* [ ] Compare `python:3.11-slim` and `python:3.11`.
* [ ] Create and use `.dockerignore`.
* [ ] Tag an image for Docker Hub.
* [ ] Push an image to Docker Hub.
* [ ] Build a multi-stage Docker image.
* [ ] Scan a Docker image for vulnerabilities.
* [ ] Identify and remediate CRITICAL CVEs.
* [ ] Create an AWS ECR repository.
* [ ] Authenticate Docker with ECR.
* [ ] Tag and push an image to ECR.
* [ ] Pull the image from ECR on a fresh EC2 instance.
* [ ] Run and test the application from ECR.
