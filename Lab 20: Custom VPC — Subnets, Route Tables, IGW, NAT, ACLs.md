# Session 12 — AWS Advanced: VPC, Security & Stateful Services

> **Lab 20:** Custom VPC — Subnets, Route Tables, IGW, NAT, ACLs

---

## 🎯 Objective

Build a **production-grade Amazon VPC** with:

* Public and private subnets
* Internet Gateway (IGW)
* NAT Gateway
* Route tables
* Network Access Control Lists (NACLs)
* Multi-AZ architecture
* Secure connectivity between public and private resources

---

## 🏗️ Architecture

The lab uses the following network architecture:

```text
VPC: 10.0.0.0/16
│
├── Availability Zone: ap-south-1a
│   ├── Public Subnet 1: 10.0.1.0/24
│   └── Private Subnet 1: 10.0.10.0/24
│
└── Availability Zone: ap-south-1b
    ├── Public Subnet 2: 10.0.2.0/24
    └── Private Subnet 2: 10.0.20.0/24
```

### 🌐 Traffic Architecture

```text
                         Internet
                            │
                            ▼
                  ┌──────────────────┐
                  │ Internet Gateway │
                  │      (IGW)       │
                  └────────┬─────────┘
                           │
                    Public Route Table
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
       Public Subnet 1           Public Subnet 2
       10.0.1.0/24               10.0.2.0/24
       ap-south-1a               ap-south-1b
              │
              ▼
        NAT Gateway
              │
              ▼
       Private Route Table
              │
       ┌──────┴──────┐
       ▼             ▼
 Private Subnet 1  Private Subnet 2
 10.0.10.0/24      10.0.20.0/24
 ap-south-1a       ap-south-1b
```

### 📋 Network Components

| Component           | Configuration                                       |
| ------------------- | --------------------------------------------------- |
| VPC                 | `10.0.0.0/16`                                       |
| Public Subnet 1     | `10.0.1.0/24` — `ap-south-1a`                       |
| Public Subnet 2     | `10.0.2.0/24` — `ap-south-1b`                       |
| Private Subnet 1    | `10.0.10.0/24` — `ap-south-1a`                      |
| Private Subnet 2    | `10.0.20.0/24` — `ap-south-1b`                      |
| Internet Gateway    | Public internet connectivity                        |
| Public Route Table  | Routes public traffic through IGW                   |
| NAT Gateway         | Outbound internet access for private subnets        |
| Private Route Table | Routes private outbound traffic through NAT Gateway |
| NACLs               | Subnet-level network access control                 |

> 💡 **Best Practice:** Production VPCs commonly use multiple Availability Zones to improve availability and fault tolerance.

---

# 🖥️ Console Steps

Follow these steps in the **AWS Management Console**.

### 1. Create the VPC

Navigate to:

**AWS Console → VPC → Your VPCs → Create VPC**

Configure:

* **Name:** `devops-vpc`
* **IPv4 CIDR:** `10.0.0.0/16`

---

### 2. Create Four Subnets

Create four subnets across two Availability Zones.

#### Public Subnets

* **Public Subnet 1**

  * CIDR: `10.0.1.0/24`
  * AZ: `ap-south-1a`

* **Public Subnet 2**

  * CIDR: `10.0.2.0/24`
  * AZ: `ap-south-1b`

#### Private Subnets

* **Private Subnet 1**

  * CIDR: `10.0.10.0/24`
  * AZ: `ap-south-1a`

* **Private Subnet 2**

  * CIDR: `10.0.20.0/24`
  * AZ: `ap-south-1b`

---

### 3. Create and Attach an Internet Gateway

Navigate to:

**VPC → Internet Gateways → Create Internet Gateway**

Create the gateway and attach it to:

```text
devops-vpc
```

The Internet Gateway provides internet connectivity for resources in properly configured public subnets.

---

### 4. Configure the Public Route Table

Create:

```text
Public-RT
```

Add the following route:

```text
Destination: 0.0.0.0/0
Target: Internet Gateway
```

Associate the route table with:

```text
Public Subnet 1
Public Subnet 2
```

---

### 5. Create the NAT Gateway

First allocate an **Elastic IP**.

Then navigate to:

**VPC → NAT Gateways → Create NAT Gateway**

Configure:

```text
Subnet: Public Subnet 1
Elastic IP: Allocated Elastic IP
```

The NAT Gateway allows resources in private subnets to initiate outbound internet connections without allowing unsolicited inbound internet connections.

