# Create Subnets and Allocate IP Addresses in an Amazon VPC

> **Lab Type:** AWS Cloud Support Scenario  
> **Duration:** ~1 hour  
> **Role:** Cloud Support Engineer at AWS  
> **Core Concepts:** VPC Creation, CIDR Notation, Subnets, IP Address Allocation, RFC 1918

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [The Customer Scenario](#the-customer-scenario)
3. [Architecture](#architecture)
4. [Key Concepts Before You Start](#key-concepts-before-you-start)
5. [Task 1 — Investigate the Customer's Needs](#task-1--investigate-the-customers-needs)
   - [Step 1: Work Out the Correct CIDR Blocks](#step-1-work-out-the-correct-cidr-blocks)
   - [Step 2: Navigate to the VPC Console](#step-2-navigate-to-the-vpc-console)
   - [Step 3: Create the VPC](#step-3-create-the-vpc)
   - [Step 4: Verify the VPC and Subnet](#step-4-verify-the-vpc-and-subnet)
6. [Task 2 — Respond to the Customer](#task-2--respond-to-the-customer)
7. [Key Takeaways](#key-takeaways)

---

## Lab Overview

In this lab I acted as a **Cloud Support Engineer at AWS**, responding to a support ticket from a startup owner who was new to AWS. The customer needed help building their first VPC with the correct private IP range, enough addresses for their team, and a public subnet for their operations department.

The lab required me to:
- Analyse the customer's IP address requirements and calculate the correct CIDR blocks
- Confirm which `192.x.x.x` range qualifies as a private address per RFC 1918
- Build the VPC using the AWS Console with the correct configuration
- Verify all resources were created successfully

---

## The Customer Scenario

**Support Ticket — From: Paulo Santos (Startup Owner)**

> *"I'm new to AWS and I need help setting up a VPC. I would like to build only the VPC part and would like it to have around 15,000 private IP addresses available. I would also like the VPC IPv4 CIDR block to be a 192.x.x.x. I don't remember which is a private range though — can you confirm that? I would also like to allocate at least 50 IP addresses for the public subnet."*

**Three things to solve:**
1. Which `192.x.x.x` range is private per RFC 1918?
2. Which CIDR gives ~15,000 addresses?
3. Which CIDR gives 50+ addresses for the public subnet?

---

## Architecture

**What Paulo needs:**

```
                        Internet
                           │
                           ▼
                  Internet Gateway (IGW)
                           │
                           ▼
        ┌──────────────────────────────────────┐
        │         First VPC                    │
        │       192.168.0.0/18                 │
        │       (~16,384 addresses)            │
        │                                      │
        │   ┌──────────────────────────────┐   │
        │   │       Public Subnet          │   │
        │   │      192.168.1.0/26          │   │
        │   │      (64 addresses)          │   │
        │   │                              │   │
        │   │   [Operations Department]    │   │
        │   └──────────────────────────────┘   │
        └──────────────────────────────────────┘
```

---

## Key Concepts Before You Start

### What is a VPC?

A **Virtual Private Cloud (VPC)** is your own isolated network inside AWS — like having a private data center in the cloud. Inside it you can:
- Launch EC2 instances, databases, and other resources
- Control all inbound and outbound traffic
- Define your own IP address ranges using CIDR blocks

### What is a Subnet?

A **subnet** is a smaller network carved out of your VPC's IP range. Think of it like this:

```
VPC  =  The entire office building
Subnet  =  One floor of that building
```

- **Public subnet** — has a route to the internet via an Internet Gateway
- **Private subnet** — no direct internet access, used for backend resources

### RFC 1918 — Private IP Ranges

RFC 1918 defines the three ranges that are reserved for private use and will never be routed on the public internet:

| Range | CIDR | Addresses Available |
|-------|------|-------------------|
| `10.0.0.0` | `10.0.0.0/8` | ~16 million |
| `172.16.0.0` | `172.16.0.0/12` | ~1 million |
| `192.168.0.0` | `192.168.0.0/16` | ~65,000 |

> Paulo wanted a `192.x.x.x` address — the correct private range is **`192.168.0.0/16`** and anything within it.

---

## Task 1 — Investigate the Customer's Needs

### Step 1: Work Out the Correct CIDR Blocks

Before touching the console, the CIDR math needs to be done first.

#### VPC CIDR — Needs ~15,000 addresses

| CIDR | Addresses | Fits 15,000? |
|------|-----------|-------------|
| `/16` | 65,536 | ✅ Yes (too many) |
| `/17` | 32,768 | ✅ Yes (still large) |
| `/18` | 16,384 | ✅ Yes — closest above 15,000 |
| `/19` | 8,192 | ❌ Too few |

**Answer: `192.168.0.0/18`** → 16,384 addresses ✅

#### Subnet CIDR — Needs 50+ addresses

| CIDR | Addresses | Fits 50? |
|------|-----------|---------|
| `/25` | 128 | ✅ Yes |
| `/26` | 64 | ✅ Yes — smallest that fits |
| `/27` | 32 | ❌ Too few |

**Answer: `192.168.1.0/26`** → 64 addresses ✅

#### Final Configuration Summary

| Resource | CIDR | Addresses |
|----------|------|-----------|
| **VPC** | `192.168.0.0/18` | 16,384 |
| **Public Subnet** | `192.168.1.0/26` | 64 |

---

### Step 2: Navigate to the VPC Console

1. In the AWS Management Console search bar, type `VPC`
2. Select **VPC** from the results
3. You are now on the **VPC Dashboard**

---

📸 **Screenshot:**
> _Insert screenshot: AWS Management Console search bar with "VPC" typed and VPC service highlighted in results._

---

📸 **Screenshot:**
> _Insert screenshot: VPC Dashboard home page showing the Launch VPC Wizard or Create VPC button._

---

### Step 3: Create the VPC

1. Click **Create VPC** (orange button, top right)
2. Under **Resources to create**, select **VPC and more**

---

📸 **Screenshot:**
> _Insert screenshot: Create VPC page — "VPC and more" option selected under Resources to create._

---

#### Top section — fill in:

| Field | Value |
|-------|-------|
| **Name tag** | `First VPC` |
| **IPv4 CIDR** | `192.168.0.0/18` |
| **IPv6 CIDR block** | No IPv6 CIDR block |

---

📸 **Screenshot:**
> _Insert screenshot: Top of Create VPC form showing Name tag set to "First VPC" and IPv4 CIDR set to 192.168.0.0/18._

---

#### Middle section — configure subnets:

| Field | Value |
|-------|-------|
| **Number of Availability Zones** | `1` |
| **Number of public subnets** | `1` |
| **Number of private subnets** | `0` |
| **Public subnet CIDR** | `192.168.1.0/26` |

---

📸 **Screenshot:**
> _Insert screenshot: Subnet configuration section showing 1 AZ, 1 public subnet, 0 private subnets._

---

#### Bottom section — configure remaining options:

| Field | Value |
|-------|-------|
| **NAT gateways** | None |
| **VPC endpoints** | None |
| **Enable DNS hostnames** | ✅ Checked |
| **Enable DNS resolution** | ✅ Checked |

> ⚠️ **NAT Gateways cost money.** Since Paulo has no private subnets, set this to **None** to avoid unnecessary charges.

---

📸 **Screenshot:**
> _Insert screenshot: Bottom of Create VPC form showing NAT gateways set to None, VPC endpoints set to None, DNS options both checked, and the Preview panel showing "Subnets (1)" with one public subnet in us-west-2a._

---

#### Click Create VPC

After clicking **Create VPC**, AWS will provision all resources automatically. You will see a workflow screen with each resource being created:

- ✅ VPC
- ✅ Subnet
- ✅ Internet Gateway
- ✅ Route Table
- ✅ Route Table associations

---

📸 **Screenshot:**
> _Insert screenshot: VPC creation workflow screen showing all resources (VPC, subnet, IGW, route table) with green checkmarks._

---

### Step 4: Verify the VPC and Subnet

#### Verify the VPC

1. In the left navigation pane, click **Your VPCs**
2. Find **First VPC-vpc** — State should show **Available** ✅
3. Click on it and confirm the **IPv4 CIDR is `192.168.0.0/18`**

---

📸 **Screenshot:**
> _Insert screenshot: Your VPCs list showing "First VPC-vpc" with State: Available._

---

📸 **Screenshot:**
> _Insert screenshot: First VPC-vpc details panel showing IPv4 CIDR as 192.168.0.0/18._

---

#### Verify the Subnet

1. In the left navigation pane, click **Subnets**
2. Use the **Filter by VPC** dropdown and select **First VPC-vpc**
3. Confirm the public subnet appears with **CIDR `192.168.1.0/26`**

> **Note:** The default AWS VPC has its own subnets (with `172.31.x.x` CIDRs) that will show in the list — filter by your VPC to see only your subnet.

---

📸 **Screenshot:**
> _Insert screenshot: Subnets page filtered by First VPC-vpc, showing the public subnet with IPv4 CIDR 192.168.1.0/26._

---

#### Verify the Internet Gateway

1. In the left navigation pane, click **Internet gateways**
2. Confirm an IGW is attached to **First VPC-vpc**
3. State should show **Attached** ✅

---

📸 **Screenshot:**
> _Insert screenshot: Internet gateways list showing the IGW with State "Attached" and VPC set to First VPC-vpc._

---

#### Verify the Route Table

1. In the left navigation pane, click **Route tables**
2. Filter by **First VPC-vpc**
3. Confirm a route exists: `0.0.0.0/0 → igw-xxxxxxx`

---

📸 **Screenshot:**
> _Insert screenshot: Route tables page showing the route table for First VPC-vpc with a route pointing 0.0.0.0/0 to the Internet Gateway._

---

## Task 2 — Respond to the Customer

### My Response to Paulo

> *"Hi Paulo,*
>
> *Happy to help you get your first VPC set up! Here's what I've done and what you need to know:*
>
> *Regarding your question about the 192.x.x.x private range — the correct private range is 192.168.0.0/16, as defined by RFC 1918. This range is reserved exclusively for private networks and will never conflict with public internet addresses.*
>
> *For your VPC, I used the CIDR block 192.168.0.0/18, which gives you 16,384 IP addresses — the closest available block above your 15,000 requirement.*
>
> *For your public subnet, I used 192.168.1.0/26, which gives you 64 IP addresses — comfortably above the 50 you requested for your operations department.*
>
> *The VPC wizard also automatically created an Internet Gateway and Route Table, so your public subnet is already configured to reach the internet.*
>
> *Best regards,*
> *Cloud Support Engineer"*

---

## Key Takeaways

### 1. Always Use RFC 1918 for VPC CIDRs
Private IP ranges (`10.x.x.x`, `172.16.x.x`, `192.168.x.x`) are safe to use inside a VPC because they are never routed on the public internet. Using a public range as a VPC CIDR causes routing conflicts.

### 2. CIDR Block Size Chart — Quick Reference

| CIDR | Addresses | Good for |
|------|-----------|---------|
| `/16` | 65,536 | Large enterprise VPCs |
| `/17` | 32,768 | Large VPCs |
| `/18` | 16,384 | Medium VPCs (~15,000 requirement) |
| `/24` | 256 | Standard subnets |
| `/26` | 64 | Small subnets (50+ requirement) |
| `/27` | 32 | Very small subnets |
| `/28` | 16 | Minimum AWS subnet size |

### 3. The Subnet Must Always Be Smaller Than the VPC
The subnet CIDR must fit inside the VPC CIDR. You cannot have a `/16` subnet inside a `/18` VPC.

```
✅ Correct:   VPC /18  →  Subnet /26  (subnet fits inside VPC)
❌ Incorrect: VPC /26  →  Subnet /18  (subnet is larger than VPC)
```

### 4. VPC and More Wizard Saves Time
Choosing **"VPC and more"** automatically creates:
- The VPC
- Subnets (public and/or private)
- Internet Gateway
- Route Tables and associations

This eliminates manual setup steps and reduces the chance of misconfiguration.

### 5. NAT Gateway = Cost
NAT Gateways are not free. Only add one when you have **private subnets** that need outbound internet access. Paulo's setup needed neither — one public subnet with a direct IGW was sufficient.

---

> 💡 *Add your screenshots inline after each section. Save images to a `/screenshots/` folder in your repo and reference them using `![description](./screenshots/filename.png)`. Recommended capture tools: Snipping Tool (Windows), Screenshot (Mac), or Flameshot (Linux).*
