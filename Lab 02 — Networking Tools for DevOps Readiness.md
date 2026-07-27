# Lab 02 — Networking Tools for DevOps Readiness

## 🎯 Objective

Master the **core networking commands** that DevOps engineers use daily for **network troubleshooting, connectivity testing, DNS diagnostics, port verification, and network monitoring**.

---

## 🔧 Key Commands

The following commands are commonly used by DevOps engineers to troubleshoot network connectivity and identify potential network-related issues.

### Connectivity

```bash
# Check connectivity
ping google.com -c 4

traceroute google.com
```

### DNS Lookup

```bash
# DNS lookup
nslookup google.com

dig google.com
```

### Port Connectivity

```bash
# Port connectivity
telnet google.com 80

nc -zv google.com 443
```

### Network Interfaces

```bash
# Network interfaces
ip addr show

ifconfig
```

### Active Connections

```bash
# Active connections
netstat -tuln

ss -tuln
```

### Firewall Rules

```bash
# Firewall rules
sudo iptables -L -n
```

> 💡 **Tip:** Become comfortable with both older and newer networking tools. For example, `netstat` is widely encountered in existing environments, while `ss` is commonly preferred on modern Linux systems.

---

# 🟢 Task Set 1 — Guided

Follow the tasks step by step and observe the output returned by each command.

| Task     | What to do                                                       |
| -------- | ---------------------------------------------------------------- |
| **T1.1** | SSH into EC2. Run: `ping 8.8.8.8 -c 4`. Observe TTL and latency. |
| **T1.2** | Run: `netstat -tuln \| grep LISTEN`. Note all listening ports.   |
| **T1.3** | Run: `curl -I https://google.com`. Read HTTP response headers.   |

### T1.1 — Test Connectivity

SSH into your EC2 instance and run:

```bash
ping 8.8.8.8 -c 4
```

Observe the following values in the output:

* **TTL (Time to Live)**
* **Latency / response time**
* Number of packets transmitted
* Number of packets received
* Packet loss

---

### T1.2 — Identify Listening Ports

Run:

```bash
netstat -tuln | grep LISTEN
```

Review the output and note **all listening ports**.

Pay attention to:

* Local addresses
* Port numbers
* Listening state

---

### T1.3 — Inspect HTTP Response Headers

Run:

```bash
curl -I https://google.com
```

Read the returned **HTTP response headers**.

Look for information such as:

* HTTP status code
* Server information
* Content type
* Redirect information
* Other response headers

---

# 🟡 Task Set 2 — Practice

Apply the networking commands independently and use the results to troubleshoot and understand network behavior.

| Task     | What to do                                                                 |
| -------- | -------------------------------------------------------------------------- |
| **T2.1** | Use `traceroute google.com` — count hops. Which hop shows highest latency? |
| **T2.2** | Run `ss -tuln`. Find which port SSH is running on. Why is it important?    |
| **T2.3** | Test AWS EC2 SG: add port 8080 open, then `nc -zv 8080` from laptop.       |

### T2.1 — Analyze Network Hops

Run:

```bash
traceroute google.com
```

Then:

1. Count the number of hops.
2. Review the latency reported for each hop.
3. Identify which hop shows the **highest latency**.

> 💡 **Tip:** Network latency can vary between individual probes. A single high-latency response does not necessarily indicate a persistent network problem.

---

### T2.2 — Identify the SSH Port

Run:

```bash
ss -tuln
```

Find the port on which **SSH** is running.

Then consider:

> **Why is identifying the SSH port important for DevOps engineers?**

Consider its importance for:

* Remote administration
* Security Group configuration
* Firewall configuration
* Troubleshooting SSH connectivity
* Securing exposed services

---

### T2.3 — Test an AWS EC2 Security Group

Configure the EC2 **Security Group (SG)** to allow inbound traffic on port:

```text
8080
```

Then test connectivity from your laptop using `nc`.

> ⚠️ **Important:** The original task specifies `nc -zv 8080` from the laptop. In practice, the command must include the EC2 instance's hostname or IP address, for example:

```bash
nc -zv <EC2-IP-or-DNS> 8080
```

Verify whether the port is reachable after updating the Security Group.

---

# 🔴 Task Set 3 — Challenge

Use your networking knowledge to complete practical troubleshooting and automation scenarios.

| Task     | What to do                                                                         |
| -------- | ---------------------------------------------------------------------------------- |
| **T3.1** | Script: ping 5 servers in a loop, log UP/DOWN status to `/tmp/ping_report.txt`.    |
| **T3.2** | Use `nmap -p 22,80,443` to scan an EC2 instance for open ports.                    |
| **T3.3** | Simulate connectivity loss: block port 80 with iptables, test with curl, then fix. |

### T3.1 — Build a Server Connectivity Check

Create a script that:

1. Pings **5 servers** in a loop.
2. Determines whether each server is **UP** or **DOWN**.
3. Logs the status to:

```text
/tmp/ping_report.txt
```

Your final report should provide a clear record of the connectivity status of all five servers.

---

### T3.2 — Scan an EC2 Instance for Open Ports

Use `nmap` to scan an EC2 instance for the following ports:

```text
22
80
443
```

Command:

```bash
nmap -p 22,80,443 <EC2-IP-or-DNS>
```

Review the results and identify which ports are:

* Open
* Closed
* Filtered

> ⚠️ **Security:** Only scan systems and IP addresses that you own or have explicit authorization to test.

---

### T3.3 — Simulate Connectivity Loss

Simulate a connectivity problem by blocking **port 80** using `iptables`.

Then:

1. Block port 80.
2. Test connectivity using `curl`.
3. Observe the result.
4. Identify the connectivity failure.
5. Fix the firewall configuration.
6. Test again with `curl`.
7. Confirm that connectivity has been restored.

Example firewall inspection command:

```bash
sudo iptables -L -n
```

> ⚠️ **Caution:** Firewall changes can interrupt network access, including SSH access. Perform this exercise only on a test EC2 instance and ensure you have a recovery method before modifying firewall rules.

---

## 💡 Best-Practice Tips

When troubleshooting network issues, follow a structured approach:

1. **Check basic connectivity** with `ping`.
2. **Trace the network path** with `traceroute`.
3. **Verify DNS resolution** using `nslookup` or `dig`.
4. **Check listening services** using `ss` or `netstat`.
5. **Test specific ports** with `nc` or `telnet`.
6. **Inspect HTTP behavior** using `curl`.
7. **Check firewall rules** using `iptables`.
8. **Validate AWS Security Group rules** when working with EC2.
9. **Use `nmap` only on authorized systems.**
10. **Document findings** so troubleshooting can be repeated and reviewed.

---

## ✅ Lab 02 Completion Checklist

* [ ] Understand basic network connectivity testing.
* [ ] Run `ping` and interpret TTL and latency.
* [ ] Use `traceroute` to inspect network hops.
* [ ] Perform DNS lookups using `nslookup` and `dig`.
* [ ] Identify listening ports using `netstat` and `ss`.
* [ ] Test port connectivity using `nc` and `telnet`.
* [ ] Inspect HTTP response headers using `curl`.
* [ ] Understand AWS EC2 Security Group port access.
* [ ] Create a server connectivity monitoring script.
* [ ] Scan authorized EC2 ports using `nmap`.
* [ ] Understand basic `iptables` troubleshooting.
* [ ] Restore connectivity after a simulated firewall block.