---

### 6. Configure the Private Route Table

Create:

```text
Private-RT
```

Add:

```text
Destination: 0.0.0.0/0
Target: NAT Gateway
```

Associate:

```text
Private Subnet 1
Private Subnet 2
```

---

### 7. Enable Public IP Assignment

Enable **Auto-assign Public IP** on the public subnets.

> ⚠️ **Security Note:** A public IP alone does not make a resource accessible. Security Groups, NACLs, routing, and the resource's own configuration must also permit the traffic.

---

### 8. Test Private Subnet Internet Access

Launch an EC2 instance in a private subnet and test:

```bash
curl google.com
```

Expected behavior:

```text
Private EC2
    │
    ▼
Private Route Table
    │
    ▼
NAT Gateway
    │
    ▼
Internet Gateway
    │
    ▼
Internet
```

The private EC2 instance should be able to make **outbound** internet connections through the NAT Gateway.

It should **not** be directly reachable from the public internet.

---

# 🧪 Task Set 1 — Guided

> **Goal:** Build the VPC architecture manually and understand how public/private networking works.

| Task     | What to do                                                                                                          |
| -------- | ------------------------------------------------------------------------------------------------------------------- |
| **T1.1** | Create VPC and 4 subnets manually in Console. Draw the network diagram as you build.                                |
| **T1.2** | Create IGW and attach. Create NAT GW. Set up route tables with correct associations.                                |
| **T1.3** | Launch EC2 in public subnet (SSH ok). Launch EC2 in private subnet. From public EC2, SSH to private EC2 (jump box). |

### T1.1 — Build the VPC

Create:

```text
VPC
├── Public Subnet 1
├── Public Subnet 2
├── Private Subnet 1
└── Private Subnet 2
```

As you create each component, draw the architecture and document:

* CIDR ranges
* Availability Zones
* Route tables
* Internet Gateway
* NAT Gateway
* Subnet associations

---

### T1.2 — Configure Internet and Routing

Create and configure:

```text
Internet Gateway
       │
       ▼
Public Route Table
       │
       ├── Public Subnet 1
       └── Public Subnet 2
```

Then configure:

```text
Private Subnets
       │
       ▼
Private Route Table
       │
       ▼
NAT Gateway
       │
       ▼
Internet Gateway
```

Verify all route-table associations carefully.

---

### T1.3 — Test Public-to-Private Connectivity

Launch:

* One EC2 instance in a public subnet
* One EC2 instance in a private subnet

Use the public EC2 instance as a **jump box**.

From the public EC2 instance, SSH to the private EC2 instance.

Example:

```bash
ssh -i devops-key.pem ubuntu@<PRIVATE-IP>
```

> 💡 **Best Practice:** In production, consider AWS Systems Manager Session Manager instead of exposing SSH access through a bastion/jump host.

---

# 🔧 Task Set 2 — Practice

> **Goal:** Extend the VPC with security controls, VPC connectivity, and network monitoring.

| Task     | What to do                                                                            |
| -------- | ------------------------------------------------------------------------------------- |
| **T2.1** | Add NACLs: block all inbound TCP port 23 (Telnet) on public subnets.                  |
| **T2.2** | VPC Peering: create second VPC (`10.1.0.0/16`). Peer both VPCs. Ping EC2 in peer VPC. |
| **T2.3** | VPC Flow Logs: enable to S3. Launch traffic. Download logs and analyse connections.   |

### T2.1 — Configure NACLs

Configure the NACL associated with the public subnets to block:

```text
Protocol: TCP
Port: 23
Traffic: Inbound
```

Telnet should not be permitted.

> ⚠️ **Important:** NACLs are **stateless**. If you modify inbound rules, carefully consider the corresponding outbound rules required for return traffic.

---

### T2.2 — VPC Peering

Create a second VPC:

```text
VPC 1: 10.0.0.0/16
VPC 2: 10.1.0.0/16
```

Create a VPC peering connection between them.

Update the appropriate route tables so traffic can flow between the VPCs.

Then test connectivity by pinging an EC2 instance in the peer VPC.

---

### T2.3 — VPC Flow Logs

Enable **VPC Flow Logs** and configure delivery to:

```text
Amazon S3
```

Generate traffic between instances and analyze the resulting logs.

Look for information such as:

* Source IP
* Destination IP
* Source port
* Destination port
* Protocol
* Accepted/rejected traffic
* Traffic volume

---

# 🚀 Task Set 3 — Challenge

