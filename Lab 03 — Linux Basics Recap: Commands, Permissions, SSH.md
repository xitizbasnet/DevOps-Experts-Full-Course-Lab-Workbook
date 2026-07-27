# Lab 03 — Linux Basics Recap: Commands, Permissions, SSH

## 🎯 Objective

Recap essential Linux skills, including:

* File operations
* File and directory permissions
* User and group management
* SSH access
* Process management
* Service management

---

# 🔧 Essential Commands

## 📁 File & Directory Operations

### Long Listing with Hidden Files

```bash
# File & Directory
ls -la /etc | head -20
```

Displays a long listing of `/etc`, including hidden files, and limits the output to the first 20 entries.

### Find All Log Files

```bash
find /var/log -name "*.log"
```

Finds all files with the `.log` extension under `/var/log`.

### Search Logs for Errors

```bash
grep -r "ERROR" /var/log/
```

Recursively searches log files under `/var/log/` for the text:

```text
ERROR
```

### Follow a Live Log

```bash
tail -f /var/log/syslog
```

Follows the log file and displays new entries as they are added.

---

# 🔐 Permissions

Linux permissions use the following values:

```text
rwx = 4 + 2 + 1
```

Where:

| Permission    | Value |
| ------------- | ----: |
| `r` — Read    |     4 |
| `w` — Write   |     2 |
| `x` — Execute |     1 |

### Set Script Permissions

```bash
chmod 755 script.sh
```

Result:

```text
Owner: rwx
Group: r-x
Other: r-x
```

### Protect an SSH Private Key

```bash
chmod 600 ~/.ssh/id_rsa
```

Result:

```text
Owner: rw
Group: ---
Other: ---
```

This gives the owner read/write access only.

### Change File Ownership

```bash
chown ubuntu:ubuntu /opt/app
```

Changes the owner and group of `/opt/app` to:

```text
ubuntu:ubuntu
```

---

# 👥 Users & Groups

### Create a User

```bash
sudo adduser devops
```

Creates a new user named `devops`.

### Add User to the Sudo Group

```bash
sudo usermod -aG sudo devops
```

Adds `devops` to the `sudo` group.

### Switch User

```bash
sudo su - devops
```

Switches to the `devops` user.

### Display Current User Information

```bash
whoami && id
```

Displays information about the current user and their identity/group memberships.

---

# ⚙️ Process Management

### Find an Nginx Process

```bash
ps aux | grep nginx
```

Searches the running process list for processes associated with Nginx.

### Interactive Process Viewer

```bash
top
```

Opens the interactive Linux process viewer.

### Advanced Process Viewer

```bash
htop
```

Provides an enhanced interactive process viewer.

### Force-Kill a Process

```bash
kill -9 <PID>
```

Forcefully terminates the process identified by `<PID>`.

> ⚠️ **Caution:** `kill -9` immediately terminates a process and does not allow it to perform normal cleanup. Prefer a normal termination signal when possible.

### Check Service Status

```bash
systemctl status nginx
```

Displays the current status of the Nginx service.

---

# 🟢 Task Set 1 — Guided

Follow the tasks step by step and observe the output.

| Task     | What to do                                                                |
| -------- | ------------------------------------------------------------------------- |
| **T1.1** | SSH into EC2: `chmod 400 key.pem && ssh -i key.pem ubuntu@`               |
| **T1.2** | Run `ls -la /home`, `cat /etc/passwd`, and `df -h`. Understand outputs.   |
| **T1.3** | Create file: `echo 'Hello DevOps' > ~/test.txt`. Read it back with `cat`. |

### T1.1 — SSH into EC2

First, set appropriate permissions on the private key:

```bash
chmod 400 key.pem
```

Then connect to the EC2 instance:

```bash
ssh -i key.pem ubuntu@<EC2-IP-or-DNS>
```

> 💡 **Note:** Replace `<EC2-IP-or-DNS>` with the public IP address or DNS hostname of your EC2 instance.

---

### T1.2 — Explore the Linux System

Run:

```bash
ls -la /home
```

Then:

```bash
cat /etc/passwd
```

And:

```bash
df -h
```

Understand what information each command provides.

Pay attention to:

* `/home` directory contents
* Linux user account entries
* Disk space usage
* Filesystem capacity
* Used and available storage

---

### T1.3 — Create and Read a File

Create a file containing `Hello DevOps`:

```bash
echo 'Hello DevOps' > ~/test.txt
```

Read the contents back:

```bash
cat ~/test.txt
```

Expected content:

```text
Hello DevOps
```

---

# 🟡 Task Set 2 — Practice

Apply the Linux concepts independently.

| Task     | What to do                                                                |
| -------- | ------------------------------------------------------------------------- |
| **T2.1** | Create user `appuser`, set password, add to sudo group, SSH as that user. |
| **T2.2** | Create `/opt/app` dir, chown to `appuser`, verify with `ls -la /opt/`     |
| **T2.3** | Find all files larger than 10MB: `find / -size +10M -type f 2>/dev/null`  |

### T2.1 — Create and Configure `appuser`

Create the user:

```bash
sudo adduser appuser
```

Set a password when prompted.

Add the user to the sudo group:

```bash
sudo usermod -aG sudo appuser
```

Verify the user's group membership:

```bash
id appuser
```

Then attempt to SSH as the new user.

