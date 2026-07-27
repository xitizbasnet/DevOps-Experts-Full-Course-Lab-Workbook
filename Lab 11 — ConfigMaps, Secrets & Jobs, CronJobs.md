## Lab 11 — ConfigMaps, Secrets & Jobs/CronJobs

### 🎯 Objective

Externalize configuration with **ConfigMaps**, store sensitive data in **Secrets**, and automate tasks with **Jobs and CronJobs**.

This lab covers:

* ConfigMaps for non-sensitive application configuration
* Secrets for sensitive values
* Environment-variable injection
* Configuration mounted as files
* Jobs and CronJobs for automated tasks
* AWS Secrets Manager integration with EKS
* Backing up PVC data to Amazon S3
* Sealed Secrets for GitOps-safe secret management

---

# 🧠 Key Concepts

Kubernetes applications should separate **application configuration** from the container image.

```text
┌─────────────────────────────────────────────┐
│              Kubernetes Workload            │
│                                             │
│  Application                                │
│      │                                      │
│      ├──────────────► ConfigMap              │
│      │                 Non-sensitive config  │
│      │                                      │
│      ├──────────────► Secret                 │
│      │                 Sensitive values      │
│      │                                      │
│      └──────────────► Job / CronJob           │
│                        Automated tasks       │
└─────────────────────────────────────────────┘
```

---

# ⚙️ ConfigMaps

A **ConfigMap** stores non-sensitive configuration data.

Examples include:

* Database hostnames
* Application environment names
* Feature flags
* Configuration files
* Application settings

> 💡 **Best Practice:** Do not store passwords, API keys, tokens, or other sensitive values in a ConfigMap.

---

# 🔐 Secrets

A Kubernetes **Secret** is designed for sensitive configuration such as:

* Passwords
* API keys
* Tokens
* Certificates
* Credentials

> ⚠️ **Important:** Kubernetes Secret values are commonly represented as Base64-encoded data. **Base64 encoding is not encryption.** Production environments should use appropriate encryption-at-rest and external secret-management controls.

---

# ⏱️ Jobs and CronJobs

A **Job** runs a task until it completes successfully.

A **CronJob** creates Jobs according to a schedule.

```text
CronJob
   │
   ├── Job
   │     └── Pod
   │          └── Task
   │
   ├── Job
   │     └── Pod
   │          └── Task
   │
   └── Job
         └── Pod
              └── Task
```

---

# 📦 ConfigMap & Secret

## ConfigMap from Literal

Create a ConfigMap directly from literal values:

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql \
  --from-literal=APP_ENV=production
```

Verify:

```bash
kubectl get configmap app-config
```

View its contents:

```bash
kubectl describe configmap app-config
```

---

## ConfigMap from File

Create a ConfigMap from a configuration file:

```bash
kubectl create configmap nginx-conf \
  --from-file=nginx.conf
```

Verify:

```bash
kubectl get configmap nginx-conf
```

Inspect:

```bash
kubectl describe configmap nginx-conf
```

---

# 🔑 Create a Secret

Create a generic Kubernetes Secret:

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=SuperSecret123
```

Kubernetes automatically Base64-encodes the value when storing it in the Secret's `data` field.

Verify:

```bash
kubectl get secret db-secret
```

Inspect metadata:

```bash
kubectl describe secret db-secret
```

> ⚠️ **Security Warning:** `SuperSecret123` is intentionally preserved from the original lab exercise. Do not use this password in a production environment.

---

# 🔗 Using ConfigMap and Secret in a Pod

In a Pod specification:

```yaml
envFrom:
  - configMapRef:
      name: app-config

  - secretRef:
      name: db-secret
```

This makes the ConfigMap and Secret values available as environment variables inside the container.

Conceptually:

```text
ConfigMap
   │
   ├── DB_HOST=mysql
   └── APP_ENV=production
             │
             ▼
          Pod ENV

Secret
   │
   └── DB_PASSWORD=********
             │
             ▼
          Pod ENV
```

---

# ⏰ CronJob — Runs Every 5 Minutes

The following CronJob runs every five minutes:

```yaml
apiVersion: batch/v1
kind: CronJob

metadata:
  name: log-cleanup

spec:
  schedule: "*/5 * * * *"

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
            - name: cleanup
              image: busybox
              command:
                - /bin/sh
                - -c
                - echo Cleanup at $(date)
```