> **Goal:** Build a more production-oriented architecture using managed AWS services, security controls, and Infrastructure as Code.

| Task     | What to do                                                                                          |
| -------- | --------------------------------------------------------------------------------------------------- |
| **T3.1** | Deploy RDS MySQL in private subnets (multi-AZ). Connect from EC2 in private subnet only.            |
| **T3.2** | Add WAF + Shield to ALB in this VPC. Configure rate limiting rule (100 req/IP/5min).                |
| **T3.3** | Full Terraform: replicate entire VPC setup above as IaC. `terraform destroy` and reapply from code. |

---

### T3.1 — Deploy RDS MySQL

Deploy an **Amazon RDS MySQL** database into the private subnets.

Architecture:

```text
Internet
   │
   ▼
ALB
   │
   ▼
Application
   │
   ▼
Private EC2
   │
   ▼
RDS MySQL
```

Configure RDS for **Multi-AZ** deployment.

The database should only be accessible from the appropriate private application resources.

> 🔐 **Security Best Practice:** Do not expose the RDS database directly to the internet. Use Security Groups to permit MySQL traffic only from the required application tier.

---

### T3.2 — Add AWS WAF and Shield

Add:

* **AWS WAF**
* **AWS Shield**

to the Application Load Balancer.

Configure a rate-based WAF rule:

```text
100 requests / IP / 5 minutes
```

Test the behavior and verify that excessive requests are handled according to the configured rule.

---

### T3.3 — Rebuild the VPC with Terraform

Recreate the complete VPC architecture using Terraform.

The Terraform implementation should include:

```text
VPC
├── Public Subnet 1
├── Public Subnet 2
├── Private Subnet 1
├── Private Subnet 2
├── Internet Gateway
├── Public Route Table
├── Private Route Table
├── NAT Gateway
├── Elastic IP
├── NACLs
├── Security Groups
└── Required AWS resources
```

Initialize and validate the configuration:

```bash
terraform init
terraform validate
terraform plan
```

Apply the infrastructure:

```bash
terraform apply
```

After testing the infrastructure, destroy it:

```bash
terraform destroy
```

Then reapply the infrastructure from the Terraform code.

> 💡 **Challenge Objective:** The entire environment should be reproducible from code without manually recreating the AWS resources.

---

# ✅ Lab 20 Completion Checklist

Use this checklist to confirm that the lab is complete.

### VPC Foundation

* [ ] VPC `10.0.0.0/16` created
* [ ] Public subnet `10.0.1.0/24` created
* [ ] Public subnet `10.0.2.0/24` created
* [ ] Private subnet `10.0.10.0/24` created
* [ ] Private subnet `10.0.20.0/24` created
* [ ] Resources distributed across `ap-south-1a` and `ap-south-1b`

### Routing

* [ ] Internet Gateway created and attached
* [ ] Public route table configured
* [ ] Public subnets associated
* [ ] NAT Gateway created
* [ ] Elastic IP allocated
* [ ] Private route table configured
* [ ] Private subnets associated

### Security

* [ ] NACL configured
* [ ] Telnet port 23 blocked
* [ ] Public/private access behavior tested
* [ ] Private EC2 has outbound internet through NAT
* [ ] Private EC2 has no direct inbound internet access

### Advanced

* [ ] VPC Peering configured
* [ ] VPC Flow Logs enabled
* [ ] RDS MySQL deployed privately
* [ ] Multi-AZ configuration tested
* [ ] WAF configured
* [ ] Shield configured
* [ ] Rate limiting tested
* [ ] VPC recreated using Terraform
* [ ] `terraform destroy` tested
* [ ] Infrastructure successfully reapplied from code

---

## 💡 Key Takeaways

By completing **Lab 20**, you should understand how to:

1. Design a multi-AZ AWS VPC.
2. Separate public and private workloads.
3. Use an **Internet Gateway** for public connectivity.
4. Use a **NAT Gateway** for private outbound connectivity.
5. Configure route tables and subnet associations.
6. Apply NACLs for subnet-level traffic control.
7. Establish connectivity using VPC Peering.
8. Analyze network traffic using VPC Flow Logs.
9. Deploy stateful services such as **Amazon RDS** securely.
10. Protect ALB workloads with **AWS WAF and Shield**.
11. Reproduce AWS networking infrastructure using **Terraform**.

> **🏁 Lab Outcome:** You have progressed from manually creating AWS networking components to designing, securing, monitoring, and automating a production-oriented VPC architecture.
