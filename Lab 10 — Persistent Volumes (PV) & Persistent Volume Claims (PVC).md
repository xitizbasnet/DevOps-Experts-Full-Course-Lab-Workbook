# Session 4 — Kubernetes Advanced

## Lab 10 — Persistent Volumes (PV) & Persistent Volume Claims (PVC)

### 🎯 Objective

Provide **durable storage to Pods** using:

* **Persistent Volumes (PV)**
* **Persistent Volume Claims (PVC)**
* **StorageClasses**
* Dynamic volume provisioning
* Persistent application data across Pod deletion and recreation

By the end of this lab, you will understand how Kubernetes separates **application workloads from persistent storage** and how storage can survive Pod lifecycle events.

---

# 🧠 Storage Architecture

Containers and Pods are generally considered **ephemeral**. If a Pod is deleted and recreated, data stored only inside the container filesystem can be lost.

Persistent storage solves this problem.

```text
┌──────────────────────────────────────────────┐
│              Kubernetes Cluster              │
│                                              │
│   Pod                                        │
│   ┌──────────────────────────────┐           │
│   │ Application                  │           │
│   │                              │           │
│   │ /data ───────────────┐       │           │
│   └──────────────────────┼───────┘           │
│                          │                   │
│                          ▼                   │
│                  PersistentVolumeClaim      │
│                          │                   │
│                          ▼                   │
│                    PersistentVolume          │
│                          │                   │
└──────────────────────────┼───────────────────┘
                           │
                           ▼
                    Cloud Storage
```

---

# 📦 PV, PVC & StorageClass

## Persistent Volume — PV

A **PersistentVolume (PV)** represents storage available to Kubernetes.

The storage can be backed by infrastructure such as:

* AWS EBS
* AWS EFS
* Other cloud storage systems
* Storage systems in on-premises environments

A PV exists independently of an individual Pod.

---

## Persistent Volume Claim — PVC

A **PersistentVolumeClaim (PVC)** is a request for storage made by an application.

For example:

```text
Application requirement:
5 GiB storage
ReadWriteOnce
StorageClass: gp2
```

The PVC requests this storage from Kubernetes.

---

## StorageClass

A **StorageClass** defines how storage should be dynamically provisioned.

For example:

```text
StorageClass
     │
     ▼
gp2 / gp3
     │
     ▼
Dynamic volume provisioning
     │
     ▼
Persistent Volume
```

---

# 🔗 PV / PVC Relationship

The relationship is:

```text
Application Pod
      │
      │ volumeMount
      ▼
     PVC
      │
      │ claim
      ▼
      PV
      │
      │ provisioned by
      ▼
StorageClass
      │
      ▼
Cloud Storage
```

The application generally interacts with the **PVC**, rather than directly managing the underlying storage infrastructure.

---

# 📄 PVC YAML

Create:

```text
pvc.yaml
```

Use:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: app-pvc
  namespace: dev

spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: gp2

  resources:
    requests:
      storage: 5Gi
```

---

# 🔍 Understanding the PVC YAML

## API Version

```yaml
apiVersion: v1
```

PVCs belong to the Kubernetes core `v1` API.

---

## Resource Type

```yaml
kind: PersistentVolumeClaim
```

Defines this resource as a PVC.

---

## PVC Name

```yaml
metadata:
  name: app-pvc
  namespace: dev
```

The PVC is named:

```text
app-pvc
```

and belongs to:

```text
dev
```

---

## Access Mode

```yaml
accessModes:
  - ReadWriteOnce