The schedule:

```text
*/5 * * * *
```

means:

```text
Every 5 minutes
```

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                                                              |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| **T1.1** | Create ConfigMap with `DB_HOST` and `APP_ENV`. Mount as environment variables in an nginx Pod. Verify: `kubectl exec -- env \| grep DB` |
| **T1.2** | Create Secret with `DB_PASSWORD`. Mount as environment variable. Verify the secret is not visible in Pod `describe`.                    |
| **T1.3** | Create CronJob running every minute. After 3 minutes: `kubectl get jobs` — list completed Jobs.                                         |

---

# T1.1 — ConfigMap as Environment Variables

Create:

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql \
  --from-literal=APP_ENV=production
```

Verify:

```bash
kubectl get configmap app-config
```

Create an nginx Pod:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-config
  namespace: dev

spec:
  containers:
    - name: nginx
      image: nginx

      envFrom:
        - configMapRef:
            name: app-config
```

Apply:

```bash
kubectl apply -f nginx-config.yaml
```

Verify:

```bash
kubectl get pod nginx-config -n dev
```

Check the environment variables:

```bash
kubectl exec nginx-config -n dev -- env | grep DB
```

Expected:

```text
DB_HOST=mysql
```

Check the application environment:

```bash
kubectl exec nginx-config -n dev -- env | grep APP
```

Expected:

```text
APP_ENV=production
```

---

# T1.2 — Secret as an Environment Variable

Create:

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=SuperSecret123
```

Create a Pod:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-secret
  namespace: dev

spec:
  containers:
    - name: nginx
      image: nginx

      envFrom:
        - secretRef:
            name: db-secret
```

Apply:

```bash
kubectl apply -f nginx-secret.yaml
```

Check:

```bash
kubectl get pod nginx-secret -n dev
```

Inspect the Pod:

```bash
kubectl describe pod nginx-secret -n dev
```

The Secret's actual value should **not** be displayed by `kubectl describe pod`.

You can verify that the environment variable exists inside the container:

```bash
kubectl exec nginx-secret -n dev -- env | grep DB_PASSWORD
```

> ⚠️ **Security Note:** Be careful with this command because it prints the secret value to your terminal. Avoid running it in shared terminals, CI logs, screenshots, or recorded sessions.

---

# T1.3 — CronJob Running Every Minute

Create:

```yaml
apiVersion: batch/v1
kind: CronJob

metadata:
  name: log-cleanup
  namespace: dev

spec:
  schedule: "* * * * *"

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
            - name: cleanup
              image: busybox
              command:
                - /bin/sh
                - -c
                - echo Cleanup at $(date)
```

Apply:

```bash
kubectl apply -f cronjob.yaml
```

Check:

```bash
kubectl get cronjobs -n dev
```

Wait a few minutes and run:

```bash
kubectl get jobs -n dev
```

You should see Jobs created by the CronJob.

For more details:

```bash
kubectl describe cronjob log-cleanup -n dev
```

View the generated Pods:

```bash
kubectl get pods -n dev
```

View logs:

```bash
kubectl logs <pod-name> -n dev
```

Expected output will resemble:

```text
Cleanup at Mon Jul 27 10:00:00 UTC 2026
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                                                      |
| -------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **T2.1** | Mount ConfigMap as a volume file. Pod reads `nginx.conf` from `/etc/nginx/custom.conf`.                                         |
| **T2.2** | Update ConfigMap value. Does running Pod pick it up? *(Hint: volume mount does, environment variable does not without restart)* |
| **T2.3** | Decode a Secret: `kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' \| base64 -d`                                  |

---

# T2.1 — Mount ConfigMap as a Volume

Create a sample configuration file:

```bash
cat > nginx.conf <<'EOF'
server {
    listen 80;

    location / {
        return 200 "Hello from ConfigMap\n";
    }
}
EOF
```

Create the ConfigMap:

```bash
kubectl create configmap nginx-conf \
  --from-file=nginx.conf \
  -n dev
```

Create a Pod that mounts the ConfigMap:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-config-file
  namespace: dev

spec:
  containers:
    - name: nginx
      image: nginx

      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/custom.conf
          subPath: nginx.conf

  volumes:
    - name: nginx-config
      configMap:
        name: nginx-conf
