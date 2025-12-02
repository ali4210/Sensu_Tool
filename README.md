markdown
# 🔍 Sensu Monitoring Infrastructure: Multi-Platform Deployment Guide

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Sensu Go Version](https://img.shields.io/badge/Sensu_Go-6.13.0+-green.svg?logo=sensu&logoColor=white)](https://sensu.io/)
[![Platform Support](https://img.shields.io/badge/Platform-Ubuntu%20%7C%20Kali%20%7C%20CentOS-blue.svg?logo=linux)](https://github.com)
[![Database: PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL_13+-336791.svg?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Status: Production Ready](https://img.shields.io/badge/Status-Production_Ready-brightgreen.svg)](https://github.com)

A comprehensive, battle-tested guide for deploying a **production-grade Sensu monitoring infrastructure** across heterogeneous environments. This repository provides step-by-step instructions for setting up a central Sensu backend on Ubuntu Server with **PostgreSQL database integration**, and monitored agents on Kali Linux (with GPG security fixes) and CentOS systems.

## 🚀 Quick Start

```bash
# Clone and deploy
git clone https://github.com/yourusername/sensu-monitoring-infra.git
cd sensu-monitoring-infra

# Review architecture and prerequisites
cat ARCHITECTURE.md

# Deploy using our automated scripts (optional)
./deploy-backend.sh
📊 Architecture Overview



















✨ Key Features
🔧 Multi-Platform Support: Complete setup guides for Ubuntu, Kali Linux, and CentOS

🗄️ PostgreSQL Integration: Production-grade database setup for Sensu backend

🛡️ Security-First Approach: GPG/SHA-1 workaround for modern Kali Linux distributions

📈 Production Ready: Includes monitoring checks for CPU, memory, and disk usage

🔌 Asset Management: Utilizes official Sensu assets for professional-grade monitoring

🎯 Agent Segmentation: Logical subscription-based grouping for targeted monitoring

📊 Visual Dashboard: Sensu Web UI for real-time monitoring and alerting

📋 Prerequisites
Hardware Requirements
Component	Minimum	Recommended
Backend Server	2 CPU, 4GB RAM	4 CPU, 8GB RAM
PostgreSQL Database	2 CPU, 4GB RAM	4 CPU, 8GB RAM
Agent Nodes	1 CPU, 2GB RAM	2 CPU, 4GB RAM
Storage	20GB free space	50GB free space
Database Requirements
PostgreSQL: Version 13+ (recommended for Sensu)

Storage: 20GB+ for metrics data

Memory: 4GB+ dedicated for PostgreSQL

Backup: Regular backup strategy implementation

Network Requirements
Backend Server: Static IP address

Open Ports:

8080 (Sensu API)

8081 (Agent WebSocket)

3000 (Web UI Dashboard)

5432 (PostgreSQL, if remote access needed)

Network Latency: < 100ms between agents and backend

Software Requirements
Ubuntu Server: 20.04 LTS or 22.04 LTS

Kali Linux: 2022.3+ (with GPG security patch)

CentOS: 7 or 8

PostgreSQL: 13+ for Sensu compatibility

Docker: Optional, for containerized deployment

🛠️ Installation Guide
Part 1: Ubuntu Backend (Home Server)
Step 1: System Preparation
bash
# Update system and install prerequisites
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg2 ca-certificates software-properties-common

# Configure hostname (optional but recommended)
sudo hostnamectl set-hostname sensu-backend
Step 2: Install Sensu Backend
bash
# Add Sensu repository
curl -s https://packagecloud.io/install/repositories/sensu/stable/script.deb.sh | sudo bash

# Install Sensu backend
sudo apt install sensu-go-backend -y

# Verify installation
sensu-backend version
Step 3: Initialize Sensu Backend
bash
# Set environment variables for initialization
export SENSU_BACKEND_CLUSTER_ADMIN_USERNAME=admin
export SENSU_BACKEND_CLUSTER_ADMIN_PASSWORD=$(openssl rand -base64 16)
export SENSU_BACKEND_CLUSTER_ADMIN_PASSWORD='StrongPassword123'  # Or use generated

# Initialize the backend
sudo --preserve-env sensu-backend init

# Save credentials securely
echo "Username: admin" > ~/sensu-credentials.txt
echo "Password: $SENSU_BACKEND_CLUSTER_ADMIN_PASSWORD" >> ~/sensu-credentials.txt
chmod 600 ~/sensu-credentials.txt
Step 4: Configure Services
bash
# Enable and start the backend service
sudo systemctl enable sensu-backend
sudo systemctl start sensu-backend
sudo systemctl status sensu-backend

# Configure firewall
sudo ufw allow 8080/tcp comment 'Sensu API'
sudo ufw allow 8081/tcp comment 'Agent WebSocket'
sudo ufw allow 3000/tcp comment 'Web UI'
sudo ufw reload
Part 2: PostgreSQL Database Setup for Sensu
Why PostgreSQL with Sensu?
While Sensu comes with embedded etcd by default, PostgreSQL offers:

Better persistence and reliability

Standard SQL interface for queries

Enterprise backup/recovery capabilities

Better performance for large deployments

Familiar tooling for DBAs

Step 1: Install PostgreSQL on Ubuntu Backend
bash
# Install PostgreSQL 13 (or latest stable)
sudo apt update
sudo apt install -y postgresql postgresql-contrib postgresql-client

# Check PostgreSQL version
psql --version

# Start and enable PostgreSQL
sudo systemctl enable --now postgresql
sudo systemctl status postgresql
Step 2: Configure PostgreSQL for Sensu
bash
# Switch to postgres user
sudo -i -u postgres

# Create Sensu database and user
psql -c "CREATE USER sensu WITH PASSWORD 'StrongDBPassword123!';"
psql -c "CREATE DATABASE sensu_db OWNER sensu;"
psql -c "GRANT ALL PRIVILEGES ON DATABASE sensu_db TO sensu;"

# For production, adjust connection limits
psql -d sensu_db -c "ALTER USER sensu WITH CONNECTION LIMIT 100;"

# Exit postgres user
exit
Step 3: Configure PostgreSQL Authentication
bash
# Edit PostgreSQL authentication file
sudo nano /etc/postgresql/13/main/pg_hba.conf

# Add these lines at the end (adjust IP ranges as needed):
host    sensu_db        sensu           127.0.0.1/32            md5
host    sensu_db        sensu           192.168.1.0/24          md5
host    sensu_db        sensu           ::1/128                 md5

# Enable remote connections (optional for multi-node)
sudo nano /etc/postgresql/13/main/postgresql.conf

# Uncomment and modify:
listen_addresses = '*'  # Or 'localhost,192.168.1.100'
port = 5432
max_connections = 200

# Restart PostgreSQL
sudo systemctl restart postgresql
Step 4: Configure Sensu Backend with PostgreSQL
bash
# Stop Sensu backend before configuration
sudo systemctl stop sensu-backend

# Create Sensu configuration directory
sudo mkdir -p /etc/sensu

# Create backend configuration file with PostgreSQL settings
sudo tee /etc/sensu/backend.yml << 'EOF'
# PostgreSQL Datastore Configuration
datastore:
  postgresql:
    host: "localhost"
    port: 5432
    user: "sensu"
    password: "StrongDBPassword123!"
    database: "sensu_db"
    sslmode: "disable"  # For production: "require" or "verify-full"
    pool_size: 20
    statement_timeout: 5000

# Sensu Backend Configuration
backend:
  state-dir: "/var/lib/sensu/sensu-backend"
  log-level: "info"
  api-listen-address: "0.0.0.0:8080"
  agent-ws-listen-address: "0.0.0.0:8081"
  agent-tcp-listen-address: "0.0.0.0:3031"
  dashboard-listen-address: "0.0.0.0:3000"

# Event Logging
event-log-file: "/var/log/sensu/sensu-backend/events.log"
EOF

# Set proper permissions
sudo chown sensu:sensu /etc/sensu/backend.yml
sudo chmod 600 /etc/sensu/backend.yml

# Start Sensu with PostgreSQL
sudo systemctl start sensu-backend

# Verify PostgreSQL connection
sudo journalctl -u sensu-backend -n 20 --no-pager | grep -i postgres
Step 5: Verify PostgreSQL Integration
bash
# Check if Sensu created necessary tables
PGPASSWORD='StrongDBPassword123!' psql -h localhost -U sensu -d sensu_db -c "\dt"

# Check Sensu backend logs for PostgreSQL connection
sudo journalctl -u sensu-backend -n 50 --no-pager | grep -i postgres

# Verify Sensu is using PostgreSQL
sensuctl cluster health --format json | jq '.'
Part 3: Kali Linux Agent (With GPG Fix)
Step 1: Resolve GPG SHA-1/Weak Digest Error
bash
# Method 1: Manual GPG Key Installation (Recommended)
sudo apt update
sudo apt install -y curl gnupg2 debian-archive-keyring

# Download and convert GPG key to avoid SHA-1 error
curl -fsSL https://packagecloud.io/sensu/stable/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/sensu-archive-keyring.gpg

# Add repository with explicit keyring reference
echo "deb [signed-by=/usr/share/keyrings/sensu-archive-keyring.gpg] \
  https://packagecloud.io/sensu/stable/debian/ buster main" | \
  sudo tee /etc/apt/sources.list.d/sensu.list

# Method 2: Alternative approach for newer Kali versions
echo "deb [arch=amd64] https://packagecloud.io/sensu/stable/debian/ any main" | \
  sudo tee /etc/apt/sources.list.d/sensu.list
  
curl -L https://packagecloud.io/sensu/stable/gpgkey | sudo apt-key add -
Step 2: Install and Configure Agent
bash
# Update and install agent
sudo apt update
sudo apt install sensu-go-agent -y

# Create agent configuration
sudo tee /etc/sensu/agent.yml << EOF
---
name: "kali-$(hostname)"
namespace: "default"
subscriptions:
  - system
  - linux
  - kali
  - development
backend-url:
  - "ws://YOUR_UBUNTU_IP:8081"
cache-dir: "/var/cache/sensu/sensu-agent"
config-file: "/etc/sensu/agent.yml"
labels:
  environment: "development"
  os: "kali-linux"
annotations:
  project: "Sensu Monitoring"
  maintainer: "Ops Team"
EOF

# Replace with your Ubuntu server IP
sudo sed -i "s/YOUR_UBUNTU_IP/192.168.1.100/g" /etc/sensu/agent.yml
Step 3: Start and Verify Agent
bash
# Enable and start agent service
sudo systemctl enable sensu-agent
sudo systemctl start sensu-agent

# Check service status
sudo systemctl status sensu-agent

# View agent logs
sudo journalctl -u sensu-agent -f --no-pager
Part 4: CentOS Agent
Step 1: Install Sensu Agent
bash
# Add Sensu repository for CentOS/RHEL
curl -s https://packagecloud.io/install/repositories/sensu/stable/script.rpm.sh | sudo bash

# Install agent
sudo yum install sensu-go-agent -y

# Alternative for CentOS 8/RHEL 8
sudo dnf install sensu-go-agent -y
Step 2: Configure Agent
bash
# Create configuration file
sudo tee /etc/sensu/agent.yml << EOF
---
name: "centos-$(hostname | cut -d. -f1)"
namespace: "default"
subscriptions:
  - system
  - linux
  - centos
  - production
backend-url:
  - "ws://YOUR_UBUNTU_IP:8081"
cache-dir: "/var/cache/sensu/sensu-agent"
config-file: "/etc/sensu/agent.yml"
labels:
  environment: "production"
  os: "centos"
  role: "webserver"
annotations:
  department: "IT Operations"
  sla: "24x7"
EOF

# Update IP address
sudo sed -i "s/YOUR_UBUNTU_IP/192.168.1.100/g" /etc/sensu/agent.yml
Step 3: Manage Agent Service
bash
# Start and enable service
sudo systemctl enable sensu-agent
sudo systemctl start sensu-agent

# Configure SELinux if enabled
sudo setsebool -P sensu_agent_can_network 1

# Open firewall for agent
sudo firewall-cmd --permanent --add-port=3030/tcp
sudo firewall-cmd --reload
Part 5: Sensu CLI Configuration
Configure sensuctl on Backend
bash
# Install sensuctl if not already installed
sudo apt install sensu-go-cli -y

# Configure sensuctl
sensuctl configure -n \
  --username 'admin' \
  --password 'StrongPassword123' \
  --namespace 'default' \
  --url 'http://localhost:8080' \
  --format 'tabular' \
  --timeout '30s'

# Verify configuration
sensuctl config view

# Test connection
sensuctl cluster health
Create Dedicated Agent User
bash
# Create agent user with limited permissions
sensuctl user create agent \
  --password 'AgentPassword123' \
  --groups 'agents'

# Create role for agents
sensuctl create -f - << EOF
{
  "type": "Role",
  "api_version": "core/v2",
  "metadata": {
    "name": "agent"
  },
  "rules": [
    {
      "verbs": ["get", "list"],
      "resources": ["events", "entities"],
      "resource_names": []
    }
  ]
}
EOF
📦 Asset Management
Install Essential Monitoring Assets
bash
# CPU monitoring
sensuctl asset add sensu/check-cpu-usage -r check-cpu-usage

# Memory monitoring
sensuctl asset add sensu/check-memory-usage -r check-memory-usage

# Disk monitoring
sensuctl asset add sensu/check-disk-usage -r check-disk-usage

# Process monitoring
sensuctl asset add sensu/check-processes -r check-processes

# Network monitoring
sensuctl asset add sensu/check-network-interface -r check-network

# PostgreSQL monitoring
sensuctl asset add sensu/postgresql-check -r postgresql-check

# List all assets
sensuctl asset list --format yaml
⚙️ Monitoring Checks Configuration
System Health Checks
bash
# CPU Check (Triggers warning at 75%, critical at 90%)
sensuctl check create check-cpu \
  --command 'check-cpu-usage -w 75 -c 90' \
  --interval 30 \
  --subscriptions system \
  --runtime-assets check-cpu-usage \
  --timeout 10 \
  --handlers slack,email \
  --publish true \
  --severity warning

# Memory Check
sensuctl check create check-memory \
  --command 'check-memory-usage -w 80 -c 90' \
  --interval 60 \
  --subscriptions system \
  --runtime-assets check-memory-usage \
  --labels "team=operations" \
  --annotations "runbook=https://wiki.example.com/runbooks/memory"

# Disk Check with custom thresholds
sensuctl check create check-disk \
  --command 'check-disk-usage -w 80 -c 90 -p / -p /var -p /home' \
  --interval 300 \
  --subscriptions system \
  --runtime-assets check-disk-usage \
  --ttl 600
PostgreSQL Database Monitoring
bash
# PostgreSQL Health Check
sensuctl check create postgresql-health \
  --command 'postgresql-check -H localhost -u sensu -p StrongDBPassword123! -d sensu_db' \
  --interval 60 \
  --subscriptions database \
  --runtime-assets postgresql-check \
  --handlers slack,email \
  --labels "service=postgresql" \
  --annotations "runbook=https://wiki.example.com/runbooks/postgresql"

# PostgreSQL Connection Monitoring
sensuctl check create postgresql-connections \
  --command 'postgresql-check -H localhost -u sensu -p StrongDBPassword123! --check-connections -w 150 -c 180' \
  --interval 120 \
  --subscriptions database \
  --runtime-assets postgresql-check

# PostgreSQL Database Size Monitoring
sensuctl check create postgresql-disk \
  --command 'check-disk-usage -w 80 -c 90 -p /var/lib/postgresql' \
  --interval 300 \
  --subscriptions database \
  --runtime-assets check-disk-usage \
  --handlers pagerduty
🗄️ PostgreSQL Maintenance & Backup
Database Optimization
bash
# Regular vacuum and analyze (add to cron)
PGPASSWORD='StrongDBPassword123!' psql -h localhost -U sensu -d sensu_db -c "VACUUM ANALYZE;"

# Check database size
PGPASSWORD='StrongDBPassword123!' psql -h localhost -U sensu -d sensu_db -c "
SELECT pg_database.datname, 
       pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
WHERE pg_database.datname = 'sensu_db';
"
Backup and Recovery Script
bash
#!/bin/bash
# save as: /usr/local/bin/sensu-postgres-backup.sh

BACKUP_DIR="/backup/sensu-postgres"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup PostgreSQL database
PGPASSWORD='StrongDBPassword123!' pg_dump -h localhost -U sensu -d sensu_db \
  -F c -b -v -f "$BACKUP_DIR/sensu_db_$DATE.backup"

# Backup Sensu configuration
tar czf "$BACKUP_DIR/sensu_config_$DATE.tar.gz" /etc/sensu/

# Remove old backups
find $BACKUP_DIR -name "*.backup" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

# Log backup completion
echo "Backup completed: $BACKUP_DIR/sensu_db_$DATE.backup" >> /var/log/sensu-backup.log

# Add to cron (daily at 2 AM)
# sudo crontab -e
# 0 2 * * * /usr/local/bin/sensu-postgres-backup.sh
Restore from Backup
bash
# Stop Sensu backend
sudo systemctl stop sensu-backend

# Drop and recreate database
sudo -i -u postgres psql -c "DROP DATABASE sensu_db;"
sudo -i -u postgres psql -c "CREATE DATABASE sensu_db OWNER sensu;"

# Restore from backup
PGPASSWORD='StrongDBPassword123!' pg_restore -h localhost -U sensu -d sensu_db \
  -v "/backup/sensu-postgres/sensu_db_20241202_0200.backup"

# Restart Sensu
sudo systemctl start sensu-backend

# Verify restoration
sensuctl entity list
sensuctl check list
🔍 Verification and Testing
Verify Agent Connectivity
bash
# List all entities (agents)
sensuctl entity list --format json | jq '.[] | {name: .metadata.name, subscriptions: .subscriptions}'

# Check specific agent status
sensuctl entity info kali-linux --format yaml

# Test agent subscription matching
sensuctl event list --subscriptions kali
Test Monitoring Checks
bash
# Manually execute a check on an agent
sensuctl check execute check-cpu kali-linux

# View events from checks
sensuctl event list --format table

# Filter events by status
sensuctl event list --format json | jq '.[] | select(.check.status != 0)'
System Health Verification
bash
# Check Sensu backend health
sensuctl cluster health --format json

# Check PostgreSQL connection
PGPASSWORD='StrongDBPassword123!' psql -h localhost -U sensu -d sensu_db -c "SELECT 1 as connection_test;"

# Check service logs
sudo journalctl -u sensu-backend -f --no-pager
sudo journalctl -u sensu-agent -f --no-pager
🌐 Web Dashboard Access
Access Sensu Web UI
bash
# Determine your server IP
ip addr show | grep inet

# Access the dashboard
echo "Sensu Web UI: http://$(hostname -I | awk '{print $1}'):3000"
echo "Username: admin"
echo "Password: $(cat ~/sensu-credentials.txt 2>/dev/null | grep Password | cut -d' ' -f2)"
Dashboard Features
Entity Viewer: Real-time agent status

Event Management: View and silence alerts

Check Configuration: Manage monitoring checks

Role-Based Access: Secure multi-user access

Dark/Light Theme: Customizable interface

🚨 Troubleshooting Guide
PostgreSQL Connection Issues
bash
# Check PostgreSQL is running
sudo systemctl status postgresql

# Check listening ports
sudo ss -tlnp | grep 5432

# Check pg_hba.conf configuration
sudo cat /etc/postgresql/13/main/pg_hba.conf | grep sensu

# Test PostgreSQL connection
PGPASSWORD='StrongDBPassword123!' psql -h localhost -U sensu -d sensu_db -c "SELECT 1;"
Kali Linux GPG/SHA-1 Error
bash
# Error: "Weak digest algorithm (SHA1)"
# Solution: Use manual key installation method

# Remove existing configuration
sudo rm /etc/apt/sources.list.d/sensu.list
sudo rm /usr/share/keyrings/sensu-archive-keyring.gpg 2>/dev/null || true

# Reinstall using the secure method
curl -fsSL https://packagecloud.io/sensu/stable/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/sensu-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/sensu-archive-keyring.gpg] \
  https://packagecloud.io/sensu/stable/debian/ any main" | \
  sudo tee /etc/apt/sources.list.d/sensu.list
Agent Connection Issues
bash
# Check network connectivity
ping YOUR_UBUNTU_IP
telnet YOUR_UBUNTU_IP 8081

# Verify agent configuration
sensu-agent config view

# Check backend firewall
sudo ufw status verbose

# Debug agent connection
sudo sensu-agent start --log-level debug
Database Performance Issues
bash
# Check slow queries in PostgreSQL
PGPASSWORD='StrongDBPassword123!' psql -h localhost -U sensu -d sensu_db -c "
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
"

# Check locks in PostgreSQL
PGPASSWORD='StrongDBPassword123!' psql -h localhost -U sensu -d sensu_db -c "
SELECT pid, usename, pg_blocking_pids(pid) as blocked_by, query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
"
📈 Performance Tuning
PostgreSQL Performance Optimization
bash
# Edit PostgreSQL configuration
sudo nano /etc/postgresql/13/main/postgresql.conf

# Recommended settings for Sensu (adjust based on your RAM):
shared_buffers = 1GB                    # 25% of available RAM
work_mem = 16MB                         # For sorting operations
maintenance_work_mem = 256MB            # For VACUUM, CREATE INDEX
effective_cache_size = 3GB              # ~75% of available RAM
max_connections = 200                   # Sufficient for Sensu
checkpoint_completion_target = 0.9      # Smooth out checkpoints
wal_buffers = 16MB                      # Default is fine
default_statistics_target = 100         # Better query planning

# For write-heavy Sensu deployments:
max_wal_size = 2GB
min_wal_size = 1GB

# Restart PostgreSQL
sudo systemctl restart postgresql
🔄 Maintenance Operations
Backup and Restore
bash
# Backup etcd data (if still using etcd)
sensu-backend etcd snapshot save /backup/sensu-backup.db

# Restore from backup
sensu-backend etcd snapshot restore /backup/sensu-backup.db \
  --data-dir /var/lib/sensu/sensu-backend/etcd
Version Upgrade
bash
# Check current version
sensu-backend version
psql --version

# Upgrade procedure
sudo apt update
sudo apt install sensu-go-backend postgresql-13
sudo systemctl restart sensu-backend postgresql

# Verify upgrade
sensu-backend version
psql --version
sensuctl cluster health
🤝 Contributing
We welcome contributions! Please see our Contributing Guidelines for details.

Fork the repository

Create a feature branch (git checkout -b feature/AmazingFeature)

Commit your changes (git commit -m 'Add some AmazingFeature')

Push to the branch (git push origin feature/AmazingFeature)

Open a Pull Request

📝 License
This project is licensed under the MIT License - see the LICENSE file for details.

👨‍💻 Author
Saleem Ali - DevOps & AIOps Specialist

LinkedIn: linkedin.com/in/saleem-ali-189719325

GitHub: github.com/ali4210

Portfolio: saleem-ali.dev

Currently studying AIOps at Al-Nafi International College

🙏 Acknowledgments
Sensu Team for their excellent monitoring platform

PostgreSQL Team for robust database solutions

Al-Nafi International College for educational support

The DevOps community for best practices and inspiration
