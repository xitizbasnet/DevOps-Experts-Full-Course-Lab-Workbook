# Session 14 — Configuration Management — Ansible

## Lab 22 — Ansible Playbooks, Inventory & Modules

> **Level:** Intermediate
> **Platform:** Linux / AWS EC2
> **Tool:** Ansible
> **Focus:** Configuration Management, Automation, Idempotency, Playbooks, Inventory, and Modules

---

## 🎯 Objective

Use **Ansible** for idempotent server configuration to:

* Install packages
* Manage services
* Deploy applications
* Configure multiple servers consistently
* Automate infrastructure configuration

> [!NOTE]
> **Idempotency** means that running the same Ansible playbook multiple times should produce the same desired state without unnecessarily changing resources that are already correctly configured.

---

## 🛠️ Setup & Inventory

### Install Ansible on the Controller EC2

Run the following commands:

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

---

## 📋 Inventory Configuration

Create an inventory file named:

```text
inventory.ini
```

Use the following configuration:

```ini
[webservers]
web01 ansible_host=10.0.1.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/devops-key.pem
web02 ansible_host=10.0.1.11 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/devops-key.pem

[dbservers]
db01 ansible_host=10.0.10.20 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/devops-key.pem

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

### Test Connectivity

Verify that Ansible can communicate with all hosts:

```bash
ansible all -i inventory.ini -m ping
```

Expected result:

```text
web01 | SUCCESS => {
    "ping": "pong"
}

web02 | SUCCESS => {
    "ping": "pong"
}

db01 | SUCCESS => {
    "ping": "pong"
}
```

> [!TIP]
> If a host does not return `pong`, verify SSH connectivity, the private-key path, security-group rules, username, and the host's IP address.

---

# 📘 Playbook Example

Create a playbook named:

```text
site.yml
```

The following playbook configures the web servers.

```yaml
---
- name: Configure Web Servers
  hosts: webservers
  become: true

  vars:
    nginx_version: latest
    app_dir: /var/www/app

  tasks:

    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install nginx
      apt:
        name: nginx
        state: "{{ nginx_version }}"

    - name: Create app directory
      file:
        path: "{{ app_dir }}"
        state: directory
        mode: '0755'

    - name: Deploy app from template
      template:
        src: index.html.j2
        dest: "{{ app_dir }}/index.html"
      notify: Reload Nginx

    - name: Ensure Nginx running
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:

    - name: Reload Nginx
      service:
        name: nginx
        state: reloaded
```

---

## ▶️ Run the Playbook

Run the playbook normally:

```bash
ansible-playbook -i inventory.ini site.yml
```

### Dry Run

Before making changes, perform a check:

```bash
ansible-playbook -i inventory.ini site.yml --check
```

### Show Changes

Use `--diff` to display configuration differences:

```bash
ansible-playbook -i inventory.ini site.yml --diff
```

### Verbose Output

Run the playbook with verbose output:

```bash
ansible-playbook -i inventory.ini site.yml -v
```

### Run Against One Host

To limit execution to `web01`:

```bash
ansible-playbook -i inventory.ini site.yml --limit web01
```

> [!IMPORTANT]
> Use `--check` before applying significant configuration changes in production environments.

---

# 🧭 Task Set 1 — Guided

| Task     | What to Do                                                                                                        |
| -------- | ----------------------------------------------------------------------------------------------------------------- |
| **T1.1** | Set up passwordless SSH to 2 EC2s. Run `ansible all -i inventory.ini -m ping` — all hosts should return **PONG**. |
| **T1.2** | Apply `site.yml`. Verify that Nginx is running on both web servers using `curl`.                                  |
| **T1.3** | Run with `--check` first (dry run). No changes should be made. Then run the playbook for real.                    |

### T1.1 — Test Ansible Connectivity

```bash
ansible all -i inventory.ini -m ping
```

Expected outcome:

```text
PONG
```

### T1.2 — Verify Nginx

After applying the playbook, verify Nginx from the configured web servers:

```bash
curl http://10.0.1.10
curl http://10.0.1.11
```

### T1.3 — Perform a Dry Run

```bash
ansible-playbook -i inventory.ini site.yml --check
```

Then execute the actual configuration:

```bash
ansible-playbook -i inventory.ini site.yml
```

---

# 🛠️ Task Set 2 — Practice

## T2.1 — Install `htop` Using an Ad Hoc Command

Install `htop` on all web servers:

```bash
ansible webservers -m apt -a 'name=htop' --become
```

---

## T2.2 — Use Group Variables

Create:

```text
group_vars/webservers.yml
```

Move the web-server variables from the playbook into this file.

The objective is to remove the variables from the playbook while producing the **same result**.

Recommended structure:

```text
ansible-project/
├── inventory.ini
├── site.yml
├── group_vars/
│   └── webservers.yml
└── templates/
    └── index.html.j2