```

Apply:

```bash
kubectl apply -f nginx-config-file.yaml
```

Verify the mounted file:

```bash
kubectl exec nginx-config-file -n dev -- \
  cat /etc/nginx/custom.conf
```

The ConfigMap data should be visible in the mounted file.

> 💡 **Important:** The `subPath` technique has different update behavior from mounting the entire ConfigMap directory. When using `subPath`, updates to the ConfigMap are not automatically reflected in the mounted file.

---

# T2.2 — Update ConfigMap

Update the ConfigMap:

```bash
kubectl edit configmap app-config -n dev
```

Change:

```text
APP_ENV=production
```

to another value, for example:

```text
APP_ENV=testing
```

Check the ConfigMap:

```bash
kubectl get configmap app-config -n dev -o yaml
```

### Environment Variables

If the ConfigMap is consumed using:

```yaml
envFrom:
  - configMapRef:
      name: app-config
```

the running Pod does **not** automatically receive the updated environment variable.

Restart the Pod:

```bash
kubectl delete pod nginx-config -n dev
```

Recreate it:

```bash
kubectl apply -f nginx-config.yaml
```

Then verify:

```bash
kubectl exec nginx-config -n dev -- env | grep APP_ENV
```

### ConfigMap Volume

When a ConfigMap is mounted as a regular volume, Kubernetes can update the projected files after the ConfigMap changes, subject to Kubernetes' update mechanism and the mount configuration.

> 💡 **Key Learning:**
> **ConfigMap as environment variable → Pod restart required.**
> **ConfigMap as volume → Kubernetes can refresh the mounted data, subject to mount configuration.**

---

# T2.3 — Decode a Secret

Retrieve the encoded Secret value:

```bash
kubectl get secret db-secret \
  -o jsonpath='{.data.DB_PASSWORD}'
```

Decode it:

```bash
kubectl get secret db-secret \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

Expected lab output:

```text
SuperSecret123
```

> ⚠️ **Security Note:** Base64 is encoding, not encryption. Anyone who can retrieve the Secret's `data` field may be able to decode it. Production Kubernetes clusters should use encryption at rest and strong RBAC controls, and many organizations integrate Kubernetes with dedicated secret-management systems.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                           |
| -------- | ---------------------------------------------------------------------------------------------------- |
| **T3.1** | Integrate AWS Secrets Manager with EKS using External Secrets Operator. Sync a Secret to Kubernetes. |
| **T3.2** | CronJob: back up PVC data to S3 every night at 2 AM using AWS CLI inside the Job container.          |
| **T3.3** | Implement Sealed Secrets to encrypt Kubernetes Secrets at rest in Git (GitOps-safe Secrets).         |

---

# T3.1 — AWS Secrets Manager + External Secrets Operator

For production environments, avoid manually copying sensitive credentials into Kubernetes manifests whenever possible.

A recommended architecture is:

```text
                    AWS Secrets Manager
                            │
                            │ Secret
                            ▼
                External Secrets Operator
                            │
                            ▼
                 Kubernetes Secret
                            │
                            ▼
                         Pod
```

The **External Secrets Operator (ESO)** can synchronize secrets from AWS Secrets Manager into Kubernetes Secrets.

### Step 1 — Install External Secrets Operator

Follow the current External Secrets Operator installation documentation for your EKS environment.

Verify:

```bash
kubectl get pods -n external-secrets
```

Check CRDs:

```bash
kubectl get crd | grep external-secrets
```

---

### Step 2 — Create Secret in AWS Secrets Manager

Create a secret in AWS Secrets Manager containing the application credential.

For example:

```text
Secret name:
devops/db-password
```

Do not place the real production password directly into Git.

---

### Step 3 — Configure AWS Access

The External Secrets Operator needs permission to read the secret.

For EKS, use an appropriate AWS IAM integration such as:

* EKS Pod Identity
* IAM Roles for Service Accounts (IRSA)

The IAM policy should grant only the required Secrets Manager permissions.

For example, conceptually:

```text
secretsmanager:GetSecretValue
```

for the specific secret ARN.

> 🔐 **Best Practice:** Follow least privilege. Avoid granting broad permissions such as access to every secret in the AWS account.

---

### Step 4 — Create an ExternalSecret

Example:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret

metadata:
  name: db-secret
  namespace: dev

spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore

  target:
    name: db-secret

  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: devops/db-password
