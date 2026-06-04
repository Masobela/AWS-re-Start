# AWS Networking Concepts — Detailed Lab Walkthrough

> **Badge:** AWS Networking Concepts  
> **Platform:** AWS Practice Lab  
> **Region:** US East (N. Virginia) — `us-east-1`  
> **Subnets:** Web Server `10.10.0.0/24` | DB Server `10.10.2.0/24`

---

## 📋 Table of Contents

1. [Lab Overview](#lab-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Step 1 — Confirm Region & Navigate to EC2](#step-1--confirm-region--navigate-to-ec2)
4. [Step 2 — Locate the Web Server Instance](#step-2--locate-the-web-server-instance)
5. [Step 3 — Review Instance Networking Details](#step-3--review-instance-networking-details)
6. [Step 4 — Open the Subnet from the Instance](#step-4--open-the-subnet-from-the-instance)
7. [Step 5 — Inspect the Route Table (RouteTable2)](#step-5--inspect-the-route-table-routetable2)
8. [Step 6 — Remove the NAT Gateway Route](#step-6--remove-the-nat-gateway-route)
9. [Step 7 — Add an Internet Gateway Route](#step-7--add-an-internet-gateway-route)
10. [Step 8 — Verify the New Route](#step-8--verify-the-new-route)
11. [Step 9 — Edit WebServerSecurityGroup Inbound Rules](#step-9--edit-webserversecuritygroup-inbound-rules)
12. [Step 10 — Add HTTP Inbound Rule](#step-10--add-http-inbound-rule)
13. [Step 11 — Review Outbound Rules (Port 3306)](#step-11--review-outbound-rules-port-3306)
14. [Step 12 — DIY Challenge: Open Port 3306 on DbServerSecurityGroup](#step-12--diy-challenge-open-port-3306-on-dbserversecuritygroup)
15. [Step 13 — Validate the Connection](#step-13--validate-the-connection)
16. [Key Takeaways](#key-takeaways)

---

## Lab Overview

This lab demonstrates how to build a secure AWS network architecture that controls communication between internal resources and the internet. The core components explored are:

- **VPC** — isolated virtual network
- **Subnets** — public (web) and private (database) segments
- **Security Groups** — stateful firewall rules at the instance level
- **Internet Gateway (IGW)** — enables internet access for public subnets
- **Route Tables** — direct traffic within and outside the VPC
- **NAT Gateway** — outbound-only internet access for private subnets (reviewed and replaced)

The lab culminates in a **DIY challenge** where I independently configured a security group rule to allow the web server to communicate with the database server over MySQL port 3306.

---

## Architecture Diagram

```
Internet
    │
    ▼
Internet Gateway (igw-xxxxxxx)
    │
    ▼
┌─────────────────────────────────┐
│           VPC                   │
│                                 │
│  ┌──────────────────────────┐   │
│  │  Public Subnet           │   │
│  │  10.10.0.0/24            │   │
│  │                          │   │
│  │  [Web Server]            │   │
│  │  WebServerSecurityGroup  │   │
│  └──────────┬───────────────┘   │
│             │ port 3306         │
│  ┌──────────▼───────────────┐   │
│  │  Private Subnet          │   │
│  │  10.10.2.0/24            │   │
│  │                          │   │
│  │  [DB Server]             │   │
│  │  DbServerSecurityGroup   │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

---

## Step 1 — Confirm Region & Navigate to EC2

**What I did:**  
Confirmed the AWS region was set to **N. Virginia (us-east-1)** using the top navigation bar, then navigated to the EC2 service via the Services search box.

**Why it matters:**  
All resources (instances, security groups, VPCs) are region-specific. Working in the wrong region means your resources won't be visible.

```
Services Search → ec2 → Click EC2
```

---

<img width="1366" height="768" alt="Screenshot (94)" src="https://github.com/user-attachments/assets/4cb200b5-e884-4247-a2fe-553cb3b20628" />


---

## Step 2 — Locate the Web Server Instance

**What I did:**  
1. Clicked **Instances** in the left navigation pane
2. Selected the **Web Server** instance using the checkbox
3. Copied the **Public IPv4 address** from the Details tab

**Key note:** Public IP addresses are dynamically assigned in practice labs and will differ from any documentation screenshots.

---

<img width="1366" height="768" alt="Screenshot (95)" src="https://github.com/user-attachments/assets/ce004491-5477-4f24-a2fb-c64436734af0" />


---

## Step 3 — Review Instance Networking Details

**What I did:**  
With the Web Server instance still selected, clicked the **Networking tab** to review:
- Public IPv4 address
- Private IPv4 address
- Subnet ID

**Why it matters:**  
Understanding both the public and private IPs is fundamental to diagnosing connectivity — the public IP is used for internet-facing access, while the private IP governs internal VPC communication.

---

📸 **Screenshot:**
> _Insert screenshot: EC2 instance Networking tab showing Public IPv4, Private IPv4, and Subnet ID fields._

---

## Step 4 — Open the Subnet from the Instance

**What I did:**  
Clicked the **Subnet ID** link on the Networking tab. This opened the **Amazon VPC console** in a new browser tab, directly showing the subnet associated with the Web Server.

**Why it matters:**  
Navigating directly from the instance to its subnet is a fast way to trace network configuration without hunting through the VPC console manually.

---

📸 **Screenshot:**
> _Insert screenshot: Subnet ID link clicked from EC2 Networking tab, VPC console opened in a new tab showing WebServerSubnet selected._

---

## Step 5 — Inspect the Route Table (RouteTable2)

**What I did:**  
1. In the Subnets section, selected **WebServerSubnet**
2. Clicked the **Route table tab**
3. Clicked the link for **RouteTable2**
4. Reviewed the two existing routes:
   - `10.10.0.0/16` → `local` (VPC-internal traffic stays local)
   - `0.0.0.0/0` → `nat-xxxxxxx` (all other traffic routes through a NAT Gateway)

**Why it matters:**  
The NAT Gateway route means the subnet was previously behaving as a **private** subnet — instances could reach the internet outbound, but were not directly reachable from the internet. To make the web server publicly accessible, this needed to change.

---

📸 **Screenshot:**
> _Insert screenshot: RouteTable2 Routes tab showing two entries — local route and 0.0.0.0/0 pointing to a NAT gateway._

---

## Step 6 — Remove the NAT Gateway Route

**What I did:**  
Clicked **Remove** next to the `0.0.0.0/0 → nat-xxxxxxx` route to delete it.

**What this means:**  
Removing this route cut off any outbound internet access for instances in this subnet (temporarily). The subnet was now fully isolated until a new default route was added.

> ⚠️ **Note:** After removal, instances in this subnet cannot connect to external services until the next step is completed.

---

📸 **Screenshot:**
> _Insert screenshot: RouteTable2 Routes tab with the NAT gateway route highlighted and Remove button visible/clicked._

---

## Step 7 — Add an Internet Gateway Route

**What I did:**  
1. Clicked **Add route**
2. Set **Destination** to `0.0.0.0/0`
3. Set **Target** to `Internet Gateway`
4. Selected `igw-xxxxxxx` from the dropdown
5. Clicked **Save changes**

**Why it matters:**  
This replaced the NAT Gateway with a direct Internet Gateway, turning the web server's subnet into a true **public subnet**. Traffic destined for the internet now routes directly through the IGW rather than through a NAT device.

| Route | Before | After |
|-------|--------|-------|
| `0.0.0.0/0` | `nat-xxxxxxx` | `igw-xxxxxxx` |

---

📸 **Screenshot:**
> _Insert screenshot: Add route dialog with Destination 0.0.0.0/0, Target set to Internet Gateway, and igw-xxxxxxx selected._

---

## Step 8 — Verify the New Route

**What I did:**  
After saving, reviewed the **Routes tab** to confirm the updated entry:
- `0.0.0.0/0` → `igw-xxxxxxx` (Internet Gateway)

The subnet is now publicly reachable from the internet.

---

📸 **Screenshot:**
> _Insert screenshot: RouteTable2 Routes tab showing the new route with 0.0.0.0/0 pointing to the Internet Gateway (igw-xxxxxxx)._

---

## Step 9 — Edit WebServerSecurityGroup Inbound Rules

**What I did:**  
1. Returned to the EC2 Instances tab in the browser
2. Selected the **Web Server** instance
3. Clicked the **Security tab**
4. Clicked **WebServerSecurityGroup** under Security groups
5. On the **Inbound rules** tab, clicked **Edit inbound rules**

**Why it matters:**  
Even with the correct route in place, the security group acts as a firewall. Without an inbound rule allowing HTTP traffic, web requests would be silently dropped.

---

📸 **Screenshot:**
> _Insert screenshot: EC2 instance Security tab showing WebServerSecurityGroup link highlighted._

---

📸 **Screenshot:**
> _Insert screenshot: WebServerSecurityGroup details page, Inbound rules tab, Edit inbound rules button visible._

---

## Step 10 — Add HTTP Inbound Rule

**What I did:**  
1. Clicked **Add rule**
2. Set **Type** to `HTTP` (not HTTPS)
3. Set **Source** to `Anywhere-IPv4` (`0.0.0.0/0`)
4. Clicked **Save rules**

**Key observation during this step:**  
While scrolling through the Type dropdown, I noted the **MYSQL/Aurora** protocol (port 3306) — this would be required in the upcoming DIY section.

**Also noted:**  
For the DIY section, the Source field requires searching for a **specific security group** rather than selecting Anywhere-IPv4.

---

📸 **Screenshot:**
> _Insert screenshot: Add inbound rule dialog with Type set to HTTP and Source set to Anywhere-IPv4._

---

📸 **Screenshot:**
> _Insert screenshot: Success alert confirming the inbound rules were updated._

---

## Step 11 — Review Outbound Rules (Port 3306)

**What I did:**  
1. Clicked the **Outbound rules** tab on WebServerSecurityGroup
2. Reviewed the existing rule allowing outbound traffic on **port 3306**

**Why this matters:**  
The web server already had an outbound rule permitting MySQL traffic. This means the web server is configured to *initiate* database connections — now the DB server's security group needs to *accept* them.

> MySQL databases use port **3306** by default.

---

📸 **Screenshot:**
> _Insert screenshot: WebServerSecurityGroup Outbound rules tab showing the existing rule for port 3306._

---

## Step 12 — DIY Challenge: Open Port 3306 on DbServerSecurityGroup

**Objective:**  
Allow the web server (using `WebServerSecurityGroup`) to connect to the DB server (using `DbServerSecurityGroup`) over **TCP port 3306**.

**What I did:**  
1. Navigated to **Security Groups** and selected **DbServerSecurityGroup**
2. Clicked the **Inbound rules** tab → **Edit inbound rules**
3. Clicked **Add rule** and configured:

| Field | Value |
|-------|-------|
| **Type** | MYSQL/Aurora |
| **Protocol** | TCP |
| **Port Range** | 3306 |
| **Source** | `WebServerSecurityGroup` (searched by name) |

4. Clicked **Save rules**

**Why source = security group (not IP range)?**  
Using a security group as the source means *any instance belonging to WebServerSecurityGroup* is allowed through — regardless of its IP address. This is more robust and follows the principle of least privilege: only the web server tier can reach the database, not the open internet.

---

📸 **Screenshot:**
> _Insert screenshot: DbServerSecurityGroup Inbound rules — Edit inbound rules dialog showing new rule: Type MYSQL/Aurora, Port 3306, Source = WebServerSecurityGroup._

---

📸 **Screenshot:**
> _Insert screenshot: Success confirmation after saving the inbound rule on DbServerSecurityGroup._

---

## Step 13 — Validate the Connection

**What I did:**  
Returned to the lab's network diagram view. After applying the security group rule, the status between the web server (`10.10.0.0/24`) and the DB server (`10.10.2.0/24`) updated to:

> ✅ **Status: Connected**

The lab's test server independently verified that:
- The web server subnet can reach the DB server subnet
- The connection uses TCP port 3306
- Traffic is correctly filtered by security group membership

---

📸 **Screenshot:**
> _Insert screenshot: Lab network diagram showing the connection status between Web Server and DB Server as "Connected"._

---

## Key Takeaways

### 🔒 Security Group Chaining
Rather than opening the database to a CIDR range, I referenced `WebServerSecurityGroup` directly as the inbound source on `DbServerSecurityGroup`. This is a **best practice** — only instances in the web tier can talk to the database tier.

### 🌐 NAT Gateway vs Internet Gateway
| | NAT Gateway | Internet Gateway |
|--|-------------|-----------------|
| **Outbound internet** | ✅ Yes | ✅ Yes |
| **Inbound from internet** | ❌ No | ✅ Yes |
| **Use case** | Private subnets needing outbound access | Public-facing subnets |

### 🗺️ Route Tables Control Traffic Flow
Changing a single route (`0.0.0.0/0`) from a NAT Gateway to an Internet Gateway fundamentally changed the subnet's exposure model from private to public.

### 🛡️ Defence in Depth
The architecture uses **two layers** of access control:
1. **Route tables** — control whether traffic can enter/leave a subnet at all
2. **Security groups** — control which ports and sources are allowed at the instance level

---

> 💡 *All screenshots should be added inline after each section above. Recommended tool: Snipping Tool (Windows), Screenshot (Mac), or Flameshot (Linux).*
