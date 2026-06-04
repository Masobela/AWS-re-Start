# Internet Protocols — Static and Dynamic IP Addresses

> **Lab Type:** AWS Cloud Support Scenario  
> **Duration:** ~60 minutes  
> **Role:** Cloud Support Engineer at AWS  
> **Core Concepts:** Dynamic vs Static IP Addresses, Elastic IP (EIP), EC2 Instance Lifecycle

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [The Customer Scenario](#the-customer-scenario)
3. [Architecture](#architecture)
4. [Key Concepts Before You Start](#key-concepts-before-you-start)
5. [Task 1 — Replicate and Investigate the Issue](#task-1--replicate-and-investigate-the-issue)
   - [Step 1: Launch a New EC2 Instance](#step-1-launch-a-new-ec2-instance)
   - [Step 2: Observe the Dynamic IP Behaviour](#step-2-observe-the-dynamic-ip-behaviour)
   - [Step 3: Allocate an Elastic IP (EIP)](#step-3-allocate-an-elastic-ip-eip)
   - [Step 4: Associate the EIP to the Instance](#step-4-associate-the-eip-to-the-instance)
   - [Step 5: Verify the Fix](#step-5-verify-the-fix)
6. [Task 2 — Respond to the Customer](#task-2--respond-to-the-customer)
7. [Key Takeaways](#key-takeaways)

---

## Lab Overview

In this lab I acted as a **Cloud Support Engineer at AWS**, responding to a support ticket from a Fortune 500 customer. Their EC2 instance was assigned a new public IP address every time it was stopped and restarted — breaking all dependent resources that relied on a fixed IP.

The lab required me to:
- Replicate the issue by launching and stopping/starting a fresh EC2 instance
- Observe the difference between dynamic and static IP behaviour
- Fix the issue by allocating and associating an **Elastic IP (EIP)**
- Confirm the fix by stopping and restarting the instance and verifying the IP remained the same

---

## The Customer Scenario

**Support Ticket — From: Bob (Cloud Admin)**

> *"We are having issues with one of our EC2 instances. The IP changes every time we start and stop this instance called Public Instance. This causes everything to break since it needs a static IP address. We are not sure why the IP changes on this instance to a random IP every time. Can you please investigate?"*

**The problem in plain terms:**
- Bob's instance uses a **dynamic public IP** (auto-assigned at launch)
- Every time the instance is **stopped and restarted**, AWS releases the old public IP and assigns a new one
- Any service, DNS record, or config pointing to the old IP immediately breaks
- Bob cannot leave the instance running 24/7 due to cost, so he needs the IP to **persist across stop/start cycles**

---

## Architecture

```
                        Internet
                           │
                           ▼
                  Internet Gateway (IGW)
                           │
                           ▼
        ┌──────────────────────────────────┐
        │             VPC                  │
        │                                  │
        │   ┌──────────────────────────┐   │
        │   │      Public Subnet       │   │
        │   │                          │   │
        │   │   [Public Instance]      │   │
        │   │   Private IP: Fixed ✅   │   │
        │   │   Public IP:  Changes ❌  │   │
        │   └──────────────────────────┘   │
        └──────────────────────────────────┘
```

After applying the fix:

```
                        Internet
                           │
                           ▼
                  Internet Gateway (IGW)
                           │
                           ▼
        ┌──────────────────────────────────┐
        │             VPC                  │
        │                                  │
        │   ┌──────────────────────────┐   │
        │   │      Public Subnet       │   │
        │   │                          │   │
        │   │   [Public Instance]      │   │
        │   │   Private IP: Fixed ✅   │   │
        │   │   Elastic IP:  Fixed ✅  │   │
        │   └──────────────────────────┘   │
        └──────────────────────────────────┘
```

---

## Key Concepts Before You Start

### Dynamic vs Static IP Addresses

| Type | Behaviour | AWS Implementation |
|------|-----------|-------------------|
| **Dynamic** | Changes on every stop/start | Auto-assigned Public IPv4 |
| **Static** | Persists across stop/start cycles | Elastic IP Address (EIP) |

### What Happens to IPs When You Stop an EC2 Instance

| IP Type | On Stop | On Start |
|---------|---------|----------|
| **Private IPv4** | ✅ Retained | ✅ Same address returns |
| **Auto-assigned Public IPv4** | ❌ Released back to AWS pool | ❌ New address assigned |
| **Elastic IP (EIP)** | ✅ Retained | ✅ Same address returns |

> **Why does this happen?**  
> AWS manages a shared pool of public IP addresses. When an instance is stopped, its auto-assigned public IP is returned to that pool so other customers can use it. When the instance starts again, it draws a different IP from the pool — there's no guarantee it gets the same one back.

### What is an Elastic IP (EIP)?

An **Elastic IP** is a static, public IPv4 address that you allocate to your AWS account. Unlike auto-assigned IPs:
- It belongs to **your account** until you explicitly release it
- It **stays attached** to your instance across stop/start cycles
- It can be **remapped** to a different instance if needed (useful for failover)

> ⚠️ **Cost note:** AWS charges for EIPs that are **allocated but not associated** with a running instance. Always release EIPs you're not using to avoid unnecessary charges.

---

## Task 1 — Replicate and Investigate the Issue

### Step 1: Launch a New EC2 Instance

To replicate Bob's environment, I launched a fresh EC2 instance configured the same way as his.

**Navigation:** AWS Console → Search `EC2` → **EC2 Dashboard** → **Launch instances**

**Instance configuration:**

| Setting | Value |
|---------|-------|
| **AMI** | Amazon Linux 2 AMI (HVM) |
| **Instance type** | t3.micro |
| **Network** | Lab VPC |
| **Subnet** | Public Subnet 1 |
| **Auto-assign Public IP** | Enabled |
| **Storage** | Default |
| **Tag — Name** | `test instance` |
| **Security Group** | Linux Instance SG (existing) |
| **Key Pair** | vockey \| RSA |

After launching, I waited for the instance state to show **2/2 checks passed** before proceeding.

---

📸 **Screenshot:**
> _Insert screenshot: EC2 Launch Instance wizard — Step 3 (Configure Instance Details) showing Network set to Lab VPC, Subnet set to Public Subnet 1, and Auto-assign Public IP set to Enable._

---

📸 **Screenshot:**
> _Insert screenshot: EC2 Instances list showing the newly launched "test instance" with Instance State: Running and 2/2 status checks passed._

---

### Step 2: Observe the Dynamic IP Behaviour

With the instance running, I recorded the initial IP addresses, then stopped and restarted it to observe what changed.

**Before stopping — record these values:**
1. Select `test instance` → **Networking tab**
2. Note the **Public IPv4 address** (e.g. `54.210.x.x`)
3. Note the **Private IPv4 address** (e.g. `10.0.1.x`)

**Stop the instance:**
- Top right → **Instance state** → **Stop instance**
- Wait for state: `Stopped`
- Check the **Networking tab** — the Public IPv4 address **disappears**

**Start the instance again:**
- **Instance state** → **Start instance**
- Wait for state: `Running`
- Check the **Networking tab** — a **new Public IPv4 address** has been assigned

---

📸 **Screenshot:**
> _Insert screenshot: Networking tab of "test instance" while running — showing the original Public IPv4 and Private IPv4 addresses._

---

📸 **Screenshot:**
> _Insert screenshot: Networking tab after instance is STOPPED — Public IPv4 field is empty, Private IPv4 is still present._

---

📸 **Screenshot:**
> _Insert screenshot: Networking tab after instance is STARTED again — a NEW Public IPv4 address is shown (different from the original)._

---

**Observations:**

| | Before Stop | After Restart |
|--|--|--|
| **Public IPv4** | `54.210.x.x` | `52.x.x.x` (different) |
| **Private IPv4** | `10.0.1.x` | `10.0.1.x` (same) |

**Conclusion:** The public IP is **dynamic** — it changes on every stop/start cycle. The private IP is **static** — it stays the same. This confirms Bob's issue has been replicated.

---

### Step 3: Allocate an Elastic IP (EIP)

Now that the issue was confirmed, the fix is to allocate an Elastic IP and attach it to the instance.

**Navigation:** EC2 Dashboard → Left navigation → **Network and Security** → **Elastic IPs**

1. Click **Allocate Elastic IP address** (top right)
2. Leave all settings as default
3. Click **Allocate**
4. Note the new EIP address (e.g. `3.92.x.x`)

---

📸 **Screenshot:**
> _Insert screenshot: Elastic IPs page showing no existing EIPs and the "Allocate Elastic IP address" button highlighted._

---

📸 **Screenshot:**
> _Insert screenshot: Newly allocated EIP listed in the Elastic IP addresses table with its address visible._

---

### Step 4: Associate the EIP to the Instance

1. Select the checkbox next to the newly created EIP
2. Click **Actions** → **Associate Elastic IP address**
3. Configure the association:

| Field | Value |
|-------|-------|
| **Resource type** | Instance |
| **Instance** | `test instance` |
| **Private IP address** | Select the private IP of the instance |

4. Click **Associate**

---

📸 **Screenshot:**
> _Insert screenshot: Associate Elastic IP address dialog — Instance selected as "test instance", Private IP populated, Associate button visible._

---

📸 **Screenshot:**
> _Insert screenshot: Success confirmation after associating the EIP to the instance._

---

### Step 5: Verify the Fix

1. Navigate back to **Instances** → select `test instance`
2. Open the **Networking tab** — confirm the **Public IPv4 now shows the EIP address**
3. **Stop the instance** → wait for `Stopped` state → check the Networking tab
4. **Start the instance** → wait for `Running` state → check the Networking tab again

**Expected result:** The EIP address should be **identical** before and after the stop/start cycle.

---

📸 **Screenshot:**
> _Insert screenshot: Networking tab showing the EIP address now listed as the Public IPv4 address for "test instance"._

---

📸 **Screenshot:**
> _Insert screenshot: Networking tab AFTER stopping and restarting — EIP address is unchanged, confirming static behaviour._

---

**Final Comparison:**

| | Dynamic Public IP | Elastic IP |
|--|--|--|
| **Before stop** | `54.210.x.x` | `3.92.x.x` |
| **After restart** | `52.x.x.x` ❌ Changed | `3.92.x.x` ✅ Same |

**The customer's issue is resolved.** ✅

---

## Task 2 — Respond to the Customer

### My Response to Bob

> *"Hi Bob,*
>
> *After investigating your environment, I found the cause of your instance's IP address changing on every stop/start: your EC2 instance is using an auto-assigned public IP address, which is dynamic by nature. AWS releases auto-assigned public IPs back into the shared pool whenever an instance is stopped, so a different IP is assigned when it starts again.*
>
> *To fix this, I allocated an Elastic IP (EIP) address and associated it directly with your instance. An EIP is a static public IP that belongs to your account and stays attached to your instance across stop/start cycles — it will never change unless you explicitly release or reassociate it.*
>
> *I tested this by stopping and restarting the instance after attaching the EIP, and the IP address remained the same both times.*
>
> *One thing to keep in mind: AWS charges a small fee for EIPs that are allocated but not associated with a running instance. So if you ever stop using the instance long-term, either keep it associated or release the EIP to avoid unnecessary charges.*
>
> *Best regards,*  
> *Cloud Support Engineer"*

---

## Key Takeaways

### 1. Auto-Assigned Public IPs Are Dynamic
When an EC2 instance is stopped, its auto-assigned public IP is **released back to AWS**. On restart, a new IP from the pool is assigned. This is by design — it allows AWS to efficiently manage its IP address inventory across all customers.

### 2. Private IPs Are Always Persistent
A private IP assigned to an EC2 instance within a VPC **does not change** when the instance is stopped and started. It is only released when the instance is **terminated**.

### 3. Elastic IPs Solve the Static IP Problem
An EIP is the AWS solution for a persistent public IP. It:
- Stays associated with your instance across stop/start cycles
- Can be remapped to another instance instantly (great for failover scenarios)
- Costs nothing when attached to a **running** instance
- Incurs charges when allocated but **not in use**

### 4. EIP Remapping = Fast Failover
A powerful secondary use case for EIPs is **disaster recovery**. If a primary instance fails, you can remap the EIP to a standby instance in seconds — with no DNS propagation delay — because the IP address itself never changes.

```
Normal state:
EIP (3.92.x.x)  →  Primary Instance

After failure/remapping:
EIP (3.92.x.x)  →  Standby Instance  ✅ (instant, no DNS changes needed)
```

### 5. Summary Comparison

| Feature | Auto-Assigned Public IP | Elastic IP |
|---------|------------------------|------------|
| Persists on stop/start | ❌ No | ✅ Yes |
| Belongs to your account | ❌ No (shared pool) | ✅ Yes |
| Cost when not attached | Free | ⚠️ Charged |
| Can be remapped | ❌ No | ✅ Yes |
| Good for production | ❌ No | ✅ Yes |

---

> 💡 *Add your screenshots inline after each section above. Save images to a `/screenshots/` folder in your repo and reference them using `![description](./screenshots/filename.png)`. Recommended capture tools: Snipping Tool (Windows), Screenshot (Mac), or Flameshot (Linux).*