```

Apply:

```bash
kubectl apply -f external-secret.yaml
```

Check:

```bash
kubectl get externalsecret -n dev
```

Verify the Kubernetes Secret:

```bash
kubectl get secret db-secret -n dev
```

Inspect the ExternalSecret:

```bash
kubectl describe externalsecret db-secret -n dev
```

The workflow is:

```text
AWS Secrets Manager
        │
        │ GetSecretValue
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
        │
        ▼
Application Pod
```

---

# T3.2 — Back Up PVC Data to Amazon S3

The challenge is to create a CronJob that:

1. Runs every night at **2 AM**.
2. Mounts the PVC.
3. Reads backup data.
4. Uses AWS CLI.
5. Uploads the backup to Amazon S3.

Cron schedule:

```text
0 2 * * *
```

Architecture:

```text
                 Kubernetes CronJob
                         │
                         ▼
                    Backup Pod
                         │
                 ┌───────┴───────┐
                 ▼               ▼
                PVC            AWS CLI
                 │               │
                 │               ▼
                 │            Amazon S3
                 │
                 └── Backup data
```

A simplified example:

```yaml
apiVersion: batch/v1
kind: CronJob

metadata:
  name: pvc-s3-backup
  namespace: dev

spec:
  schedule: "0 2 * * *"

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
            - name: backup
              image: amazon/aws-cli

              command:
                - /bin/sh
                - -c

              args:
                - |
                  tar -czf /tmp/pvc-backup.tar.gz /backup-data
                  aws s3 cp \
                    /tmp/pvc-backup.tar.gz \
                    s3://YOUR-BUCKET/pvc-backups/

              volumeMounts:
                - name: application-data
                  mountPath: /backup-data

          volumes:
            - name: application-data
              persistentVolumeClaim:
                claimName: app-pvc
```

> ⚠️ **Important:** Replace `YOUR-BUCKET` with your actual S3 bucket. Do not put AWS access keys directly into the YAML.

### AWS Authentication

The Pod should use an AWS IAM identity mechanism such as **EKS Pod Identity or IRSA**.

The role should have only the required S3 permissions, such as:

```text
s3:PutObject
s3:ListBucket
```

for the appropriate bucket/prefix.

Apply:

```bash
kubectl apply -f pvc-s3-backup.yaml
```

Check:

```bash
kubectl get cronjob -n dev
```

Check Jobs:

```bash
kubectl get jobs -n dev
```

Check the generated Pod:

```bash
kubectl get pods -n dev
```

View logs:

```bash
kubectl logs <backup-pod> -n dev
```

Verify the S3 backup:

```bash
aws s3 ls s3://YOUR-BUCKET/pvc-backups/
```

> 💡 **Production Note:** For large datasets, application-aware database backups are often preferable to blindly archiving a live database filesystem. Ensure backup consistency, retention, encryption, lifecycle policies, and restore testing are part of the design.

---

# T3.3 — Sealed Secrets for GitOps-Safe Secrets

A normal Kubernetes Secret should not be committed directly to Git in plaintext.

For example, this should **not** be committed to a repository:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
data:
  DB_PASSWORD: <secret-value>
```

Even if the value is Base64-encoded, it is not protected by encryption.

A common GitOps approach is:

```text
Developer
    │
    │ plaintext Secret
    ▼
kubeseal
    │
    │ encrypted SealedSecret
    ▼
Git Repository
    │
    ▼
GitOps Controller
    │
    ▼
Kubernetes
    │
    ▼
Kubernetes Secret
```

### Sealed Secrets Concept

The Sealed Secrets controller uses a cluster-side private key to decrypt the encrypted Secret.

The encrypted object can safely be stored in Git, subject to your organization's security requirements and key-management practices.

Conceptually:

```text
Plain Secret
     │
     ▼
kubeseal
     │
     ▼
SealedSecret
     │
     ▼
Git
     │
     ▼
Kubernetes
     │
     ▼
Sealed Secrets Controller
     │
     ▼
Kubernetes Secret
```

Install the Sealed Secrets controller according to the current project documentation.

Verify:

```bash
kubectl get pods -n kube-system | grep sealed
```

Create a normal Secret manifest without applying it:

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=SuperSecret123 \
  -n dev \
  --dry-run=client \
  -o yaml > secret.yaml
```

Seal it:

```bash
kubeseal \
  --format yaml \
  < secret.yaml \
  > sealed-secret.yaml
