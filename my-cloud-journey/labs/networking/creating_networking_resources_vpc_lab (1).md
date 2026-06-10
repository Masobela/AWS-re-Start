# Creating Networking Resources in an Amazon VPC

> **Lab Type:** AWS Cloud Support Scenario  
> **Duration:** ~60 minutes  
> **Role:** Cloud Support Engineer at AWS  
> **Core Concepts:** VPC, Internet Gateway, Route Tables, NACLs, Security Groups, EC2, Ping Connectivity

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [The Customer Scenario](#the-customer-scenario)
3. [Architecture](#architecture)
4. [Key Concepts Before You Start](#key-concepts-before-you-start)
5. [Task 1 — Build the VPC and All Networking Resources](#task-1--build-the-vpc-and-all-networking-resources)
   - [Step 1: Create the VPC](#step-1-create-the-vpc)
   - [Step 2: Create the Subnet](#step-2-create-the-subnet)
   - [Step 3: Create the Route Table](#step-3-create-the-route-table)
   - [Step 4: Create and Attach the Internet Gateway](#step-4-create-and-attach-the-internet-gateway)
   - [Step 5: Add IGW Route and Associate Subnet to Route Table](#step-5-add-igw-route-and-associate-subnet-to-route-table)
   - [Step 6: Create the Network ACL (NACL)](#step-6-create-the-network-acl-nacl)
   - [Step 7: Create the Security Group](#step-7-create-the-security-group)
6. [Task 2 — Launch an EC2 Instance](#task-2--launch-an-ec2-instance)
7. [Task 3 — Test Internet Connectivity with Ping](#task-3--test-internet-connectivity-with-ping)
8. [Key Takeaways](#key-takeaways)

---

## Lab Overview

In this lab I acted as a **Cloud Support Engineer at AWS**, responding to a follow-up ticket from a startup owner who had set up a VPC but couldn't get internet connectivity. The customer couldn't even ping outside the VPC.

The lab required me to build every networking component from scratch in the correct order:
- VPC → Subnet → Route Table → Internet Gateway → NACL → Security Group → EC2 Instance
- Test connectivity by SSHing into the instance and running `ping google.com`

The lab is only complete when a **successful ping** is achieved — confirming full end-to-end network connectivity.

---

## The Customer Scenario

**Support Ticket — From: Brock (Startup Owner)**

> *"I previously reached out to you regarding help setting up my VPC. I thought I knew how to attach all the resources to make an internet connection, but I cannot even ping outside the VPC. All I need to do is ping! Can you please help me set up my VPC to where it has network connectivity and can ping?"*

**The problem:** Brock has a VPC but is missing one or more of the components needed for internet connectivity — likely no Internet Gateway, no route to it, or security group/NACL rules blocking ICMP traffic.

**The fix:** Build all required networking resources in the correct order and verify with a live ping test.

---

## Architecture

```
                          Internet
                             │
                             ▼
                   Internet Gateway (IGW)
                   "IGW test VPC"
                             │
                             ▼
          ┌──────────────────────────────────────┐
          │           Test VPC                   │
          │         192.168.0.0/18               │
          │                                      │
          │  ┌────────────────────────────────┐  │
          │  │  Public Subnet                 │  │
          │  │  192.168.1.0/26                │  │
          │  │                                │  │
          │  │  ┌──────────────────────────┐  │  │
          │  │  │   Public Security Group  │  │  │
          │  │  │   SSH / HTTP / HTTPS     │  │  │
          │  │  │                          │  │  │
          │  │  │   [EC2 Instance]         │  │  │
          │  │  └──────────────────────────┘  │  │
          │  │                                │  │
          │  │  Public Subnet NACL            │  │
          │  │  Allow all inbound/outbound    │  │
          │  └────────────────────────────────┘  │
          │                                      │
          │  Public Route Table                  │
          │  0.0.0.0/0 → IGW                     │
          └──────────────────────────────────────┘
```

---

## Key Concepts Before You Start

### The Build Order — Top Down

The left navigation pane in the VPC console is your guide. Always build in this order:

```
1. VPC              ← The container for everything
2. Subnet           ← A segment of the VPC's IP range
3. Route Table      ← Directs traffic within and outside the VPC
4. Internet Gateway ← Enables internet access
5. NACL             ← Subnet-level firewall (stateless)
6. Security Group   ← Instance-level firewall (stateful)
7. EC2 Instance     ← The resource that actually needs connectivity
```

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Level** | Instance | Subnet |
| **State** | Stateful | Stateless |
| **Default behaviour** | Blocks everything | Allows everything |
| **Rules** | Allow only | Allow and Deny |
| **Use case** | Control per-instance access | Broad subnet-level filtering |

> **Stateful** means if you allow inbound traffic, the return traffic is automatically allowed.  
> **Stateless** means you must explicitly allow both inbound AND outbound traffic.

### What the Internet Gateway Does

An IGW has two jobs:
1. **NAT** — translates private IPs to public IPs for outbound traffic
2. **Route target** — acts as the destination for `0.0.0.0/0` in the route table, pointing all internet-bound traffic to itself

---

## Task 1 — Build the VPC and All Networking Resources

### Step 1: Create the VPC

1. In the AWS Console, navigate to **VPC**
2. In the left navigation pane, click **Your VPCs**
3. Click **Create VPC** (top right)
4. Configure:

| Field | Value |
|-------|-------|
| **Resources to create** | VPC only |
| **Name tag** | `Test VPC` |
| **IPv4 CIDR block** | `192.168.0.0/18` |
| Everything else | Leave as default |

5. Click **Create VPC**

---

📸 **Screenshot:**
> _Insert screenshot: Create VPC form filled in — Name: "Test VPC", IPv4 CIDR: 192.168.0.0/18._

---

📸 **Screenshot:**
> _Insert screenshot: Your VPCs list showing "Test VPC" with State: Available._

---

### Step 2: Create the Subnet

1. In the left navigation pane, click **Subnets**
2. Click **Create subnet** (top right)
3. Configure:

| Field | Value |
|-------|-------|
| **VPC ID** | `Test VPC` |
| **Subnet name** | `Public subnet` |
| **Availability Zone** | No preference |
| **IPv4 CIDR block** | `192.168.1.0/26` |

4. Click **Create subnet**

> The subnet CIDR `/26` gives **64 addresses** and sits inside the VPC's `/18` range — the subnet must always be smaller than the VPC.

---

📸 **Screenshot:**
> _Insert screenshot: Create subnet form — VPC set to Test VPC, Subnet name "Public subnet", IPv4 CIDR 192.168.1.0/26._

---

📸 **Screenshot:**
> _Insert screenshot: Subnets list showing "Public subnet" with IPv4 CIDR 192.168.1.0/26 and State: Available._

---

### Step 3: Create the Route Table

1. In the left navigation pane, click **Route tables**
2. Click **Create route table** (top right)
3. Configure:

| Field | Value |
|-------|-------|
| **Name** | `Public route table` |
| **VPC** | `Test VPC` |

4. Click **Create route table**

> The route table doesn't do anything yet — you'll add the IGW route and subnet association after the IGW is created.

---

📸 **Screenshot:**
> _Insert screenshot: Create route table form — Name: "Public route table", VPC: Test VPC._

---

📸 **Screenshot:**
> _Insert screenshot: Route tables list showing "Public route table" associated with Test VPC._

---

### Step 4: Create and Attach the Internet Gateway

#### Create the IGW

1. In the left navigation pane, click **Internet gateways**
2. Click **Create internet gateway** (top right)
3. Configure:

| Field | Value |
|-------|-------|
| **Name tag** | `IGW test VPC` |

4. Click **Create internet gateway**

---

📸 **Screenshot:**
> _Insert screenshot: Create internet gateway form — Name tag: "IGW test VPC"._

---

#### Attach the IGW to the VPC

After creation, the IGW state shows **Detached**. You must attach it:

1. With the IGW selected, click **Actions** → **Attach to VPC**
2. Select **Test VPC** from the dropdown
3. Click **Attach internet gateway**

The state should now show **Attached** ✅

---

📸 **Screenshot:**
> _Insert screenshot: Actions menu open with "Attach to VPC" highlighted._

---

📸 **Screenshot:**
> _Insert screenshot: Internet gateways list showing "IGW test VPC" with State: Attached and VPC: Test VPC._

---

### Step 5: Add IGW Route and Associate Subnet to Route Table

#### Add the IGW Route

1. In the left navigation pane, click **Route tables**
2. Select **Public route table**
3. Scroll down and click the **Routes** tab
4. Click **Edit routes**
5. Click **Add route** and configure:

| Field | Value |
|-------|-------|
| **Destination** | `0.0.0.0/0` |
| **Target** | Internet Gateway → select `IGW test VPC` |

6. Click **Save changes**

> `0.0.0.0/0` means "all traffic not matched by another route" — essentially any traffic going to the internet will be directed to the IGW.

---

📸 **Screenshot:**
> _Insert screenshot: Edit routes page — new route added with Destination 0.0.0.0/0 and Target set to IGW test VPC._

---

📸 **Screenshot:**
> _Insert screenshot: Routes tab showing two entries — the local route (192.168.0.0/18 → local) and the IGW route (0.0.0.0/0 → igw-xxxxxxx)._

---

#### Associate the Subnet to the Route Table

1. Still on the **Public route table**, click the **Subnet associations** tab
2. Click **Edit subnet associations**
3. Check the box next to **Public subnet**
4. Click **Save associations**

> Every route table must be associated to a subnet. Without this step, the subnet has no route to the IGW even though the route table has one.

---

📸 **Screenshot:**
> _Insert screenshot: Edit subnet associations page — Public subnet checkbox selected._

---

📸 **Screenshot:**
> _Insert screenshot: Subnet associations tab confirming Public subnet is now associated to the Public route table._

---

### Step 6: Create the Network ACL (NACL)

1. In the left navigation pane, click **Network ACLs**
2. Click **Create network ACL** (top right)
3. Configure:

| Field | Value |
|-------|-------|
| **Name** | `Public Subnet NACL` |
| **VPC** | `Test VPC` |

4. Click **Create network ACL**

#### Configure Inbound Rules

1. Select **Public Subnet NACL** from the list
2. Click the **Inbound rules** tab → **Edit inbound rules**
3. Click **Add new rule**:

| Field | Value |
|-------|-------|
| **Rule number** | `100` |
| **Type** | All traffic |
| **Source** | `0.0.0.0/0` |
| **Allow/Deny** | Allow |

4. Click **Save changes**

#### Configure Outbound Rules

1. Click the **Outbound rules** tab → **Edit outbound rules**
2. Click **Add new rule**:

| Field | Value |
|-------|-------|
| **Rule number** | `100` |
| **Type** | All traffic |
| **Destination** | `0.0.0.0/0` |
| **Allow/Deny** | Allow |

3. Click **Save changes**

> The `*` (asterisk) rule at the bottom is a default deny — it catches anything not matched by rule 100 and blocks it. Since NACLs are stateless, both inbound AND outbound rules must be explicitly set.

---

📸 **Screenshot:**
> _Insert screenshot: NACL Inbound rules tab — Rule 100 allowing All traffic from 0.0.0.0/0, with the default deny (*) rule below it._

---

📸 **Screenshot:**
> _Insert screenshot: NACL Outbound rules tab — Rule 100 allowing All traffic to 0.0.0.0/0, with the default deny (*) rule below it._

---

### Step 7: Create the Security Group

1. In the left navigation pane, click **Security Groups**
2. Click **Create security group** (top right)
3. Configure Basic details:

| Field | Value |
|-------|-------|
| **Security group name** | `public security group` |
| **Description** | `allows public access` |
| **VPC** | Remove default VPC → select `Test VPC` |

#### Add Inbound Rules

Click **Add rule** three times and configure:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| SSH | TCP | 22 | Anywhere-IPv4 (`0.0.0.0/0`) |
| HTTP | TCP | 80 | Anywhere-IPv4 (`0.0.0.0/0`) |
| HTTPS | TCP | 443 | Anywhere-IPv4 (`0.0.0.0/0`) |

#### Outbound Rules

Leave the default outbound rule:

| Type | Protocol | Port | Destination |
|------|----------|------|-------------|
| All traffic | All | All | `0.0.0.0/0` |

4. Click **Create security group**

> Unlike NACLs, security groups are **stateful** — you only need to allow inbound traffic and the return traffic is automatically permitted. The default outbound rule allows all traffic out.

---

📸 **Screenshot:**
> _Insert screenshot: Create security group — Basic details filled in with name "public security group" and VPC set to Test VPC._

---

📸 **Screenshot:**
> _Insert screenshot: Security group Inbound rules showing SSH (22), HTTP (80), and HTTPS (443) all allowed from 0.0.0.0/0._

---

📸 **Screenshot:**
> _Insert screenshot: Completed security group summary showing both inbound and outbound rules._

---

## Task 2 — Launch an EC2 Instance

1. Navigate to **EC2 → Instances → Launch instances**
2. Configure:

| Setting | Value |
|---------|-------|
| **Name** | _(leave blank)_ |
| **AMI** | Amazon Linux 2023 AMI |
| **Instance type** | `t3.micro` |
| **Key pair** | `vockey` |
| **VPC** | `Test VPC` |
| **Subnet** | `Public subnet` |
| **Auto-assign public IP** | Enable |
| **Security group** | Select existing → `public security group` |

3. Click **Launch instance**
4. Click **View all instances** and wait for **Instance state: Running** and **2/2 status checks**

---

📸 **Screenshot:**
> _Insert screenshot: EC2 Launch Instance — Network Settings section showing Test VPC selected, Public subnet selected, Auto-assign public IP enabled, and public security group selected._

---

📸 **Screenshot:**
> _Insert screenshot: EC2 Instances list showing the instance in Running state with 2/2 status checks passed._

---

📸 **Screenshot:**
> _Insert screenshot: EC2 instance Networking tab showing the assigned Public IPv4 address._

---

## Task 3 — Test Internet Connectivity with Ping

### Connect via SSH

**macOS / Linux:**
```bash
# Navigate to your key file location
cd ~/Downloads

# Set correct permissions
chmod 400 labsuser.pem

# SSH into the instance (replace with your actual public IP)
ssh -i labsuser.pem ec2-user@<public-ip>

# Type 'yes' when prompted for first connection
```

**Windows (PuTTY):**
1. Open PuTTY
2. **Session → Host Name:** enter the instance's public IP
3. **Connection → SSH → Auth:** browse to `labsuser.ppk`
4. Click **Open**

---

📸 **Screenshot:**
> _Insert screenshot: Terminal showing successful SSH connection — ec2-user prompt visible (e.g. `[ec2-user@ip-192-168-1-x ~]$`)._

---

### Run the Ping Test

Once connected, run:

```bash
ping google.com
```

**Expected successful output:**
```
PING google.com (142.250.x.x) 56(84) bytes of data.
64 bytes from lga25s71-in-f14.1e100.net: icmp_seq=1 ttl=117 time=1.23 ms
64 bytes from lga25s71-in-f14.1e100.net: icmp_seq=2 ttl=117 time=1.18 ms
64 bytes from lga25s71-in-f14.1e100.net: icmp_seq=3 ttl=117 time=1.21 ms
```

Stop the ping with **CTRL+C** (Windows) or **CMD+C** (Mac).

**What a successful ping confirms:**
- ✅ The EC2 instance has a public IP
- ✅ The route table has a route to the IGW (`0.0.0.0/0`)
- ✅ The IGW is attached to the VPC
- ✅ The security group allows outbound traffic
- ✅ The NACL allows inbound and outbound traffic
- ✅ Full end-to-end internet connectivity is working

---

📸 **Screenshot:**
> _Insert screenshot: Terminal showing ping google.com output with replies and 0% packet loss._

---

## Key Takeaways

### 1. The Build Order Matters
Always build VPC resources top-down. Skipping steps or building out of order leads to broken connectivity that's hard to debug.

```
VPC → Subnet → Route Table → IGW → Attach IGW → 
Add route → Associate subnet → NACL → Security Group → EC2
```

### 2. The IGW Needs Two Things to Work
Just creating an IGW isn't enough. It must be:
1. **Attached to the VPC**
2. **Added as a target (`0.0.0.0/0`) in the route table**

Miss either step and internet traffic has nowhere to go.

### 3. Route Table Must Be Associated to a Subnet
A route table with the correct routes does nothing until it is **explicitly associated** with a subnet. Without this, the subnet falls back to the default VPC route table.

### 4. NACLs Are Stateless — Rules Go Both Ways
Because NACLs don't track connection state, you must allow traffic in **both directions**. An inbound allow rule alone is not enough — outbound must also be configured.

### 5. Security Groups Are the Last Line of Defence
Security groups sit directly around the instance. Even with perfect routing, a missing inbound rule on the security group will silently drop all traffic before it reaches the instance.

### 6. Full Connectivity Checklist

| Component | What to Check |
|-----------|--------------|
| **VPC** | Correct CIDR (`192.168.0.0/18`) |
| **Subnet** | Inside VPC CIDR, auto-assign public IP enabled |
| **IGW** | Created AND attached to VPC |
| **Route table** | `0.0.0.0/0 → IGW` route exists AND associated to subnet |
| **NACL** | Inbound AND outbound rules allow traffic |
| **Security group** | Inbound rules allow SSH/HTTP/HTTPS, outbound allows all |
| **EC2** | Launched in correct subnet with correct security group |

---

> 💡 *Add your screenshots inline after each section. Save images to a `/screenshots/` folder in your repo and reference them using `![description](./screenshots/filename.png)`. Recommended capture tools: Snipping Tool (Windows), Screenshot (Mac), or Flameshot (Linux).*