> ⚠️ **Important:** SSH access for `appuser` also requires an appropriate SSH authentication configuration, such as an authorized public key. Creating a Linux user and password does not automatically configure SSH key-based access.

---

### T2.2 — Create and Assign Ownership

Create the application directory:

```bash
sudo mkdir -p /opt/app
```

Change ownership to `appuser`:

```bash
sudo chown appuser:appuser /opt/app
```

Verify the ownership:

```bash
ls -la /opt/
```

Confirm that `/opt/app` belongs to:

```text
appuser:appuser
```

---

### T2.3 — Find Files Larger Than 10 MB

Run:

```bash
find / -size +10M -type f 2>/dev/null
```

This searches the filesystem for files larger than **10 MB** while suppressing permission-related error messages.

> 💡 **Tip:** Searching from `/` can take some time and may produce a large amount of output.

---

# 🔴 Task Set 3 — Challenge

Apply your Linux administration knowledge to practical DevOps scenarios.

| Task     | What to do                                                                                  |
| -------- | ------------------------------------------------------------------------------------------- |
| **T3.1** | Write a system health script: shows CPU%, memory%, disk%, top 3 processes.                  |
| **T3.2** | Implement log rotation manually: archive `/var/log/syslog`, gzip, delete originals >7 days. |
| **T3.3** | Harden SSH: disable root login, change port to 2222, reload sshd, reconnect on new port.    |

---

## T3.1 — Create a System Health Script

Write a Linux system health script that reports:

* CPU usage percentage
* Memory usage percentage
* Disk usage percentage
* Top 3 processes

The script should provide a clear summary that can be used for basic system-health monitoring.

Suggested output structure:

```text
===== System Health Report =====

CPU Usage:
Memory Usage:
Disk Usage:

Top 3 Processes:
1.
2.
3.
```

> 💡 **DevOps Practice:** A script such as this can later be integrated into automated monitoring or scheduled health checks.

---

## T3.2 — Implement Manual Log Rotation

Implement log rotation manually for:

```text
/var/log/syslog
```

The task requires you to:

1. Archive `/var/log/syslog`.
2. Compress the archive using `gzip`.
3. Delete originals older than **7 days**.

Useful commands to investigate include:

```bash
tar
gzip
find
```

> ⚠️ **Caution:** `/var/log/syslog` is an active system log. Do not blindly delete or truncate it on a production server. Perform this exercise on a test environment and understand how the system's logging service handles the file.

> 💡 **Best Practice:** In production Linux environments, use the operating system's `logrotate` mechanism rather than manually managing active system logs.

---

## T3.3 — Harden SSH

Harden the SSH configuration by completing the following:

1. Disable root login.
2. Change the SSH port to `2222`.
3. Reload the SSH service.
4. Reconnect using the new port.

The target SSH port is:

```text
2222
```

After changing the SSH configuration, reconnect using:

```bash
ssh -p 2222 -i key.pem ubuntu@<EC2-IP-or-DNS>
```

> 🚨 **Critical Warning:** Before changing the SSH port, ensure that port `2222` is permitted in the EC2 **Security Group** and the instance firewall. Keep your existing SSH session open until you have successfully tested the new connection.

> ⚠️ **Important:** The SSH service may be named `ssh` or `sshd`, depending on the Linux distribution. Verify the correct service name before reloading it.

---

# 🛡️ SSH Hardening Checklist

Before considering the challenge complete, verify:

* [ ] Root SSH login is disabled.
* [ ] SSH is configured to use port `2222`.
* [ ] EC2 Security Group permits TCP `2222`.
* [ ] Instance firewall permits TCP `2222`, if applicable.
* [ ] SSH configuration is valid.
* [ ] SSH service has been successfully reloaded.
* [ ] Existing SSH session remains available during testing.
* [ ] New SSH connection works on port `2222`.
* [ ] Root login is no longer permitted.

---

# 💡 Best-Practice Tips

Keep the following Linux administration practices in mind:

* Use **least privilege** when assigning user permissions.
* Protect private SSH keys with restrictive permissions.
* Avoid using `kill -9` unless necessary.
* Verify service status after configuration changes.
* Keep an existing SSH session open when changing SSH configuration.
* Validate firewall and Security Group rules before changing SSH ports.
* Avoid manually deleting active production logs.
* Use **`logrotate`** for production log management.
* Regularly monitor CPU, memory, disk, and process utilization.
* Document configuration changes for troubleshooting and auditing.

---

# ✅ Lab 03 Completion Checklist

* [ ] Understand Linux file and directory operations.
* [ ] Use `ls`, `find`, `grep`, and `tail`.
* [ ] Understand Linux `rwx` permissions.
* [ ] Use `chmod` and `chown`.
* [ ] Create and manage Linux users.
* [ ] Add users to the sudo group.
* [ ] Use `whoami` and `id`.
* [ ] Manage processes using `ps`, `top`, and `htop`.
* [ ] Check services using `systemctl`.
* [ ] SSH into an EC2 instance using a private key.
* [ ] Create and read files from the command line.
* [ ] Find files larger than 10 MB.
* [ ] Create a system health script.
* [ ] Understand manual log rotation.
* [ ] Understand SSH hardening.
* [ ] Change SSH to port `2222` safely.
* [ ] Verify SSH connectivity after configuration changes.