```

`ReadWriteOnce` means the volume can be mounted as read-write by workloads on a single node, subject to the capabilities and semantics of the underlying storage implementation.

Common access modes include:

```text
ReadWriteOnce      RWO
ReadOnlyMany       ROX
ReadWriteMany      RWX
ReadWriteOncePod   RWOP
```

> 💡 **Important:** Supported access modes depend on the storage driver and underlying storage system.

---

## StorageClass

```yaml
storageClassName: gp2
```

This requests storage from the AWS EBS `gp2` StorageClass.

> ⚠️ **Important:** The workbook uses `gp2` because that is the original lab configuration. On newer EKS clusters, `gp3` is generally preferred, and the exact available StorageClasses depend on the cluster configuration.

Check available StorageClasses:

```bash
kubectl get storageclass
```

If `gp2` does not exist, inspect the available classes and adapt the YAML accordingly.

For example:

```bash
kubectl get storageclass
```

---

## Storage Request

```yaml
resources:
  requests:
    storage: 5Gi
```

The application requests:

```text
5 GiB
```

of persistent storage.

---

# 📄 Pod with PVC

Create:

```text
pod-with-pvc.yaml
```

Use:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: app-with-storage
  namespace: dev

spec:
  containers:
    - name: app
      image: nginx

      volumeMounts:
        - mountPath: /data
          name: storage

  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: app-pvc
```

---

# 🔍 Understanding the Pod Configuration

The Pod mounts the PVC:

```yaml
volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: app-pvc
```

The volume is mounted inside the container:

```yaml
volumeMounts:
  - mountPath: /data
    name: storage
```

Therefore:

```text
Container
    │
    │ /data
    ▼
PVC: app-pvc
    │
    ▼
Persistent Volume
    │
    ▼
AWS Storage
```

---

# 🚀 Deploy the PVC

Apply:

```bash
kubectl apply -f pvc.yaml
```

Check the PVC:

```bash
kubectl get pvc -n dev
```

Initially, depending on the StorageClass and provisioning workflow, you may see:

```text
STATUS
Pending
```

After successful provisioning, it should become:

```text
Bound
```

---

# 🔎 Inspect Persistent Volumes

Run:

```bash
kubectl get pv
```

If dynamic provisioning is configured correctly, Kubernetes should create or bind an appropriate PV for the claim.

You can inspect the PVC:

```bash
kubectl describe pvc app-pvc -n dev
```

Look at:

```text
Status
Volume
StorageClass
Capacity
Access Modes
Events
```

---

# 🚀 Deploy the Pod

Apply:

```bash
kubectl apply -f pod-with-pvc.yaml
```

Verify:

```bash
kubectl get pods -n dev
```

The expected status is:

```text
Running
```

---

# ✍️ Write Persistent Data

Enter the Pod:

```bash
kubectl exec -it app-with-storage -n dev -- sh
```

Inside the container:

```bash
echo "Persistent data" > /data/test.txt
```

Read the file:

```bash
cat /data/test.txt
```

Expected output:

```text
Persistent data
```

Exit:

```bash
exit
```

---

# 🔄 Test Persistence

Delete the Pod:

```bash
kubectl delete pod app-with-storage -n dev
```

Recreate it:

```bash
kubectl apply -f pod-with-pvc.yaml
```

Check:

```bash
kubectl get pods -n dev
```

Enter the new Pod:

```bash
kubectl exec -it app-with-storage -n dev -- sh
```

Check:

```bash
cat /data/test.txt
```

The file should still contain:

```text
Persistent data
```

This confirms that the data was stored on the persistent volume rather than only in the temporary container filesystem.

---

# 🟢 Task Set 1 — Guided

| Task     | What to do                                                                                   |
| -------- | -------------------------------------------------------------------------------------------- |
| **T1.1** | Apply `pvc.yaml`. Run `kubectl get pvc -n dev` — observe status `Pending` then `Bound`.      |
| **T1.2** | Apply `pod-with-pvc.yaml`. Write data: `kubectl exec ... -- sh -c 'echo test > /data/a.txt'` |
| **T1.3** | Delete and recreate the Pod. Verify `/data/a.txt` still exists (persistence confirmed).      |

---

# T1.1 — Create the PVC

Apply:

```bash
kubectl apply -f pvc.yaml
```

Check:

```bash
kubectl get pvc -n dev
```

Observe the transition:

