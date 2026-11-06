# Resource-Constrained Proxmox Homelab

![Architecture Diagram](https://github.com/user-attachments/assets/f840163e-ffad-4390-88d9-c78334f31f45">)

## 📖 Overview
This project documents the deployment of an enterprise-grade virtualization and containerization environment using legacy hardware. The primary goal was to maximize resource efficiency, demonstrating the ability to run modern IT infrastructure on highly constrained hardware (4GB RAM).

## 🎯 The Challenge
Modern hypervisors like Proxmox VE typically require 8GB+ of RAM to run traditional Virtual Machines (VMs) comfortably. The challenge was to build a fully functional, nested Docker environment on a Dell Optiplex 7010 with only **4GB of RAM** and a mechanical **500GB HDD**, without experiencing system thrashing or performance degradation.

## 💡 The Solution
Instead of resource-heavy Type-2 Virtual Machines, I utilized **LXC (Linux Containers)** at the Proxmox level. By enabling "Nesting" and "keyctl" features, I created a lightweight Debian 12 container that acts as a dedicated Docker host. This approach reduces the RAM footprint of the compute layer from ~2048MB (VM) to just **~120MB (LXC)**.

## 🛠️ Tech Stack
* **Hardware:** Dell Optiplex 7010 (4GB DDR3, 500GB HDD)
* **Hypervisor:** Proxmox VE (Bare-metal Type-1)
* **Compute:** LXC (Debian 12 Standard)
* **Containerization:** Docker Engine & Docker Compose
* **Networking:** Linux Bridge (vmbr0)

## 🚀 Deployment Steps

### 1. Proxmox Host Configuration
* Installed Proxmox VE via ISO.
* Configured `no-subscription` repository for stable, free updates.
* Removed the `pve-enterprise` repository to prevent update errors.

### 2. LXC Container Creation
Created a highly optimized container with the following resource limits:
* **CPU:** 1 Core
* **RAM:** 512 MB (Swap: 512 MB)
* **Storage:** 4GB 
* **Features:** `nesting=1`, `keyctl=1` (Required to run Docker inside LXC).

### 3. Docker Installation
Executed the official Docker installation script for Debian within the LXC console:
```bash
# Added Docker official GPG key and repository
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
# Installed Docker CE and Compose plugin
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
