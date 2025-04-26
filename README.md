# 🛡️ VPN Access via Private IP SNAT for Kubernetes Services

## 🔍 Project Overview

This project documents how I configured a secure and compliant network route for my Kubernetes workloads to communicate with **Vendor A** through a **site-to-site VPN**, while meeting their strict requirement of **private IP whitelisting**. The core solution was implemented using **iptables SNAT**, and it ensures all outbound traffic from my Kubernetes cluster appears to come from a single approved private IP.

---

## 📌 Problem Statement

Vendor A requires:

- Communication **only over a secure AWS site-to-site VPN**.
- Only **one specific private IP address** whitelisted for all incoming connections.

My infrastructure:

- A **Kubernetes cluster** in a **private subnet**.
- An **NGINX Ingress Controller** for traffic routing.
- A **NAT Gateway** for external internet access.
- No dedicated IP address per pod/service — which doesn't meet Vendor A's IP control requirement.

---

## ✅ Solution Summary

I configured **Source NAT (SNAT)** using `iptables` on an EC2 instance connected to the Kubernetes cluster network. This made all traffic to Vendor A appear as if it originated from the whitelisted private IP (`172.31.87.184`), satisfying their requirement.

---

## 🛠️ Step-by-Step Implementation


![Screenshot (321)](https://github.com/user-attachments/assets/07c0c59c-670e-424e-9ad2-dbd0caa2baf7)


### 1. 🔍 Identify Whitelisted Private IP

First, I got the IP assigned to the EC2 instance that would serve as the SNAT gateway:

```bash
hostname -I
```

I selected the IP `172.31.87.184` — which matches the EC2 hostname (`ip-172-31-87-184`) and is in the private VPC subnet.

---
![Screenshot (320)](https://github.com/user-attachments/assets/8474b6ec-881c-49fa-bf65-0aed909b6068)

### 2. 🔧 Enable IP Forwarding

This allows the EC2 instance to route packets between interfaces:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

To make this change persistent, I can add the following line to `/etc/sysctl.conf`:

```bash
net.ipv4.ip_forward = 1
```

---
![Screenshot (320)](https://github.com/user-attachments/assets/5a00adb9-f438-4ee5-9493-9ce619aea554)


### 3. 🔁 Add SNAT Rule via iptables

I used iptables to masquerade all outbound traffic going through the VPN interface (`eth0`) with the approved IP:

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 172.31.87.184
```

This ensures any connection to Vendor A looks like it came from `172.31.87.184`.

---

### 4. 🔒 Ensure VPN Connection is Up

Vendor A’s VPN is configured with a Site-to-Site VPN connection in AWS. This EC2 instance is part of the route table that handles traffic to Vendor A.

---
![Screenshot (323)](https://github.com/user-attachments/assets/abd2becc-5138-4c1b-a4b9-8c3edda13dfb)

### 5. 🔁 Make iptables Persistent (Optional but Recommended)

Install the persistent package (for Debian/Ubuntu):

```bash
sudo apt install iptables-persistent
```

Save the rules:

```bash
sudo netfilter-persistent save
```

---

### ✅ Result

- All Kubernetes services can now communicate with Vendor A’s systems **via the VPN**.
- **Vendor A receives traffic only from `172.31.87.184`**, satisfying their IP whitelisting rule.
- The traffic remains private and secure over the site-to-site VPN.

---