```text
Pending
   │
   ▼
Storage provisioning
   │
   ▼
Bound
```

Then inspect:

```bash
kubectl describe pvc app-pvc -n dev
```

Check the PV:

```bash
kubectl get pv
```

> 💡 **Key Concept:** With dynamic provisioning, Kubernetes can automatically provision the underlying volume when the PVC requests storage through an appropriate StorageClass.

---

# T1.2 — Write Data to the PVC

Deploy the Pod:

```bash
kubectl apply -f pod-with-pvc.yaml
```

Write data:

```bash
kubectl exec app-with-storage -n dev -- \
  sh -c 'echo test > /data/a.txt'
```

Read it:

```bash
kubectl exec app-with-storage -n dev -- \
  cat /data/a.txt
```

Expected:

```text
test
```

---

# T1.3 — Verify Persistence

Delete the Pod:

```bash
kubectl delete pod app-with-storage -n dev
```

Recreate:

```bash
kubectl apply -f pod-with-pvc.yaml
```

Wait for the Pod:

```bash
kubectl get pod app-with-storage -n dev -w
```

Once it is `Running`, verify:

```bash
kubectl exec app-with-storage -n dev -- \
  cat /data/a.txt
```

Expected:

```text
test
```

### ✅ Persistence Confirmed

The Pod was deleted, but:

```text
Pod filesystem
     ❌ Deleted

PVC
     ✅ Retained

Persistent Volume
     ✅ Retained

Stored data
     ✅ Retained
```

---

# 🟡 Task Set 2 — Practice

| Task     | What to do                                                                                                                |
| -------- | ------------------------------------------------------------------------------------------------------------------------- |
| **T2.1** | Deploy MySQL with a PVC for `/var/lib/mysql`. Insert data. Delete the Pod. Reattach the PVC to a new Pod — data persists. |
| **T2.2** | Try `ReadWriteMany` PVC with EFS StorageClass. Mount the same PVC in 2 Pods simultaneously.                               |
| **T2.3** | `kubectl describe pvc app-pvc` — find which node the volume is bound to.                                                  |

---

# T2.1 — MySQL with Persistent Storage

MySQL stores its database files under:

```text
/var/lib/mysql
```

A persistent volume should therefore be mounted at that path.

A simplified example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: mysql-pvc
  namespace: dev

spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: gp2

  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Pod

metadata:
  name: mysql
  namespace: dev

spec:
  containers:
    - name: mysql
      image: mysql:8

      env:
        - name: MYSQL_ROOT_PASSWORD
          value: root123

      volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql

  volumes:
    - name: mysql-storage
      persistentVolumeClaim:
        claimName: mysql-pvc
```

> ⚠️ **Security Note:** `root123` is suitable only as a temporary lab password. Never use a simple password like this in a production environment. Production credentials should be managed through Kubernetes Secrets or an external secrets-management system.

Apply:

```bash
kubectl apply -f mysql.yaml
```

Check:

```bash
kubectl get pods -n dev
kubectl get pvc -n dev
```

Connect to MySQL:

```bash
kubectl exec -it mysql -n dev -- \
  mysql -uroot -proot123
```

Create test data:

```sql
CREATE DATABASE devopsdb;

USE devopsdb;

CREATE TABLE test (
    id INT PRIMARY KEY,
    message VARCHAR(255)
);

INSERT INTO test VALUES (1, 'Persistent Kubernetes Data');

SELECT * FROM test;
```

Exit:

```sql
exit
```

Delete the Pod:

```bash
kubectl delete pod mysql -n dev
```

Recreate the Pod using the same PVC.

Then verify:

```sql
SELECT * FROM devopsdb.test;
```

The database data should remain available because the data resides on the persistent volume.

> 💡 **Production Note:** A standalone MySQL Pod is useful for this lab, but production database workloads generally require a more robust architecture, such as a managed database service or a carefully designed StatefulSet/operator deployment.

---

# T2.2 — ReadWriteMany with AWS EFS

AWS EBS volumes commonly support:

```text
ReadWriteOnce
```

For multiple Pods on multiple nodes to mount the same volume simultaneously with read/write access, AWS EFS is a better fit.

Conceptually:

```text
                  EFS
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
      Pod 1                 Pod 2
        │                     │
        └────── Read/Write ───┘
