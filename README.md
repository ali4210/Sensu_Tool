# Sensu Monitoring Setup: Ubuntu Server + Kali Linux Agent

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Sensu](https://img.shields.io/badge/Sensu-6.11.0-green.svg)](https://sensu.io/)
[![Platform](https://img.shields.io/badge/Platform-Ubuntu%20%7C%20Kali-blue.svg)](https://github.com)

A complete guide to setting up Sensu monitoring infrastructure with Ubuntu as the backend server and Kali Linux as a monitored agent. This project demonstrates enterprise-grade monitoring capabilities for cybersecurity and system administration environments.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Part 1: Ubuntu Backend Setup](#part-1-ubuntu-backend-setup)
  - [Part 2: Kali Linux Agent Setup](#part-2-kali-linux-agent-setup)
- [Configuration](#configuration)
- [Verification](#verification)
- [Monitoring Checks](#monitoring-checks)
- [Troubleshooting](#troubleshooting)
- [Screenshots](#screenshots)
- [Contributing](#contributing)
- [License](#license)

## 🎯 Overview

This project implements a distributed monitoring solution using Sensu, where:
- **Ubuntu Server** acts as the Sensu backend (monitoring server)
- **Kali Linux** runs the Sensu agent (monitored client)
- Real-time monitoring of system metrics, resources, and health status
- Web-based dashboard for visualization and management

### Key Features

✅ Real-time system monitoring  
✅ Automated keepalive checks  
✅ Custom health checks (CPU, Memory, Disk)  
✅ Web UI dashboard access  
✅ Secure authentication between agent and backend  
✅ Compatible with Kali Linux (SHA-1 restrictions handled)

## 🏗️ Architecture

```
┌─────────────────────┐         WebSocket (port 8081)        ┌─────────────────────┐
│   Ubuntu Server     │◄────────────────────────────────────►│   Kali Linux        │
│                     │                                       │                     │
│  Sensu Backend      │         Authentication               │   Sensu Agent       │
│  - API (8080)       │         (user/password)              │   - Keepalive       │
│  - WebSocket (8081) │                                       │   - Metrics         │
│  - Web UI (3000)    │                                       │   - Health Checks   │
└─────────────────────┘                                       └─────────────────────┘
         │
         │
         ▼
    Monitoring Data
    Events & Alerts
```

## 📦 Prerequisites

### Ubuntu Server Requirements
- Ubuntu 20.04 LTS or higher
- Minimum 2GB RAM
- Root or sudo access
- Open ports: 8080, 8081, 3000

### Kali Linux Requirements
- Kali Linux (latest rolling release)
- Minimum 1GB RAM
- Root or sudo access
- Network connectivity to Ubuntu server

## 🚀 Installation

### Part 1: Ubuntu Backend Setup

#### Step 1: Add Sensu Repository

```bash
# Update system
sudo apt-get update

# Install required packages
sudo apt-get install -y curl gnupg2 apt-transport-https

# Add Sensu GPG key
curl -s https://packagecloud.io/sensu/stable/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/sensu-archive-keyring.gpg

# Add Sensu repository
echo "deb [signed-by=/usr/share/keyrings/sensu-archive-keyring.gpg] https://packagecloud.io/sensu/stable/ubuntu/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/sensu.list

# Update package list
sudo apt-get update
```

#### Step 2: Install Sensu Backend

```bash
# Install Sensu backend
sudo apt-get install -y sensu-go-backend

# Initialize the backend
sudo sensu-backend init

# Start and enable service
sudo systemctl start sensu-backend
sudo systemctl enable sensu-backend

# Verify status
sudo systemctl status sensu-backend
```

#### Step 3: Configure Sensu CLI

```bash
# Install sensuctl
sudo apt-get install -y sensu-go-cli

# Configure sensuctl
sensuctl configure -n \
  --username 'admin' \
  --password 'P@ssw0rd!' \
  --namespace default \
  --url 'http://127.0.0.1:8080'

# Verify connection
sensuctl cluster health
```

#### Step 4: Create Agent User

```bash
# Create dedicated agent user
sensuctl user create agent \
  --password 'SecureAgentPassword123!' \
  --interactive=false
```

#### Step 5: Configure Firewall

```bash
# Allow required ports
sudo ufw allow 8080/tcp  # API
sudo ufw allow 8081/tcp  # WebSocket
sudo ufw allow 3000/tcp  # Web UI
```

### Part 2: Kali Linux Agent Setup

#### Step 1: Download Sensu Agent Binary

```bash
# Create directories
sudo mkdir -p /usr/local/bin
sudo mkdir -p /etc/sensu
sudo mkdir -p /var/cache/sensu/sensu-agent
sudo mkdir -p /var/lib/sensu

# Download Sensu agent
curl -LO https://s3-us-west-2.amazonaws.com/sensu.io/sensu-go/6.11.0/sensu-go_6.11.0_linux_amd64.tar.gz

# Extract binary
tar -xvzf sensu-go_6.11.0_linux_amd64.tar.gz

# Move binary to system path
sudo mv sensu-agent /usr/local/bin/

# Make executable
sudo chmod +x /usr/local/bin/sensu-agent
```

#### Step 2: Create Agent Configuration

```bash
# Create configuration file
sudo nano /etc/sensu/agent.yml
```

Add the following configuration (replace `YOUR_UBUNTU_IP` with your Ubuntu server's IP):

```yaml
---
backend-url:
  - "ws://YOUR_UBUNTU_IP:8081"

namespace: "default"

subscriptions:
  - system
  - kali-linux

name: "kali-agent"

labels:
  environment: "monitoring"
  os: "kali"

cache-dir: "/var/cache/sensu/sensu-agent"

# Authentication
user: "agent"
password: "SecureAgentPassword123!"
```

#### Step 3: Create Systemd Service

```bash
# Create service file
sudo nano /etc/systemd/system/sensu-agent.service
```

Add this content:

```ini
[Unit]
Description=Sensu Agent
After=network.target

[Service]
Type=simple
User=root
Group=root
ExecStart=/usr/local/bin/sensu-agent start --config-file /etc/sensu/agent.yml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### Step 4: Start Sensu Agent

```bash
# Reload systemd
sudo systemctl daemon-reload

# Start agent
sudo systemctl start sensu-agent

# Enable on boot
sudo systemctl enable sensu-agent

# Check status
sudo systemctl status sensu-agent

# View logs
sudo journalctl -u sensu-agent -f
```

## ⚙️ Configuration

### Agent Authentication

The agent uses password authentication to connect to the backend. Ensure the credentials match:

**Backend User Creation:**
```bash
sensuctl user create agent --password 'YourPassword'
```

**Agent Configuration:**
```yaml
user: "agent"
password: "YourPassword"
```

### Custom Subscriptions

Modify subscriptions in `/etc/sensu/agent.yml`:

```yaml
subscriptions:
  - system
  - linux
  - kali
  - security-tools
```

## ✅ Verification

### Check Agent Connection (Ubuntu Server)

```bash
# List all entities
sensuctl entity list

# Get detailed agent info
sensuctl entity info kali-agent

# View keepalive events
sensuctl event list
```

Expected output:
```
Name         Class    OS           Subscriptions                  Last Seen
kali-agent   agent    linux        system,kali-linux              2025-12-02 05:17:54 +0600 +06
```

### Check Agent Status (Kali Linux)

```bash
# View agent logs
sudo journalctl -u sensu-agent -n 50

# Check service status
sudo systemctl status sensu-agent
```

Look for: `"msg":"successfully connected"`

### Access Web UI

Open browser and navigate to: `http://YOUR_UBUNTU_IP:3000`

**Default Credentials:**
- Username: `admin`
- Password: `P@ssw0rd!`

## 📊 Monitoring Checks

### Create CPU Monitoring Check

```bash
sensuctl check create check-cpu \
  --command 'top -bn1 | grep "Cpu(s)" | awk '"'"'{print $2}'"'"' | cut -d% -f1' \
  --interval 30 \
  --subscriptions system \
  --handlers default
```

### Create Memory Monitoring Check

```bash
sensuctl check create check-memory \
  --command 'free -m | grep Mem | awk '"'"'{printf "%.2f", ($3/$2)*100}'"'"'' \
  --interval 30 \
  --subscriptions system \
  --handlers default
```

### Create Disk Usage Check

```bash
sensuctl check create check-disk \
  --command 'df -h / | tail -1 | awk '"'"'{print $5}'"'"' | cut -d% -f1' \
  --interval 60 \
  --subscriptions system \
  --handlers default
```

### List All Checks

```bash
sensuctl check list
```

## 🔧 Troubleshooting

### Agent Connection Issues

**Problem:** `handshake failed with status 401: bad credentials`

**Solution:**
1. Verify user credentials on backend
2. Update agent configuration with correct password
3. Restart agent service

```bash
sudo systemctl restart sensu-agent
sudo journalctl -u sensu-agent -f
```

### Backend Not Reachable

**Problem:** Agent cannot connect to backend

**Solution:**
```bash
# Test connectivity
ping YOUR_UBUNTU_IP
telnet YOUR_UBUNTU_IP 8081

# Check firewall
sudo ufw status

# Verify backend is running
sudo systemctl status sensu-backend
```

### View Detailed Logs

**Ubuntu Backend:**
```bash
sudo journalctl -u sensu-backend -n 100 --no-pager
```

**Kali Agent:**
```bash
sudo journalctl -u sensu-agent -n 100 --no-pager
```

## 📸 Screenshots

### Sensu Web UI Dashboard
![Dashboard](screenshots/dashboard.png)

### Agent Status
![Agent Status](screenshots/agent-status.png)

### Event Monitoring
![Events](screenshots/events.png)

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add new feature'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Create a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 👨‍💻 Author

**Saleem Ali**
- LinkedIn: [linkedin.com/in/saleem-ali-189719325](https://www.linkedin.com/in/saleem-ali-189719325/)
- GitHub: [github.com/ali4210](https://github.com/ali4210)
- Currently studying AIOps at Al-Nafi International College

## 🙏 Acknowledgments

- [Sensu Documentation](https://docs.sensu.io/)
- [Ubuntu Server](https://ubuntu.com/server)
- [Kali Linux](https://www.kali.org/)
- Al-Nafi International College for educational support

## 📚 Additional Resources

- [Sensu Official Documentation](https://docs.sensu.io/)
- [Sensu Community Forums](https://discourse.sensu.io/)
- [Sensu GitHub Repository](https://github.com/sensu/sensu-go)

---

**Project Status:** ✅ Active and Maintained

**Last Updated:** December 2025

If you found this project helpful, please ⭐ star this repository!