```

Run the playbook again:

```bash
ansible-playbook -i inventory.ini site.yml
```

> [!NOTE]
> This exercise demonstrates how Ansible separates **configuration data** from **automation logic**.

---

## T2.3 — Create a Database Playbook

Write a DB playbook that:

1. Installs MySQL.
2. Sets the root password.
3. Creates a database named:

```text
devops
```

---

# 🚀 Task Set 3 — Challenge

## T3.1 — Dynamic AWS Inventory

Replace the static inventory file with Ansible's AWS EC2 dynamic inventory.

Use the `aws_ec2` plugin to automatically discover EC2 instances based on their tags instead of maintaining a static inventory.

### Goal

Instead of manually maintaining:

```ini
web01 ansible_host=10.0.1.10
web02 ansible_host=10.0.1.11
```

Ansible should automatically discover the EC2 instances.

> [!TIP]
> Use EC2 tags such as `Environment=Dev` and `Role=Web` to organize dynamic inventory groups.

---

## T3.2 — Protect Credentials with Ansible Vault

Use **Ansible Vault** to encrypt the database password in a variables file.

Run the playbook with:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

### Objective

Ensure sensitive credentials are not stored in plaintext inside your Git repository.

> [!WARNING]
> Never commit plaintext database passwords, API keys, SSH private keys, or other credentials to GitHub.

---

## T3.3 — Ansible + GitHub Actions

Create a CI/CD workflow that:

1. Uses Terraform to provision a fresh EC2 instance.
2. Obtains the newly provisioned infrastructure details.
3. Runs Ansible against the new EC2 instance.
4. Installs and configures the required software.
5. Deploys the application automatically.

### Target Automation Flow

```text
GitHub Repository
       │
       ▼
GitHub Actions
       │
       ├── Terraform
       │      │
       │      ▼
       │   AWS EC2
       │
       ▼
   Ansible
       │
       ├── Install Packages
       ├── Configure Services
       ├── Deploy Application
       └── Verify Configuration
```

---

# 🔍 Verification Checklist

After completing the lab, verify the following:

* [ ] Ansible is installed on the controller EC2.
* [ ] `ansible --version` works successfully.
* [ ] `inventory.ini` contains the required hosts.
* [ ] SSH connectivity works without interactive password entry.
* [ ] `ansible all -i inventory.ini -m ping` returns **PONG**.
* [ ] Nginx is installed on the web servers.
* [ ] Nginx is running and enabled.
* [ ] The application directory `/var/www/app` exists.
* [ ] The application template is deployed.
* [ ] `--check` works before applying configuration.
* [ ] `--diff` displays configuration differences.
* [ ] `--limit web01` limits execution to the selected server.
* [ ] `group_vars/webservers.yml` is used for web-server variables.
* [ ] A database playbook is created.
* [ ] Dynamic AWS EC2 inventory is configured.
* [ ] Sensitive credentials are protected with Ansible Vault.
* [ ] GitHub Actions can trigger Ansible after Terraform provisioning.

---
