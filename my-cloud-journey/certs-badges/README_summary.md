# AWS Networking Concepts — Badge Lab Summary

## 🏅 Badge Earned: AWS Networking Concepts

This lab was completed as part of an AWS practice environment focused on core networking concepts. The objective was to build and configure a secure, functional network architecture on AWS from scratch.

---

## 🎯 What I Did

| Task | Description |
|------|-------------|
| **VPC & Subnets** | Explored a pre-built VPC with public and private subnets |
| **Route Tables** | Replaced a NAT gateway route with a direct Internet Gateway route for the web server subnet |
| **Internet Gateway** | Associated an IGW to make the web server subnet publicly reachable |
| **Security Groups** | Configured inbound rules on `WebServerSecurityGroup` (HTTP) and `DbServerSecurityGroup` (port 3306) |
| **DB Isolation** | Allowed the web server to communicate with the DB server over MySQL port 3306 while keeping the DB in a private subnet |

---

## 🧠 Key Concepts Practiced

- **VPCs and Subnets** — public (`10.10.0.0/24`) vs private (`10.10.2.0/24`) subnet design
- **Route Tables** — controlling traffic flow with NAT gateway vs Internet Gateway routes
- **Security Groups** — stateful, rule-based traffic control at the instance level
- **Principle of Least Privilege** — DB server only accepts traffic from the web server's security group, not the open internet
- **DIY Challenge** — independently configured port 3306 inbound rule on `DbServerSecurityGroup` sourced from `WebServerSecurityGroup`

---

## 🛠️ Technologies Used

- Amazon EC2
- Amazon VPC
- AWS Security Groups
- Internet Gateway
- Route Tables
- NAT Gateway (reviewed and replaced)

---

## ✅ Outcome

The web server (`10.10.0.0/24`) successfully established a connection to the DB server (`10.10.2.0/24`) over port 3306, confirmed by the lab's connection status changing to **Connected**.