```

The resulting `sealed-secret.yaml` can be committed to Git.

Apply it:

```bash
kubectl apply -f sealed-secret.yaml
```

Verify:

```bash
kubectl get sealedsecret -n dev
```

Then:

```bash
kubectl get secret db-secret -n dev
```

> 🔐 **Security Note:** Treat the Sealed Secrets controller's private key as highly sensitive. Losing or improperly managing the key can affect your ability to decrypt previously sealed secrets.

---

# 🔧 Useful Commands

## List ConfigMaps

```bash
kubectl get configmaps -n dev
```

## Describe ConfigMap

```bash
kubectl describe configmap app-config -n dev
```

## View ConfigMap YAML

```bash
kubectl get configmap app-config -n dev -o yaml
```

## List Secrets

```bash
kubectl get secrets -n dev
```

## Describe Secret

```bash
kubectl describe secret db-secret -n dev
```

## View Secret Metadata

```bash
kubectl get secret db-secret -n dev -o yaml
```

> ⚠️ Avoid sharing the output if it contains sensitive data.

## List Jobs

```bash
kubectl get jobs -n dev
```

## List CronJobs

```bash
kubectl get cronjobs -n dev
```

## Describe CronJob

```bash
kubectl describe cronjob log-cleanup -n dev
```

## View Job Logs

```bash
kubectl logs job/<job-name> -n dev
```

## Delete a CronJob

```bash
kubectl delete cronjob log-cleanup -n dev
```

---

# 🛡️ Best-Practice Tips

* Use **ConfigMaps** for non-sensitive configuration.
* Use **Secrets** for sensitive values.
* Remember that **Base64 encoding is not encryption**.
* Never commit plaintext credentials to Git.
* Use AWS Secrets Manager for sensitive production credentials where appropriate.
* Consider **External Secrets Operator** for synchronizing external secrets into Kubernetes.
* Use EKS Pod Identity or IRSA rather than hard-coded AWS credentials.
* Follow least-privilege IAM policies.
* Do not expose secrets through application logs.
* Avoid printing secrets with `kubectl exec ... env`.
* Use CronJobs for recurring operational tasks.
* Use Jobs for one-time batch workloads.
* Configure appropriate `restartPolicy` values.
* Set resource requests and limits for production workloads.
* Configure CronJob history limits and concurrency behavior where appropriate.
* Store backups separately from the primary data.
* Use S3 encryption, lifecycle policies, retention, and access controls for production backups.
* Test restoring backups regularly.
* Use Sealed Secrets or an enterprise secret-management solution for GitOps workflows.
* Protect the Sealed Secrets controller's encryption keys.
* Review Kubernetes RBAC permissions for access to Secrets.
* Enable encryption at rest for Kubernetes Secrets where supported and required.

---

# 🚨 Troubleshooting

## ConfigMap Not Found

Check:

```bash
kubectl get configmaps -n dev
```

Make sure the Pod and ConfigMap are in the same namespace.

For example:

```text
Pod:
namespace: dev