```

Create a PVC using an EFS StorageClass available in your EKS environment.

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: efs-pvc
  namespace: dev

spec:
  accessModes:
    - ReadWriteMany

  storageClassName: efs-sc

  resources:
    requests:
      storage: 5Gi
```

> ⚠️ **Prerequisite:** An EFS CSI driver and an appropriate EFS StorageClass must be configured in the EKS cluster before this exercise can work.

Check:

```bash
kubectl get storageclass
```

Then:

```bash
kubectl get pvc -n dev
```

The expected access mode is:

```text
RWX
```

---

# T2.3 — Inspect PVC Details

Run:

```bash
kubectl describe pvc app-pvc -n dev
```

Review:

```text
Name
Namespace
StorageClass
Status
Volume
Labels
Annotations
Finalizers
Capacity
Access Modes
Events
```

You may also inspect the bound PV:

```bash
kubectl get pv
```

Then:

```bash
kubectl describe pv <pv-name>
```

> ⚠️ **Important:** A PVC itself does not necessarily identify a single node to which it is permanently bound. With storage such as AWS EBS, the volume has topology constraints and can be attached to a compatible node when the Pod is scheduled. The exact node relationship depends on the CSI driver, volume topology, and Pod scheduling.

---

# 🔴 Task Set 3 — Challenge

| Task     | What to do                                                                                                |
| -------- | --------------------------------------------------------------------------------------------------------- |
| **T3.1** | Automate DB backup: CronJob that mounts the same PVC and runs `mysqldump` every hour.                     |
| **T3.2** | Expand PVC storage from 5Gi to 10Gi (requires EKS gp3 StorageClass with `allowVolumeExpansion`).          |
| **T3.3** | Implement StatefulSet for MySQL with `volumeClaimTemplates`. Scale to 2 replicas — each gets its own PVC. |

---

# T3.1 — Automated Database Backup with CronJob

A Kubernetes `CronJob` can execute scheduled tasks.

The architecture is:

```text
CronJob
   │
   │ Every hour
   ▼
Backup Pod
   │
   ├── MySQL access
   │
   └── Persistent storage
           │
           ▼
       Backup file
```

A simplified CronJob example:

```yaml
apiVersion: batch/v1
kind: CronJob

metadata:
  name: mysql-backup
  namespace: dev

spec:
  schedule: "0 * * * *"

  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure

          containers:
            - name: backup
              image: mysql:8

              command:
                - /bin/sh
                - -c

              args:
                - |
                  mysqldump \
                    -h mysql \
                    -uroot \
                    -proot123 \
                    devopsdb \
                    > /backup/devopsdb.sql

              volumeMounts:
                - name: backup-storage
                  mountPath: /backup

          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: app-pvc
```

> ⚠️ **Important:** This is a lab example. In production, database credentials should not be embedded directly in the CronJob YAML. Use Kubernetes Secrets or an external secret-management system. Also, writing a backup to the **same storage volume as the database** does not protect against loss of that volume. Production backups should normally be stored separately, preferably in durable object storage such as Amazon S3.

Apply:

```bash
kubectl apply -f mysql-backup-cronjob.yaml
```

Check:

```bash
kubectl get cronjobs -n dev
```

Check Jobs:

```bash
kubectl get jobs -n dev
```

View CronJob details:

```bash
kubectl describe cronjob mysql-backup -n dev
```

---

# T3.2 — Expand PVC from 5Gi to 10Gi

First inspect the StorageClass:

```bash
kubectl get storageclass
```

Check whether expansion is supported:

```bash
kubectl get storageclass gp3 -o yaml
```

Look for:

```yaml
allowVolumeExpansion: true
```

