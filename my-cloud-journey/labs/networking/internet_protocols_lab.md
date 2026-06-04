# Internet Protocols — Public and Private IP Addresses

> **Lab Type:** AWS Cloud Support Scenario  
> **Duration:** ~1 hour  
> **Role:** Cloud Support Engineer at AWS  
> **Core Concepts:** Public vs Private IP Addresses, VPC CIDRs, SSH Connectivity, OSI Model Mapping

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [The Customer Scenario](#the-customer-scenario)
3. [Architecture](#architecture)
4. [Key Concepts Before You Start](#key-concepts-before-you-start)
5. [Task 1 — Investigate the Customer's Environment](#task-1--investigate-the-customers-environment)
6. [Task 2 — Connect via SSH to Prove the Issue](#task-2--connect-via-ssh-to-prove-the-issue)
7. [Task 3 — Respond to the Customer](#task-3--respond-to-the-customer)
8. [The Public CIDR Question Explained](#the-public-cidr-question-explained)
9. [Key Takeaways](#key-takeaways)

---

## Lab Overview

In this lab, I stepped into the role of a **Cloud Support Engineer at AWS**, responding to a real-world-style support ticket from a Fortune 500 customer. The customer had two EC2 instances in the same VPC and subnet with identical AWS configurations — yet one could reach the internet and the other could not.

The lab required me to:
- Replicate and investigate the customer's environment
- Apply OSI model thinking to troubleshoot from the bottom up
- Use SSH to prove the root cause
- Advise the customer on a secondary networking question about VPC CIDR ranges

---

## The Customer Scenario

**Support Ticket — From: Jess (Cloud Admin)**

> *"We currently have one VPC with a CIDR range of 10.0.0.0/16. In this VPC, we have two EC2 instances: instance A and instance B. Even though both are in the same subnet and have the same configurations with AWS resources, instance A cannot reach the internet, and instance B can. I think it has something to do with the EC2 instances, but I'm not sure. I also had a question about using a public range of IP addresses such as 12.0.0.0/16 for a VPC that I would like to launch. Would that cause any issues?"*

**Two problems to solve:**
1. Why can Instance B reach the internet but Instance A cannot?
2. Is it safe to use a public IP range like `12.0.0.0/16` as a VPC CIDR?

---

## Architecture

```
                        Internet
                           │
                           ▼
                  Internet Gateway (IGW)
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │              VPC                     │
        │         CIDR: 10.0.0.0/16            │
        │                                      │
        │   ┌──────────────────────────────┐   │
        │   │        Public Subnet         │   │
        │   │                              │   │
        │   │  [Instance A]  [Instance B]  │   │
        │   │  Private IP ✅  Private IP ✅ │   │
        │   │  Public IP ❌   Public IP ✅  │   │
        │   └──────────────────────────────┘   │
        └──────────────────────────────────────┘
```

Both instances sit in the **same subnet**, are attached to the **same IGW**, and share the **same security group configuration**. The only difference is that **Instance A has no public IP address**.

---

## Key Concepts Before You Start

### Private vs Public IP Addresses

| Type | Range Examples | Reachable From |
|------|----------------|----------------|
| **Private** | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Inside the VPC only |
| **Public** | Everything else (assigned by AWS or your ISP) | The internet |

- **Private IPs** are used for internal VPC communication — instance to instance, instance to database, etc.
- **Public IPs** are required for any instance that needs to send or receive traffic to/from the internet.
- Even with a perfectly configured route table and internet gateway, **an instance without a public IP cannot be reached from outside the VPC**.

---

### OSI Model Mapped to AWS Infrastructure

This lab applies a **bottom-up troubleshooting approach** based on the OSI model. Starting at the instance level (Layer 5) and working upward helps isolate where a problem lives.

| OSI Layer | Layer Name | AWS Equivalent |
|-----------|------------|----------------|
| Layer 7 | Application | Application |
| Layer 6 | Presentation | Web/App Servers |
| Layer 5 | Session | EC2 Instances |
| Layer 4 | Transport | Security Groups, NACLs |
| Layer 3 | Network | Route Tables, IGW, Subnets |
| Layer 2 | Data Link | Route Tables, IGW, Subnets |
| Layer 1 | Physical | Regions, Availability Zones |

> In this lab, the issue lived at **Layer 5 (Session / EC2 instance level)** — the instance itself was missing a public IP, so no session with the internet could ever be established regardless of what was configured above it.

---

## Task 1 — Investigate the Customer's Environment

### Steps

1. Navigate to **EC2 → Instances** in the AWS Management Console
2. Select **Instance A**, open the **Networking tab**, and record:
   - Public IPv4 address
   - Private IPv4 address
3. Deselect Instance A, then select **Instance B** and do the same
4. Compare the two

### What I Found

| | Instance A | Instance B |
|--|--|--|
| **Private IPv4** | ✅ Assigned (e.g. `10.0.1.x`) | ✅ Assigned (e.g. `10.0.1.y`) |
| **Public IPv4** | ❌ **Not assigned** | ✅ Assigned (e.g. `54.x.x.x`) |

**Root cause identified:** Instance A was launched **without a public IP address**. Everything else — the route table, internet gateway, security group, and subnet — was correctly configured. The single missing piece was the public IP.

---

📸 **Screenshot:**
> _Insert screenshot: EC2 Instances list showing both Instance A and Instance B._

---

📸 **Screenshot:**
> _Insert screenshot: Instance A — Networking tab showing Private IPv4 only, no Public IPv4 address._

---

📸 **Screenshot:**
> _Insert screenshot: Instance B — Networking tab showing both a Private IPv4 and a Public IPv4 address._

---

## Task 2 — Connect via SSH to Prove the Issue

### Purpose

SSH connection attempts were used to **empirically confirm** the root cause. If Instance A truly has no public IP, it should be unreachable from outside the VPC.

---

### For macOS / Linux Users

```bash
# Step 1 — Navigate to where your .pem key was downloaded
cd ~/Downloads

# Step 2 — Set correct permissions on the key file
chmod 400 labsuser.pem

# Step 3 — Try connecting to Instance A (private IP only)
ssh -i labsuser.pem ec2-user@<instance-A-private-IP>
# ❌ This will time out or fail — private IPs are not reachable from outside the VPC

# Step 4 — Connect to Instance B (has a public IP)
ssh -i labsuser.pem ec2-user@<instance-B-public-IP>
# ✅ This will succeed
```

### For Windows Users

1. Download the `.ppk` file from the lab credentials panel
2. Open **PuTTY**
3. Under **Session → Host Name**, enter the public IP of Instance B
4. Under **Connection → SSH → Auth**, browse to your `.ppk` file
5. Click **Open** — connection to Instance B succeeds
6. Attempt the same with Instance A's private IP — connection fails

---

### Results

| Instance | IP Used | SSH Result | Reason |
|----------|---------|------------|--------|
| Instance A | Private IP | ❌ Failed | Private IPs not routable from the internet |
| Instance B | Public IP | ✅ Success | Public IP allows inbound connection through IGW |

---

📸 **Screenshot:**
> _Insert screenshot: Terminal showing failed SSH attempt to Instance A._

---

📸 **Screenshot:**
> _Insert screenshot: Terminal showing successful SSH connection to Instance B (ec2-user prompt visible)._

---

### How to Fix Instance A

To restore internet connectivity for Instance A, either:

**Option 1 — Assign an Elastic IP (no relaunch needed):**
1. Go to **EC2 → Elastic IPs → Allocate Elastic IP**
2. Click **Associate Elastic IP**
3. Select Instance A and confirm

**Option 2 — Relaunch with Auto-assign public IP enabled:**
1. When launching the instance, under **Network Settings**
2. Set **Auto-assign public IP** to `Enable`

---

## Task 3 — Respond to the Customer

### My Response to Jess

> *"Hi Jess,*
>
> *After investigating your environment, I found the reason Instance A cannot reach the internet: it was not assigned a public IP address. Instance B has a public IP, which is why it can connect. The rest of your architecture — the internet gateway, route table, and security groups — is all correctly configured.*
>
> *To fix Instance A, you can either assign an Elastic IP address to it, or relaunch it with 'Auto-assign public IP' enabled in the network settings.*
>
> *Regarding your question about using 12.0.0.0/16 as a VPC CIDR — I would advise against this. Please see my explanation below.*
>
> *Best regards,*
> *Cloud Support Engineer"*

---

## The Public CIDR Question Explained

Jess asked whether using a **public IP range** like `12.0.0.0/16` as her VPC CIDR would cause any issues.

### Short Answer: Yes — it will cause problems.

### Why

`12.0.0.0/16` is a **publicly routable** IP range registered to **AT&T**. While AWS technically allows you to assign it as a VPC CIDR, doing so creates a serious conflict:

> When an instance in your VPC tries to reach a server at `12.x.x.x` on the internet, the traffic stays inside your VPC (because your route table sees it as local). The response never arrives. Or worse — traffic that should go to AT&T's infrastructure gets misrouted entirely.

### Recommended VPC CIDR Ranges (RFC 1918 — Private)

| Range | Available Addresses |
|-------|-------------------|
| `10.0.0.0/8` | ~16 million |
| `172.16.0.0/12` | ~1 million |
| `192.168.0.0/16` | ~65,000 |

These ranges are **reserved for private use** and will never conflict with publicly routable internet addresses.

### Comparison

| | Private CIDR (e.g. `10.0.0.0/16`) | Public CIDR (e.g. `12.0.0.0/16`) |
|--|--|--|
| **RFC 1918 compliant** | ✅ Yes | ❌ No |
| **Risk of IP conflicts** | Very low | High |
| **Internet routing issues** | None | Traffic to `12.x.x.x` trapped inside VPC |
| **AWS recommendation** | ✅ Recommended | ❌ Not recommended |

---

📸 **Screenshot:**
> _Insert screenshot: VPC console showing the customer's VPC with CIDR 10.0.0.0/16._

---

## Key Takeaways

### 1. Public IP = Internet Access
An EC2 instance **must have a public IP** (or Elastic IP) to be reachable from or to reach the internet. A working route table and internet gateway are necessary but not sufficient on their own.

```
Private IP only  →  Reachable within VPC only
Public IP        →  Reachable from the internet (through IGW)
```

### 2. Bottom-Up Troubleshooting Works
By starting at the instance level (OSI Layer 5) rather than the route table or IGW, the root cause was found immediately. Always check the basics first — is the resource itself configured correctly before moving up the stack.

### 3. Always Use RFC 1918 Ranges for VPCs
Using public IP ranges as VPC CIDRs creates routing conflicts that are difficult to diagnose. Stick to `10.x.x.x`, `172.16.x.x`, or `192.168.x.x`.

### 4. Same Subnet ≠ Same Configuration
Two instances can be in the exact same subnet, attached to the same IGW, with identical security groups — and still behave differently if one was launched without a public IP. **Always verify IP assignment at launch time.**

---

> 💡 *Add your screenshots inline after each section. Recommended tools: Snipping Tool (Windows), Screenshot (Mac), or Flameshot (Linux). Save images to an `/assets/` or `/screenshots/` folder in your repo and reference them as `![description](./screenshots/your-image.png)`.*