ConfigMap:
namespace: dev
```

---

## Secret Not Found

Check:

```bash
kubectl get secrets -n dev
```

Then:

```bash
kubectl describe secret db-secret -n dev
```

Confirm the Pod references the correct Secret name.

---

## Environment Variable Not Updated

If the ConfigMap or Secret is consumed using:

```yaml
envFrom:
```

the running Pod will not automatically receive changes.

Restart/recreate the Pod:

```bash
kubectl delete pod <pod-name> -n dev
```

---

## CronJob Is Not Creating Jobs

Check:

```bash
kubectl get cronjobs -n dev
```

Then:

```bash
kubectl describe cronjob <cronjob-name> -n dev
```

Check:

```text
Events
Schedule
Last Schedule Time
Active
```

Also check:

```bash
kubectl get jobs -n dev
```

---

## Job Failed

Check:

```bash
kubectl get jobs -n dev
```

Find the generated Pod:

```bash
kubectl get pods -n dev
```

Then:

```bash
kubectl logs <pod-name> -n dev
```

And:

```bash
kubectl describe pod <pod-name> -n dev
```

---

## ExternalSecret Is Not Ready

Check:

```bash
kubectl get externalsecret -n dev
```

Then:

```bash
kubectl describe externalsecret db-secret -n dev
```

Check:

* AWS IAM permissions
* AWS Secrets Manager secret name
* SecretStore configuration
* EKS Pod Identity/IRSA configuration
* External Secrets Operator Pods
* Kubernetes Events

---

# 🧹 Lab Cleanup

Delete the test Pods:

```bash
kubectl delete pod nginx-config nginx-secret nginx-config-file -n dev
```

Delete ConfigMaps:

```bash
kubectl delete configmap app-config nginx-conf -n dev
```

Delete Secret:

```bash
kubectl delete secret db-secret -n dev
```

Delete CronJob:

```bash
kubectl delete cronjob log-cleanup -n dev
```

Delete the S3 backup CronJob:

```bash
kubectl delete cronjob pvc-s3-backup -n dev
```

Remove ExternalSecret if created:

```bash
kubectl delete externalsecret db-secret -n dev
```

Verify:

```bash
kubectl get configmaps -n dev
kubectl get secrets -n dev
kubectl get cronjobs -n dev
kubectl get jobs -n dev
```

> ⚠️ **Important:** Deleting a Kubernetes Secret does not automatically delete the corresponding AWS Secrets Manager secret. Review cloud resources separately before cleanup.

---

# 📋 Lab 11 Completion Checklist

* [ ] Understand the purpose of ConfigMaps.
* [ ] Create a ConfigMap from literal values.
* [ ] Create a ConfigMap from a file.
* [ ] Inject ConfigMap values into a Pod as environment variables.
* [ ] Mount a ConfigMap as a file.
* [ ] Understand ConfigMap update behavior.
* [ ] Understand the purpose of Kubernetes Secrets.
* [ ] Create a Secret from a literal.
* [ ] Inject a Secret as an environment variable.
* [ ] Understand that Base64 is encoding, not encryption.
* [ ] Understand why Secrets should not be committed directly to Git.
* [ ] Create a Job/CronJob.
* [ ] Run a CronJob every minute.
* [ ] Check generated Jobs.
* [ ] View CronJob Pod logs.
* [ ] Understand Cron scheduling syntax.
* [ ] Integrate AWS Secrets Manager with EKS.
* [ ] Understand External Secrets Operator.
* [ ] Configure appropriate AWS IAM permissions.
* [ ] Sync an AWS secret to a Kubernetes Secret.
* [ ] Create a scheduled PVC-to-S3 backup workflow.
* [ ] Understand EKS Pod Identity/IRSA for AWS authentication.
* [ ] Avoid hard-coded AWS credentials.
* [ ] Understand Sealed Secrets.
* [ ] Encrypt Kubernetes Secret manifests for GitOps workflows.
* [ ] Understand Sealed Secrets controller key management.
* [ ] Apply least-privilege security principles.
* [ ] Clean up lab resources safely.

---

# 🧠 Lab 11 — Key Takeaways

| Component                     | Primary Purpose                        | Example                 |
| ----------------------------- | -------------------------------------- | ----------------------- |
| **ConfigMap**                 | Non-sensitive configuration            | `DB_HOST=mysql`         |
| **Secret**                    | Sensitive configuration                | Database password       |
| **Job**                       | One-time task                          | Database migration      |
| **CronJob**                   | Scheduled task                         | Nightly backup          |
| **AWS Secrets Manager**       | Centralized cloud secret storage       | Production credentials  |
| **External Secrets Operator** | Sync external secrets into Kubernetes  | AWS → Kubernetes Secret |
| **Sealed Secrets**            | GitOps-safe encrypted Secret manifests | Git repository          |
| **Amazon S3**                 | Durable backup/object storage          | PVC backup repository   |

The overall production-oriented pattern is:

```text
                 AWS Secrets Manager
                         │
                         ▼
              External Secrets Operator
                         │
                         ▼
                 Kubernetes Secret
                         │
                         ▼
Application ──────── Kubernetes Pod
     │                   │
     │                   │
     ▼                   ▼
ConfigMap             PVC
     │                   │
     │                   ▼
     │               Persistent Data
     │                   │
     │                   ▼
     │               Backup CronJob
     │                   │
     └───────────────────┼──────────► Amazon S3
                         │
                         ▼
                    GitOps Workflow
                         │
                         ▼
                  Sealed Secrets
```

> **Operational principle:** Keep configuration external to container images, keep sensitive credentials out of source control, automate repeatable operational tasks, use cloud-native identity instead of static credentials, and maintain independent, tested backups for persistent data.