If available, edit the PVC:

```bash
kubectl edit pvc app-pvc -n dev
```

Change:

```yaml
resources:
  requests:
    storage: 5Gi
```

to:

```yaml
resources:
  requests:
    storage: 10Gi
```

Or patch it:

```bash
kubectl patch pvc app-pvc \
  -n dev \
  -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'
```

Verify:

```bash
kubectl get pvc app-pvc -n dev
```

Then:

```bash
kubectl describe pvc app-pvc -n dev
```

> ⚠️ **Important:** Volume expansion depends on the StorageClass and CSI driver. The filesystem may also require filesystem expansion before the new capacity becomes visible inside the application container.

Check from inside the Pod:

```bash
kubectl exec -it app-with-storage -n dev -- df -h /data
```

---

# T3.3 — StatefulSet with volumeClaimTemplates

A **StatefulSet** is designed for applications that require stable identity and persistent storage.

Unlike a normal Deployment where Pods are interchangeable, StatefulSet Pods receive stable identities:

```text
mysql-0
mysql-1
mysql-2
```

With `volumeClaimTemplates`, each Pod can receive its own PVC.

```text
StatefulSet
     │
     ├── mysql-0
     │      └── PVC
     │
     └── mysql-1
            └── PVC
```

---

# 📄 StatefulSet Example

Example:

```yaml
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: mysql
  namespace: dev

spec:
  serviceName: mysql
  replicas: 2

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
        - name: mysql
          image: mysql:8

          env:
            - name: MYSQL_ROOT_PASSWORD
              value: root123

          ports:
            - containerPort: 3306

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:
    - metadata:
        name: mysql-data

      spec:
        accessModes:
          - ReadWriteOnce

        storageClassName: gp2

        resources:
          requests:
            storage: 10Gi
```

> ⚠️ **Security Note:** The password is intentionally shown because it is part of the lab exercise, but production deployments should use a Kubernetes Secret or an external secret-management solution.

Apply:

```bash
kubectl apply -f mysql-statefulset.yaml
```

Check:

```bash
kubectl get statefulset -n dev
```

Check Pods:

```bash
kubectl get pods -n dev
```

You should see:

```text
mysql-0
mysql-1
```

Check PVCs:

```bash
kubectl get pvc -n dev
```

You should see separate claims associated with the StatefulSet Pods, such as:

```text
mysql-data-mysql-0
mysql-data-mysql-1
```

Conceptually:

```text
                    StatefulSet
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
           mysql-0               mysql-1
              │                     │
              ▼                     ▼
        mysql-data-0          mysql-data-1
              │                     │
              ▼                     ▼
             PV                    PV
              │                     │
              ▼                     ▼
          Storage               Storage
```

---

# 🔄 Scaling StatefulSet

Scale to two replicas:

```bash
kubectl scale statefulset mysql --replicas=2 -n dev
```

Verify:

```bash
kubectl get pods -n dev
```

Check PVCs:

```bash
kubectl get pvc -n dev
```

Each StatefulSet Pod receives its own storage claim.

> 💡 **Important:** Running two MySQL Pods does **not automatically create a functioning MySQL replication or high-availability cluster**. The two Pods have independent storage and database instances unless replication/HA is explicitly configured.

---

# 🧠 Deployment vs. StatefulSet

| Feature                | Deployment              | StatefulSet                             |
| ---------------------- | ----------------------- | --------------------------------------- |
| Pod identity           | Usually interchangeable | Stable identity                         |
| Pod names              | Random/generated suffix | Ordered identity                        |
| Persistent storage     | Can use PVC             | Designed for per-Pod persistent storage |
| `volumeClaimTemplates` | ❌                       | ✅                                       |
| Typical use            | Stateless applications  | Stateful applications                   |
| Example                | Nginx                   | MySQL, PostgreSQL, Kafka                |

A simplified comparison:

