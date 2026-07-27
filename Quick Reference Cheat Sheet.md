# Quick Reference Cheat Sheet

> [!TIP]
> Use this cheat sheet as a quick operational reference when working with Kubernetes, Docker, Terraform, Ansible, Helm, ArgoCD, AWS, Prometheus, and Git.

| Command / Tool                | What it does                                            | When to use                         |
| ----------------------------- | ------------------------------------------------------- | ----------------------------------- |
| `kubectl get pods -A`         | Lists all pods in all namespaces                        | First thing after any deployment    |
| `kubectl describe pod <n>`    | Displays full pod details and events                    | Debugging `CrashLoopBackOff`        |
| `docker system prune -a`      | Removes all unused images and containers                | Low disk space on host              |
| `terraform plan -out=tfplan`  | Saves the Terraform execution plan to a file for review | Before any infrastructure change    |
| `ansible-playbook --check`    | Performs a dry run — no changes are applied             | Testing playbooks safely            |
| `helm diff upgrade`           | Previews Helm upgrade changes                           | Before a production upgrade         |
| `argocd app sync --prune`     | Forces synchronization and removes orphaned resources   | Manual synchronization with cleanup |
| `aws sts get-caller-identity` | Verifies the current AWS identity                       | Debugging IAM/role issues           |
| `kubectl rollout undo`        | Rolls back to the previous deployment                   | Bad deployment recovery             |
| `promtool check rules`        | Validates alert-rule syntax                             | Before applying alerts              |
| `git log --oneline --graph`   | Displays a visual branch/history graph                  | Understanding Git history           |

---

## 🧭 Command Quick Reference

### Kubernetes

```bash
kubectl get pods -A
```

**Use:** First thing after any deployment.

```bash
kubectl describe pod <n>
```

**Use:** Debugging `CrashLoopBackOff` and reviewing pod events.

```bash
kubectl rollout undo
```

**Use:** Recovering from a bad deployment.

---

### Docker

```bash
docker system prune -a
```

**Use:** When the host has low disk space.

> [!WARNING]
> `docker system prune -a` can remove unused Docker resources. Review what is no longer needed before running it on an important host.

---

### Terraform

```bash
terraform plan -out=tfplan
```

**Use:** Before any infrastructure change.

This saves the Terraform plan to a file for review.

---

### Ansible

```bash
ansible-playbook --check
```

**Use:** Safely test a playbook without applying changes.

---

### Helm

```bash
helm diff upgrade
```

**Use:** Preview Helm upgrade changes before a production upgrade.

---

### ArgoCD

```bash
argocd app sync --prune
```

**Use:** Manually synchronize an application and remove orphaned resources.

> [!WARNING]
> Use `--prune` carefully because resources managed by the ArgoCD application but no longer defined in Git may be removed.

---

### AWS

```bash
aws sts get-caller-identity
```

**Use:** Verify the AWS identity currently being used.

Useful when troubleshooting:

* IAM permissions
* IAM roles
* AWS profiles
* Authentication issues

---

### Prometheus

```bash
promtool check rules
```

**Use:** Validate alert-rule syntax before applying alert rules.

---

### Git

```bash
git log --oneline --graph
```

**Use:** Understand Git history and visualize branch relationships.

---

# ⚡ Essential Troubleshooting Sequence

When troubleshooting a Kubernetes deployment, a useful starting sequence is:

```bash
kubectl get pods -A
```

Then inspect a problematic pod:

```bash
kubectl describe pod <n>
```

If the deployment is confirmed to be bad and rollback is required:

```bash
kubectl rollout undo
```

For an ArgoCD-managed application, check synchronization and use:

```bash
argocd app sync --prune
```

when a manual sync with cleanup is intentionally required.

---

# 🛠️ Infrastructure Change Workflow

For infrastructure changes, follow this general sequence:

```text
Terraform Code
      ↓
terraform plan -out=tfplan
      ↓
Review Plan
      ↓
Approval
      ↓
Apply Change
```

For Ansible:

```text
Ansible Playbook
      ↓
ansible-playbook --check
      ↓
Review
      ↓
Apply
```

For Helm:

```text
Helm Configuration
      ↓
helm diff upgrade
      ↓
Review
      ↓
Production Upgrade
```

---

# 🔐 AWS Identity Troubleshooting

When you encounter an AWS IAM or role-related problem, start with:

```bash
aws sts get-caller-identity
```

This confirms which AWS identity your current CLI session is using.

---

# 📚 Git History

To quickly understand the current repository history:

```bash
git log --oneline --graph
```

This provides a compact visual representation of commits and branches.

---

# 🎯 Final Quick Reference

```text
Kubernetes     → kubectl get pods -A
Pod Debugging  → kubectl describe pod <n>
Docker Cleanup → docker system prune -a
Terraform      → terraform plan -out=tfplan
Ansible        → ansible-playbook --check
Helm           → helm diff upgrade
ArgoCD         → argocd app sync --prune
AWS Identity   → aws sts get-caller-identity
Rollback       → kubectl rollout undo
Prometheus     → promtool check rules
Git History    → git log --oneline --graph
```

> [!NOTE]
> **All the best on your DevOps journey! Keep building, keep shipping.** 🚀
