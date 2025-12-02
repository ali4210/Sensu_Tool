# Sensu Monitoring Infrastructure: Ubuntu Home Server + Kali & CentOS Agents

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Sensu Go](https://img.shields.io/badge/Sensu_Go-6.13.0+-green.svg?logo=sensu)](https://sensu.io/)
[![Platform](https://img.shields.io/badge/Platform-Ubuntu%20%7C%20Kali%20%7C%20CentOS-blue.svg?logo=linux)](https://github.com)

A comprehensive, enterprise-grade guide to setting up a distributed Sensu monitoring infrastructure. This project details the deployment of a **Sensu Backend on Ubuntu Server** (Home Server) and monitored **Sensu Agents on Kali Linux and CentOS**.

This guide solves the common **GPG SHA-1/Weak Digest errors** found in modern Kali Linux distributions and utilizes official **Sensu Assets** for professional-grade metric collection.

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Prerequisites](#-prerequisites)
- [Installation Guide](#-installation-guide)
  - [Part 1: Ubuntu Backend (Home Server)](#part-1-ubuntu-backend-home-server)
  - [Part 2: Kali Linux Agent (GPG/SHA-1 Fix)](#part-2-kali-linux-agent-gpgsha-1-fix)
  - [Part 3: CentOS Agent](#part-3-centos-agent)
  - [Part 4: Installing Assets (Plugins)](#part-4-installing-assets-plugins)
- [Configuring Checks](#-configuring-checks)
- [Verification](#-verification)
- [Troubleshooting](#-troubleshooting)
- [Screenshots](#-screenshots)
- [Contributing](#-contributing)
- [Author](#-author)

## 🎯 Overview

This project implements a secure monitoring solution where:
- **Ubuntu Server** acts as the central Event Pipeline (Backend/Server).
- **Kali Linux** acts as a monitored endpoint (Debian-based Agent).
- **CentOS** acts as a monitored endpoint (RHEL-based Agent).
- **Sensu Assets** are utilized to perform professional checks on CPU, Memory, and Disk usage across all nodes.

## 🏗️ Architecture

```mermaid
graph TD
    subgraph Agents
    A[Kali Linux Agent] -- WebSocket 8081 --> S
    B[CentOS Agent] -- WebSocket 8081 --> S
    end
    
    subgraph Home Server
    S[Ubuntu Backend] -- API 8080 --> C[Sensu CLI / Web UI]
    S -- Data --> D[etcd Database]
    end
    
    style A fill:#f96,stroke:#333,stroke-width:2px
    style B fill:#f96,stroke:#333,stroke-width:2px
    style S fill:#61d668,stroke:#333,stroke-width:2px


📦 Prerequisites
Ubuntu Home Server:

Ubuntu 20.04/22.04 LTS

Static IP Address (Required for agents to connect)

Ports Open: 8080 (API), 8081 (Agent WS), 3000 (Dashboard)

Agents (Kali & CentOS):

Root/Sudo privileges

Network connectivity to the Ubuntu Server IP

🚀 Installation Guide
Part 1: Ubuntu Backend (Home Server)
Perform these steps on your Ubuntu Server.

=> 1. Add Repository & Install Backend

Bash

# Update system
sudo apt-get update

# Add the official Sensu repository
curl -s [https://packagecloud.io/install/repositories/sensu/stable/script.deb.sh](https://packagecloud.io/install/repositories/sensu/stable/script.deb.sh) | sudo bash

# Install the backend
sudo apt-get install sensu-go-backend -y
=> 2. Initialize & Start Service

Bash

# Enable and start the service
sudo systemctl enable --now sensu-backend

# Initialize the configuration (Replace credentials!)
export SENSU_BACKEND_CLUSTER_ADMIN_USERNAME=admin
export SENSU_BACKEND_CLUSTER_ADMIN_PASSWORD=StrongPassword123

sudo --preserve-env sensu-backend init
=> 3. Configure Sensu CLI (sensuctl)

Bash

# Install CLI
sudo apt-get install sensu-go-cli -y

# Configure connection
sensuctl configure -n \
  --username 'admin' \
  --password 'StrongPassword123' \
  --namespace default \
  --url '[http://127.0.0.1:8080](http://127.0.0.1:8080)'
=> 4. Create a Dedicated Agent User

Bash

sensuctl user create agent --password 'AgentPassword123'
=> 5. Configure Firewall

Bash

sudo ufw allow 8080/tcp  # API
sudo ufw allow 8081/tcp  # WebSocket
sudo ufw allow 3000/tcp  # Web UI
Part 2: Kali Linux Agent (GPG/SHA-1 Fix)
⚠️ IMPORTANT: This section uses the "Signed-By" method to prevent the common GPG SHA-1/Weak Digest errors on Kali.

Perform these steps on your Kali Linux machine.

=> 1. Manually Add GPG Key (The Secure Way) Do not use the auto-script. Run these commands line-by-line:

Bash

# Update and install prerequisites
sudo apt-get update
sudo apt-get install -y curl gnupg2 debian-archive-keyring

# Download the GPG key and convert it to a specific keyring (Fixes SHA-1 error)
curl -fsSL [https://packagecloud.io/sensu/stable/gpgkey](https://packagecloud.io/sensu/stable/gpgkey) | sudo gpg --dearmor -o /usr/share/keyrings/sensu-archive-keyring.gpg

# Add the repository explicitly pointing to that key
echo "deb [signed-by=/usr/share/keyrings/sensu-archive-keyring.gpg] [https://packagecloud.io/sensu/stable/debian/](https://packagecloud.io/sensu/stable/debian/) buster main" | sudo tee /etc/apt/sources.list.d/sensu.list

# Install the agent
sudo apt-get update
sudo apt-get install sensu-go-agent -y
=> 2. Configure the Agent Edit the config: sudo nano /etc/sensu/agent.yml (Replace UBUNTU_IP with your Home Server IP)

YAML

---
backend-url:
  - "ws://UBUNTU_IP:8081"
name: "kali-linux"
subscriptions:
  - system
  - linux
  - kali
namespace: "default"
user: "agent"
password: "AgentPassword123"
=> 3. Start the Agent

Bash

sudo systemctl enable --now sensu-agent
Part 3: CentOS Agent
Perform these steps on your CentOS machine.

=> 1. Install Agent

Bash

# Add Sensu repository (RPM-based)
curl -s [https://packagecloud.io/install/repositories/sensu/stable/script.rpm.sh](https://packagecloud.io/install/repositories/sensu/stable/script.rpm.sh) | sudo bash

# Install agent
sudo yum install sensu-go-agent -y
=> 2. Configure the Agent Edit the config: sudo vi /etc/sensu/agent.yml (Replace UBUNTU_IP with your Home Server IP)

YAML

---
backend-url:
  - "ws://UBUNTU_IP:8081"
name: "centos-server"
subscriptions:
  - system
  - linux
  - centos
namespace: "default"
user: "agent"
password: "AgentPassword123"
=> 3. Start the Agent

Bash

sudo systemctl enable --now sensu-agent
Part 4: Installing Assets (Plugins)
Run these commands on your Ubuntu Backend using sensuctl. This installs the official plugins for accurate CPU, RAM, and Disk monitoring.

=> 1. Install Assets

Bash

sensuctl asset add sensu/check-cpu-usage
sensuctl asset add sensu/check-memory-usage
sensuctl asset add sensu/check-disk-usage
=> 2. Verify Assets

Bash

sensuctl asset list
Ensure all assets show as "Available".

⚙️ Configuring Checks
Create these checks on the Ubuntu Backend. They will automatically be deployed to Kali and CentOS agents.

=> CPU Check

Bash

sensuctl check create check-cpu \
  --command 'check-cpu-usage -w 75 -c 90' \
  --interval 60 \
  --subscriptions system \
  --runtime-assets sensu/check-cpu-usage
=> Memory Check

Bash

sensuctl check create check-memory \
  --command 'check-memory-usage -w 80 -c 90' \
  --interval 60 \
  --subscriptions system \
  --runtime-assets sensu/check-memory-usage
=> Disk Check

Bash

sensuctl check create check-disk \
  --command 'check-disk-usage -w 80 -c 90' \
  --interval 3600 \
  --subscriptions system \
  --runtime-assets sensu/check-disk-usage
✅ Verification
=> Check Connected Agents (On Ubuntu)

Bash

sensuctl entity list
Expected Output:

Plaintext

      ID         Class    OS           Subscriptions                   Last Seen
────────────── ─────── ─────── ───────────────────────────── ─────────────────────────
 kali-linux      agent   linux   system,linux,kali             2025-12-02 14:00:00
 centos-server   agent   linux   system,linux,centos           2025-12-02 14:00:05
=> Access Web Dashboard Open: http://YOUR_UBUNTU_IP:3000

User: admin

Pass: StrongPassword123

🔧 Troubleshooting
=> Kali Linux GPG Error If you see "weak digest" or "SHA-1" errors, ensure you followed Part 2 Step 1 exactly. Do NOT use apt-key add. The gpg --dearmor method is the only way to support modern Kali security standards.

=> Agent Connection Refused

Check Firewall on Ubuntu: sudo ufw status (Ensure 8081 is ALLOW).

Check Network: ping UBUNTU_IP from the agent.

Check Logs: sudo journalctl -u sensu-agent -f.

=> Time Synchronization Sensu requires synchronized clocks.

Bash

# Run on all servers
date
# If incorrect, install chrony
sudo apt install chrony -y
📸 Screenshots
Sensu Web UI Dashboard
Real-time view of Ubuntu, Kali, and CentOS nodes.

Event Monitoring
Active alerts for CPU and Disk usage.

🤝 Contributing
Fork the repository

Create a feature branch

Submit a Pull Request

👨‍💻 Author
Saleem Ali

LinkedIn: linkedin.com/in/saleem-ali-189719325

GitHub: github.com/ali4210

Currently studying AIOps at Al-Nafi International College

🙏 Acknowledgments
Sensu Documentation

Al-Nafi International College

Created for the DevOps Community.