```text
Deployment
    │
    ├── Pod
    ├── Pod
    └── Pod
       │
       └── Often interchangeable


StatefulSet
    │
    ├── mysql-0 → PVC-0
    └── mysql-1 → PVC-1
```

---

# 🔧 Useful Storage Commands

## List PVCs

```bash
kubectl get pvc -n dev
```

## Describe PVC

```bash
kubectl describe pvc app-pvc -n dev
```

## List Persistent Volumes

```bash
kubectl get pv
```

## Describe Persistent Volume

```bash
kubectl describe pv <pv-name>
```

## List StorageClasses

```bash
kubectl get storageclass
```

## Describe StorageClass

```bash
kubectl describe storageclass gp2
```

Or:

```bash
kubectl describe storageclass gp3
```

## Show PVC YAML

```bash
kubectl get pvc app-pvc -n dev -o yaml
```

## Show PV YAML

```bash
kubectl get pv <pv-name> -o yaml
```

## Check Pod Volume Mounts

```bash
kubectl describe pod app-with-storage -n dev
```

## Check Filesystem Usage

```bash
kubectl exec app-with-storage -n dev -- df -h
```

---

# 🛡️ Best-Practice Tips

* Use **PVCs** instead of storing important data only inside container filesystems.
* Select a StorageClass appropriate for the workload.
* Prefer modern AWS EBS storage classes such as **gp3** where appropriate.
* Use **EFS** when multiple workloads require shared read/write storage across nodes.
* Understand the access-mode limitations of your CSI driver.
* Use **StatefulSets** for applications requiring stable identities and per-Pod storage.
* Do not assume that two database Pods automatically provide high availability.
* Use Secrets instead of hard-coded database passwords.
* Test volume recovery before relying on persistent storage in production.
* Configure backups independently from the primary database volume.
* Store production backups separately from the source data.
* Consider Amazon S3 or another durable backup target for database backups.
* Monitor storage capacity and filesystem utilization.
* Use PVC expansion only when the StorageClass and CSI driver support it.
* Verify `allowVolumeExpansion` before attempting to increase storage.
* Use `kubectl describe` and Kubernetes Events when storage provisioning fails.
* Avoid deleting PVCs unless you understand the associated PV reclaim policy and data-loss implications.

---

# 🚨 Troubleshooting

## PVC Stuck in `Pending`

Check:

```bash
kubectl get pvc -n dev
```

Then:

```bash
kubectl describe pvc app-pvc -n dev
```

Review:

```text
Events
```

Check available StorageClasses:

```bash
kubectl get storageclass
```

Confirm the requested StorageClass exists:

```bash
kubectl get storageclass gp2
```

---

## Pod Cannot Mount PVC

Check:

```bash
kubectl describe pod app-with-storage -n dev
```

Look at:

```text
Events
```

Also inspect:

```bash
kubectl describe pvc app-pvc -n dev
```

Potential causes include:

* PVC not bound.
* Incorrect StorageClass.
* CSI driver issue.
* Access mode incompatibility.
* Storage topology constraints.
* Volume attachment problems.

---

## PVC Is Bound but Pod Cannot Start

Check:

```bash
kubectl get pvc -n dev
kubectl get pv
kubectl get pods -n dev
```

Then:

```bash
kubectl describe pod <pod-name> -n dev
```

Look for volume-related events.

---

## Data Is Missing After Pod Recreation

Verify the Pod is using the expected PVC:

```bash
kubectl describe pod app-with-storage -n dev
```

Confirm:

```text
Volumes
PersistentVolumeClaim
```

Then inspect the PVC:

```bash
kubectl describe pvc app-pvc -n dev
```

Check whether the PVC still exists:

```bash
kubectl get pvc app-pvc -n dev
```

> ⚠️ **Important:** Persistence across Pod deletion depends on the PVC and underlying volume remaining intact. Deleting a PVC can result in data loss depending on the PV's reclaim policy and storage implementation.

---

# 🧹 Lab Cleanup

Remove the test Pod:

```bash
kubectl delete pod app-with-storage -n dev
```

