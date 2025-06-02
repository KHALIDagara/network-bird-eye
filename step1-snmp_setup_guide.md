# Complete SNMP Exporter Setup Guide - Tested & Working

This guide covers the complete setup process for Prometheus SNMP Exporter v0.29.0, including common issues and solutions we encountered.

## Prerequisites

```bash
# Install required tools
sudo apt-get update
sudo apt-get install -y curl wget tar snmp snmp-mibs-downloader
```

## Step 1: Download and Install SNMP Exporter

```bash
# Get the latest version
SNMP_VER=$(curl -sL https://api.github.com/repos/prometheus/snmp_exporter/releases/latest \
    | grep -Po '"tag_name": "v\K[^"]*')

echo "Latest version: v$SNMP_VER"

# Download the release
wget https://github.com/prometheus/snmp_exporter/releases/download/v${SNMP_VER}/snmp_exporter-${SNMP_VER}.linux-amd64.tar.gz

# Extract
tar zxvf snmp_exporter-${SNMP_VER}.linux-amd64.tar.gz

# Install binary
sudo cp snmp_exporter-${SNMP_VER}.linux-amd64/snmp_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/snmp_exporter

# Verify installation
/usr/local/bin/snmp_exporter --version
```

## Step 2: Create System User
from what i understand you have to create a system user in order to have the access to the data provided by snmp clients 
```bash
# Create dedicated user
sudo useradd --no-create-home --shell /usr/sbin/nologin snmpusr
```

## Step 3: Create Configuration Directory and File

**CRITICAL**: Use the NEW configuration format (v0.23.0+). The old format will NOT work.

```bash
# Create config directory
sudo mkdir -p /etc/snmp_exporter

# Create configuration file with NEW format
sudo tee /etc/snmp_exporter/snmp.yml > /dev/null <<'EOF'
# SNMP Exporter Configuration - NEW FORMAT (v0.23.0+)
auths:
  public_v2:
    community: public
    security_level: noAuthNoPriv
    version: 2

modules:
  if_mib:
    walk:
      - 1.3.6.1.2.1.2      # ifTable - Interface statistics
      - 1.3.6.1.2.1.31     # ifXTable - Extended interface statistics
      
  system:
    walk:
      - 1.3.6.1.2.1.1      # System information
      
  basic:
    walk:
      - 1.3.6.1.2.1.1      # System info
      - 1.3.6.1.2.1.2      # Interface table
      
  network_device:
    walk:
      - 1.3.6.1.2.1.1      # System
      - 1.3.6.1.2.1.2      # Interfaces
      - 1.3.6.1.2.1.31     # Extended interfaces
      - 1.3.6.1.2.1.25.1   # Host resources
EOF

# Set proper permissions
sudo chown -R snmpusr:snmpusr /etc/snmp_exporter
sudo chmod 644 /etc/snmp_exporter/snmp.yml
```

## Step 4: Create Systemd Service
##### it's very essential to create Systemd service DO NOT IGNORE THIS PART 
```bash
sudo tee /etc/systemd/system/snmp_exporter.service > /dev/null <<'EOF'
[Unit]
Description=Prometheus SNMP Exporter
Documentation=https://github.com/prometheus/snmp_exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=snmpusr
Group=snmpusr
ExecStart=/usr/local/bin/snmp_exporter --config.file=/etc/snmp_exporter/snmp.yml --web.listen-address=:9116
SyslogIdentifier=snmp_exporter
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
```

## Step 5: Start and Enable Service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable service
sudo systemctl enable snmp_exporter

# Start service
sudo systemctl start snmp_exporter

# Check status
sudo systemctl status snmp_exporter
```

## Step 6: Test Installation

```bash
# Test if service is listening
curl http://localhost:9116/metrics

# Test SNMP query (replace <DEVICE_IP> with your device)
curl "http://localhost:9116/snmp?target=<DEVICE_IP>&module=system"

# Test interface statistics
curl "http://localhost:9116/snmp?target=<DEVICE_IP>&module=if_mib"
```

## Step 7: Test Direct SNMP Connectivity

```bash
# Test direct SNMP access to your device
snmpwalk -v2c -c public <DEVICE_IP> 1.3.6.1.2.1.1.1.0

# Should return system description
```

## Step 8: Configure Firewall (if needed)

```bash
# Allow SNMP Exporter port
sudo ufw allow 9116/tcp
```

---

## Troubleshooting Guide

### Issue 1: Service Fails to Start - Configuration Error

**Symptoms:**
```
Error parsing config file
field auth not found in type config.plain
Possible version missmatch between generator and snmp_exporter
```

**Solution:**
Your config is using the OLD format. Use the NEW format above (separate `auths` and `modules` sections).

**Check logs:**
```bash
sudo journalctl -u snmp_exporter -n 50 --no-pager
```

### Issue 2: Permission Denied

**Symptoms:**
```
permission denied
```

**Solution:**
```bash
# Fix permissions
sudo chown -R snmpusr:snmpusr /etc/snmp_exporter
sudo chmod 644 /etc/snmp_exporter/snmp.yml
```

### Issue 3: Port Already in Use

**Symptoms:**
```
bind: address already in use
```

**Solution:**
```bash
# Check what's using port 9116
sudo lsof -i :9116

# Kill the process or restart the service
sudo systemctl restart snmp_exporter
```

### Issue 4: SNMP Device Not Responding

**Symptoms:**
```
context deadline exceeded
no response
```

**Solution:**
```bash
# Test direct SNMP connectivity first
snmpwalk -v2c -c public <DEVICE_IP> 1.3.6.1.2.1.1.1.0

# Check if device allows SNMP from your se