Remove the PVC:

```bash
kubectl delete pvc app-pvc -n dev
```

Remove the MySQL resources if created:

```bash
kubectl delete pod mysql -n dev
kubectl delete pvc mysql-pvc -n dev
```

Remove the CronJob:

```bash
kubectl delete cronjob mysql-backup -n dev
```

Remove the StatefulSet:

```bash
kubectl delete statefulset mysql -n dev
```

Remove StatefulSet PVCs if they are no longer required:

```bash
kubectl delete pvc -l app=mysql -n dev
```

Verify:

```bash
kubectl get pvc -n dev
kubectl get pv
kubectl get pods -n dev
```

> ⚠️ **Critical:** Before deleting a PVC or PV, verify its reclaim policy and confirm that the data is no longer required. Persistent storage does not mean permanent storage if the underlying volume or claim is intentionally deleted.

---

# 📋 Lab 10 Storage Flow

The complete storage workflow covered in this lab is:

```text
                    Application
                         │
                         ▼
                        Pod
                         │
                  volumeMount: /data
                         │
                         ▼
                       PVC
                         │
                  Storage Request
                         │
                         ▼
                   StorageClass
                         │
               Dynamic Provisioning
                         │
                         ▼
                        PV
                         │
                         ▼
                  Cloud Storage
```

For AWS EKS:

```text
                 Kubernetes
                     │
                     ▼
              StorageClass
                │        │
                ▼        ▼
              EBS       EFS
             gp3/gp2     RWX
                │        │
                ▼        ▼
            Persistent Storage
```

---

# 📊 Key Concepts Summary

### Persistent Volume — PV

> The actual persistent storage resource available to Kubernetes.

### Persistent Volume Claim — PVC

> An application's request for persistent storage.

### StorageClass

> Defines how persistent storage is dynamically provisioned.

### ReadWriteOnce — RWO

> Read-write access associated with a single node at a time, depending on the storage implementation.

### ReadWriteMany — RWX

> Allows multiple nodes to mount a volume for read/write access when supported by the storage backend, such as EFS.

### StatefulSet

> Provides stable Pod identity and persistent storage patterns for stateful applications.

### `volumeClaimTemplates`

> Allows a StatefulSet to automatically create a separate PVC for each Pod.

---

# ✅ Lab 10 Completion Checklist

* [ ] Understand the difference between PV, PVC, and StorageClass.
* [ ] Understand Kubernetes persistent storage architecture.
* [ ] Create a 5Gi PVC.
* [ ] Use the `gp2` StorageClass where available.
* [ ] Understand `Pending` and `Bound` PVC states.
* [ ] Verify dynamically provisioned PVs.
* [ ] Mount a PVC into a Pod.
* [ ] Write data to `/data`.
* [ ] Delete the Pod.
* [ ] Recreate the Pod.
* [ ] Verify that data survives Pod deletion.
* [ ] Deploy MySQL with persistent storage.
* [ ] Store MySQL data under `/var/lib/mysql`.
* [ ] Delete and recreate the MySQL Pod.
* [ ] Verify database persistence.
* [ ] Understand `ReadWriteOnce`.
* [ ] Understand `ReadWriteMany`.
* [ ] Explore AWS EFS for shared storage.
* [ ] Inspect PVC and PV details.
* [ ] Understand volume topology and node attachment.
* [ ] Create a database backup CronJob.
* [ ] Understand why production backups should be stored separately.
* [ ] Expand a PVC from 5Gi to 10Gi where supported.
* [ ] Verify `allowVolumeExpansion`.
* [ ] Understand StatefulSets.
* [ ] Create a MySQL StatefulSet.
* [ ] Use `volumeClaimTemplates`.
* [ ] Scale the StatefulSet to two replicas.
* [ ] Verify that each StatefulSet Pod receives its own PVC.
* [ ] Understand that multiple MySQL Pods do not automatically provide database replication or HA.
* [ ] Clean up test storage resources safely.
