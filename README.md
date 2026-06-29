# 📚 Complete SOAR PIPELINE build guide: From Zero to Production
![GitHub last commit](https://img.shields.io/github/last-commit/sonalkr31/SOC)
![GitHub repo size](https://img.shields.io/github/repo-size/sonalkr31/SOC)
![GitHub stars](https://img.shields.io/github/stars/sonalkr31/SOC?style=social)
![Twitter Follow](https://img.shields.io/twitter/follow/yourhandle?style=social)

## Table of Contents

1. Project Overview & Architecture
2. Hardware & Environment Setup
3. Phase 1: Ubuntu VM Setup
4. Phase 2: Docker & Shuffle Installation
5. Phase 3: Building the SOAR Workflow
6. Phase 4: The ARM64 Challenge & Solutions
7. Phase 5: Building the External Webhook Server
8. Phase 6: Testing & Verification
9. Phase 7: Security Layer Integration
10. Complete Code Reference
11. Troubleshooting Guide
12. Glossary

---

## 1. PROJECT OVERVIEW & ARCHITECTURE

### What is a SOAR Pipeline?

**SOAR** stands for **Security Orchestration, Automation, and Response**. It's a technology stack that:

| Component | Purpose |
| --- | --- |
| **Security** | Detects threats (Wazuh/SIEM) |
| **Orchestration** | Connects different tools (Shuffle) |
| **Automation** | Executes response automatically (Python) |
| **Response** | Takes action (Blocking IPs) |

### Why Build This Project?

**The Problem:**

- Hackers constantly try to brute force SSH passwords
- Manual monitoring is slow and error-prone
- Security teams are overwhelmed with alerts
- Attackers can compromise servers in minutes

**The Solution:**

- Automatically detect failed login attempts
- Instantly block malicious IP addresses
- 24/7 protection without human intervention
- Learn how enterprise security works

### Our Architecture (Final Working Version)

```text
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           SOAR PIPELINE ARCHITECTURE                                 │
│                                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐      │
│  │                     ATTACKER (Malicious IP)                             │      │
│  │                    192.168.100.50 (Test)                               │      │
│  └────────────────────────────┬────────────────────────────────────────────┘      │
│                               │                                                     │
│                               ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐      │
│  │                   1. SSH BRUTE FORCE ATTACK                            │      │
│  │                   (Simulated via failed logins)                         │      │
│  └────────────────────────────┬────────────────────────────────────────────┘      │
│                               │                                                     │
│                               ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐      │
│  │                   2. WAZUH SIEM (Not yet installed)                     │      │
│  │                   - Monitors /var/log/auth.log                          │      │
│  │                   - Detects Rule 5712 (SSH Brute Force)                 │      │
│  │                   - Generates JSON Alert                                │      │
│  └────────────────────────────┬────────────────────────────────────────────┘      │
│                               │                                                     │
│                               ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐      │
│  │                   3. SHUFFLE SOAR PLATFORM                              │      │
│  │                   (Runs in Docker on Ubuntu VM)                         │      │
│  │                                                                          │      │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │      │
│  │  │                    WORKFLOW: "Wazuh Incident Response"          │   │      │
│  │  │                                                                  │   │      │
│  │  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │   │      │
│  │  │  │   Webhook    │───▶│  Extract IP  │───▶│    HTTP      │      │   │      │
│  │  │  │   Trigger    │    │   (Python)   │    │   (Flask)    │      │   │      │
│  │  │  └──────────────┘    └──────────────┘    └──────────────┘      │   │      │
│  │  │        │                   │                   │                │   │      │
│  │  │        ▼                   ▼                   ▼                │   │      │
│  │  │  Receives Alert     Extracts IP         Calls Python            │   │      │
│  │  │  from Wazuh         from JSON           Script                  │   │      │
│  │  └─────────────────────────────────────────────────────────────────┘   │      │
│  └────────────────────────────┬────────────────────────────────────────────┘      │
│                               │                                                     │
│                               ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐      │
│  │                   4. FLASK WEBHOOK SERVER                               │      │
│  │                   (Python virtual environment)                          │      │
│  │                                                                          │      │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │      │
│  │  │  Receives POST request from Shuffle                             │   │      │
│  │  │  Calls block_ip.py script                                       │   │      │
│  │  │  Returns success/failure response                               │   │      │
│  │  └─────────────────────────────────────────────────────────────────┘   │      │
│  └────────────────────────────┬────────────────────────────────────────────┘      │
│                               │                                                     │
│                               ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────┐      │
│  │                   5. BLOCK_IP.PY SCRIPT                                │      │
│  │                   (Python on VM)                                       │      │
│  │                                                                          │      │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │      │
│  │  │  Blocks IP at THREE security layers:                            │   │      │
│  │  │                                                                  │   │      │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │   │      │
│  │  │  │ hosts.deny  │  │    UFW      │  │  iptables   │            │   │      │
│  │  │  │  System     │  │  Firewall   │  │   Kernel    │            │   │      │
│  │  │  │   Level     │  │    Level    │  │    Level    │            │   │      │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘            │   │      │
│  │  └─────────────────────────────────────────────────────────────────┘   │      │
│  └──────────────────────────────────────────────────────────────────────────┘      │
│                                                                                     │
│  ✅ RESULT: IP 192.168.100.50 BLOCKED AT ALL 3 LAYERS!                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. HARDWARE & ENVIRONMENT SETUP

### Hardware Specifications

| Component | Specification | Notes |
| --- | --- | --- |
| **Host Machine** | Apple MacBook with M4 Chip | ARM64 architecture |
| **Virtualization** | UTM (Universal Turing Machine) | QEMU-based |
| **VM OS** | Ubuntu 22.04 LTS (ARM64) | Server edition |
| **VM RAM** | 4GB+ (6GB recommended) | Critical for OpenSearch |
| **VM CPU** | 2 Cores | Minimum requirement |
| **VM Storage** | 20GB+ | For Docker images |

### Why ARM64 Matters

**The Challenge:**

- M4 Mac uses ARM64 architecture
- Many Docker images are built for AMD64 (Intel)
- This creates platform compatibility issues

**Our Solution:**

- We used ARM64-compatible images where possible
- For Shuffle components, we used the Worker instead of Orborus
- We built external Python services (Flask) that work natively on ARM64

### Network Configuration

**UTM Settings:**

```text
Network Mode: Bridged (Advanced)
Bridged Interface: Automatic
Emulated Network Card: virtio-net-pci
MAC Address: 26:97:D8:DC:40:17
Guest Network: 10.0.2.0/24
DHCP Start: 10.0.2.15
```

**VM IP Address:**

```text
192.168.31.114
```

This is the IP we used to access Shuffle and test the pipeline.

---

## 3. PHASE 1: UBUNTU VM SETUP

### Step 1: Install Ubuntu in UTM

**Process:**

1. Download Ubuntu Server ARM64 ISO from [ubuntu.com](https://ubuntu.com)
2. Create new VM in UTM
3. Select "Linux" → "Ubuntu" template
4. Allocate 4GB+ RAM, 2+ CPUs
5. Install Ubuntu with default settings

### Step 2: Get VM IP Address

**Command:**

```bash
ip addr show | grep inet
```

**Output:**

```text
inet 127.0.0.1/8 scope host lo
inet 192.168.31.114/24 brd 192.168.31.255 scope global dynamic nopextxroute enp0s1
```

**What This Means:**

- `192.168.31.114` = Your VM's IP address
- You'll use this IP to access Shuffle from your Mac

### Step 3: Update System

**Why:** Ensures you have the latest security patches and software versions.

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 4. PHASE 2: DOCKER & SHUFFLE INSTALLATION

### Step 1: Install Docker

**Why Docker:**

- Containers isolate software
- Easy to install and uninstall
- Shuffle runs in containers
- Consistent environment

```bash
# Download Docker installation script
curl -fsSL https://get.docker.com -o get-docker.sh

# Run installation
sudo sh get-docker.sh

# Add user to Docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt install docker-compose-plugin -y
```

### Troubleshooting: Docker Permission Denied

**Error:**

```text
permission denied while trying to connect to the Docker API at unix:///var/run/docker.sock
```

**Why It Happened:**

- Our user wasn't in the docker group
- Group changes require logout/login

**How We Fixed It:**

```bash
# Option 1: Use sudo for now
sudo docker compose up -d

# Option 2: Logout and login
exit
# SSH back in

# Option 3: Apply group changes immediately
newgrp docker
```

### Step 2: Install Shuffle

**Why Shuffle:**

- Open-source SOAR platform
- Visual workflow builder
- Supports webhooks and Python
- Active community

```bash
# Clone Shuffle
git clone https://github.com/Shuffle/Shuffle.git
cd Shuffle

# Start containers
docker compose up -d
```

### Troubleshooting: ARM64 vs AMD64 Platform Mismatch

**Error:**

```text
The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)
```

**Why It Happened:**

- Mac M4 uses ARM64 architecture
- Some Docker images are built for AMD64 (Intel)

**How We Fixed It:**

```bash
# Install QEMU for emulation
sudo apt update
sudo apt install qemu-user-static binfmt-support -y

# Register emulation
sudo docker run --rm --privileged multiarch/qemu-user-static --reset -p yes

# Restart Shuffle
sudo docker compose down
sudo docker compose up -d
```

### Troubleshooting: OpenSearch Container Restarting

**Error:**

```text
STATUS: Restarting (137) 22 seconds ago
```

**Why It Happened:**

- Exit code 137 = Out of Memory (OOM)
- OpenSearch needs at least 4GB RAM

**How We Fixed It:**

```bash
# Check memory
free -h

# Increase VM RAM in UTM to 4GB+
# Or reduce OpenSearch memory
export OPENSEARCH_JAVA_OPTS="-Xms512m -Xmx1g"
sudo docker compose down
sudo docker compose up -d
```

### Troubleshooting: Orborus Crashing on ARM64

**Error:**

```text
shuffle-orborus: Restarting (255)
```

**Why It Happened:**

- Orborus is compiled for AMD64
- Doesn't work on ARM64 even with emulation

**How We Fixed It:**

- We stopped using Orborus
- We used Shuffle Worker instead (which works on ARM64)
- For code execution, we used an external Flask server

---

## 5. PHASE 3: BUILDING THE SOAR WORKFLOW

### Step 1: Access Shuffle Web Interface

**From Mac Browser:**

```text
http://192.168.31.114:3001
```

**What You Should See:**

- Shuffle login/signup page
- Option to create admin account

### Step 2: Create Admin Account

| Field | Value |
| --- | --- |
| Username | sonalkr31 |
| Email | your_email@example.com |
| Password | your_secure_password |

### Step 3: Create New Workflow

**Navigation:**

1. Click **"Workflows"** in left menu
2. Click **"Create New Workflow"**

**Workflow Details:**

```text
Name: Wazuh Incident Response
Description: Automated SSH brute force response pipeline
Tags: wazuh, ssh, automation, incident-response
```

### Step 4: Add Webhook Trigger

**Why:**

- Webhook is the entry point for alerts
- Wazuh will send alerts to this URL
- This triggers the workflow

**How:**

1. In left sidebar, find **"Triggers"** section
2. Drag **"Webhook"** onto the canvas
3. Rename to **"Webhook Trigger"**
4. Copy the **Webhook URL**

**Webhook URL:**

```text
http://192.168.31.114:3001/api/v1/hooks/webhook_9c32feda-a654-4da0-b118-34335e7ee430
```

### Step 5: Add Extract IP Node

**Why:**

- Wazuh sends alerts in JSON format
- Need to extract the attacker's IP

**Issue We Faced:**

- "Shuffle Tools" wouldn't activate
- "Execute python" not showing up

**How We Fixed It:**

- We used the HTTP node instead
- Called an external Flask server
- This avoided Shuffle's ARM64 issues

### Step 6: Add HTTP Node (Instead of Python)

**Why:**

- More reliable on ARM64
- Can call external services
- Easier to debug

**Configuration:**

```text
Method: POST
URL: http://localhost:5000/webhook
Body: $exec.webhook.body
```

### Step 7: Connect the Nodes

**Connection Order:**

```text
[Webhook Trigger] → [Extract IP] → [HTTP]
```

**How to Connect:**

1. Drag from BOTTOM circle of Webhook
2. Drag to TOP circle of Extract IP
3. Drag from BOTTOM of Extract IP
4. Drag to TOP of HTTP

---

## 6. PHASE 4: THE ARM64 CHALLENGE & SOLUTIONS

### Challenge 1: Shuffle's Python Execution

**Problem:**

- Shuffle's "Execute python" relies on Orborus
- Orborus crashes on ARM64
- Cannot run Python code inside Shuffle

**Solution:**

- Built external Flask server
- Used HTTP node to call it
- Python runs natively on VM (ARM64 compatible)

### Challenge 2: Docker Platform Mismatch

**Problem:**

- Many images built for AMD64
- ARM64 emulation is slow and buggy

**Solution:**

- Used only ARM64-compatible images
- Built custom services (Flask)
- Used Alpine Linux images (ARM64 compatible)

### Challenge 3: Permission Issues

**Problem:**

- Python couldn't run `sudo ufw deny`
- hosts.deny had restricted permissions

**Solution:**

```bash
# Give write permission to hosts.deny
sudo chmod 666 /etc/hosts.deny

# Give Python sudo permission (optional)
sudo visudo -f /etc/sudoers.d/soar
# Add: ALL ALL=(ALL) NOPASSWD: /usr/sbin/ufw, /sbin/iptables
```

### Challenge 4: Flask Installation

**Problem:**

```text
error: externally-managed-environment
```

**Why It Happened:**

- Ubuntu 22.04+ protects system Python
- Cannot install packages globally

**Solution:**

```bash
# Create virtual environment
python3 -m venv soar-env

# Activate it
source soar-env/bin/activate

# Install Flask
pip install flask
```

---

## 7. PHASE 5: BUILDING THE EXTERNAL WEBHOOK SERVER

### Why We Built This

| Issue with Shuffle | Solution with Flask |
| --- | --- |
| Python execution unreliable on ARM64 | Python runs natively on VM |
| Orborus crashes | Flask server stable |
| Hard to debug | Easy to test independently |
| Limited logging | Full logging capabilities |

### The Complete Flask Server Code

```python
#!/usr/bin/env python3
from flask import Flask, request, jsonify
import subprocess
import json
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    """
    Receives POST request from Shuffle HTTP node
    Extracts IP from JSON payload
    Calls block_ip.py to block the IP
    Returns success/failure response
    """
    try:
        # Get JSON data from request
        data = request.json
        app.logger.info(f"Received: {data}")
        
        # Call our block script with the data
        result = subprocess.run(
            ['python3', '/home/kumar/block_ip.py'],
            input=json.dumps(data),
            capture_output=True,
            text=True
        )
        
        # Parse the response
        response = json.loads(result.stdout)
        app.logger.info(f"Response: {response}")
        
        # Return to Shuffle
        return jsonify(response), 200
        
    except Exception as e:
        app.logger.error(f"Error: {e}")
        return jsonify({"success": False, "error": str(e)}), 500

@app.route('/health', methods=['GET'])
def health():
    """Health check endpoint for testing"""
    return jsonify({"status": "healthy"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

### The block_ip.py Script

```python
#!/usr/bin/env python3
import json
import subprocess
import sys
import os
from datetime import datetime

def block_ip(ip):
    """
    Blocks IP at three security layers:
    1. hosts.deny - System level
    2. UFW - Firewall level
    3. iptables - Kernel level
    """
    results = []
    
    # Method 1: hosts.deny (NO sudo needed!)
    try:
        with open('/etc/hosts.deny', 'a') as f:
            f.write(f"ALL: {ip}\n")
        results.append("hosts.deny")
        print(f"✅ Added {ip} to hosts.deny")
    except Exception as e:
        print(f"❌ hosts.deny error: {e}")
        results.append(f"hosts.deny_error: {e}")
    
    # Method 2: UFW (needs sudo)
    try:
        subprocess.run(
            f"sudo ufw deny from {ip} to any",
            shell=True,
            capture_output=True,
            text=True
        )
        results.append("ufw")
        print(f"✅ Added {ip} to UFW")
    except Exception as e:
        print(f"❌ UFW error: {e}")
        results.append(f"ufw_error: {e}")
    
    # Method 3: iptables (needs sudo)
    try:
        subprocess.run(
            f"sudo iptables -I INPUT -s {ip} -j DROP",
            shell=True,
            capture_output=True,
            text=True
        )
        results.append("iptables")
        print(f"✅ Added {ip} to iptables")
    except Exception as e:
        print(f"❌ iptables error: {e}")
        results.append(f"iptables_error: {e}")
    
    return results

if __name__ == "__main__":
    # Read input from HTTP request
    try:
        input_data = sys.stdin.read()
        data = json.loads(input_data)
        
        # Extract IP from various formats
        src_ip = data.get('srcip')
        if not src_ip and 'data' in data:
            src_ip = data['data'].get('srcip')
        
        if src_ip:
            methods = block_ip(src_ip)
            response = {
                "success": True,
                "ip": src_ip,
                "methods": methods,
                "timestamp": datetime.now().isoformat()
            }
        else:
            response = {
                "success": False,
                "error": "No IP found in request"
            }
        
        print(json.dumps(response))
    except Exception as e:
        print(json.dumps({"success": False, "error": str(e)}))
```

### Setting Up the Flask Server

```bash
# Create virtual environment
cd ~
python3 -m venv soar-env

# Activate it
source soar-env/bin/activate

# Install Flask
pip install flask

# Create the server file (webhook_server.py)
# Create the block script (block_ip.py)
# Make them executable
chmod +x ~/webhook_server.py
chmod +x ~/block_ip.py

# Run the server
python3 ~/webhook_server.py &
```

---

## 8. PHASE 6: TESTING & VERIFICATION

### Test 1: Flask Server Health Check

```bash
curl http://localhost:5000/health
```

**Expected Output:**

```json
{"status":"healthy"}
```

**Why This Test:**

- Confirms Flask server is running
- Checks server is reachable
- Validates the environment

### Test 2: Direct Webhook Test

```bash
curl -X POST http://localhost:5000/webhook \
  -H "Content-Type: application/json" \
  -d '{"srcip": "192.168.100.50"}'
```

**Expected Output:**

```json
{"success": true, "ip": "192.168.100.50", "methods": ["hosts.deny", "ufw", "iptables"]}
```

**Why This Test:**

- Tests the blocking script independently
- Verifies all three methods work
- Confirms IP extraction logic

### Test 3: Full Shuffle Pipeline Test

```bash
curl -X POST http://192.168.31.114:3001/api/v1/hooks/webhook_9c32feda-a654-4da0-b118-34335e7ee430 \
  -H "Content-Type: application/json" \
  -d '{"srcip": "192.168.100.50"}'
```

**Expected Output:**

```json
{"success": true, "execution_id": "ef722e2a-7f55-40db-b58f-74605623b30f"}
```

**Why This Test:**

- Tests the complete pipeline
- Validates Shuffle workflow
- Confirms webhook integration

### Test 4: Verification of Blocking

```bash
# Check hosts.deny
sudo cat /etc/hosts.deny | grep 192.168.100.50

# Check UFW
sudo ufw status | grep 192.168.100.50

# Check iptables
sudo iptables -L INPUT -n | grep 192.168.100.50
```

**Expected Output:**

```text
ALL: 192.168.100.50
192.168.100.50               DENY        Anywhere
DROP       all  --  192.168.100.50       0.0.0.0/0
```

---

## 9. PHASE 7: SECURITY LAYER INTEGRATION

### Why Three Layers?

| Layer | Tool | Level | Why |
| --- | --- | --- | --- |
| 1 | hosts.deny | System | First line of defense, works without sudo |
| 2 | UFW | Firewall | Application-level firewall, user-friendly |
| 3 | iptables | Kernel | Deepest level, most secure |

### Layer 1: hosts.deny

**What It Is:**

- `/etc/hosts.deny` is a system file
- Controls access at the TCP wrapper level
- Blocks connections before they reach services

**How It Works:**

```text
Format: SERVICE: IP
Example: ALL: 192.168.100.50
```

**Why We Used It:**

- No sudo required (after chmod 666)
- Reliable on all Linux systems
- Immediate blocking

**Our Code:**

```python
with open('/etc/hosts.deny', 'a') as f:
    f.write(f"ALL: {ip}\n")
```

### Layer 2: UFW (Uncomplicated Firewall)

**What It Is:**

- Frontend for iptables
- User-friendly firewall management
- Default on Ubuntu

**How It Works:**

```text
sudo ufw deny from 192.168.100.50 to any
```

**Why We Used It:**

- Easy to use
- Persistent across reboots
- Standard Ubuntu tool

**Our Code:**

```python
subprocess.run(
    f"sudo ufw deny from {ip} to any",
    shell=True,
    capture_output=True,
    text=True
)
```

### Layer 3: iptables

**What It Is:**

- Kernel-level firewall
- Directly manipulates netfilter
- Most powerful firewall tool

**How It Works:**

```text
sudo iptables -I INPUT -s 192.168.100.50 -j DROP
```

**Why We Used It:**

- Works at kernel level
- Most secure
- Bypasses higher-level filters

**Our Code:**

```python
subprocess.run(
    f"sudo iptables -I INPUT -s {ip} -j DROP",
    shell=True,
    capture_output=True,
    text=True
)
```

---

## 10. COMPLETE CODE REFERENCE

### File Structure

```text
/home/kumar/
├── soar-env/                    # Python virtual environment
│   ├── bin/
│   │   ├── activate
│   │   └── python
│   └── lib/
│       └── python3.13/
│           └── site-packages/
│               └── flask/
├── block_ip.py                  # IP blocking script
├── webhook_server.py            # Flask webhook server
└── docker/
    └── Shuffle/                 # Shuffle installation
        ├── docker-compose.yml
        ├── nginx.conf
        └── ...
```

### Complete Code Files

#### 1. block_ip.py

```python
#!/usr/bin/env python3
import json
import subprocess
import sys
import os
from datetime import datetime

def block_ip(ip):
    results = []
    
    # Method 1: hosts.deny
    try:
        with open('/etc/hosts.deny', 'a') as f:
            f.write(f"ALL: {ip}\n")
        results.append("hosts.deny")
        print(f"✅ Added {ip} to hosts.deny")
    except Exception as e:
        print(f"❌ hosts.deny error: {e}")
        results.append(f"hosts.deny_error: {e}")
    
    # Method 2: UFW
    try:
        subprocess.run(
            f"sudo ufw deny from {ip} to any",
            shell=True,
            capture_output=True,
            text=True
        )
        results.append("ufw")
        print(f"✅ Added {ip} to UFW")
    except Exception as e:
        print(f"❌ UFW error: {e}")
        results.append(f"ufw_error: {e}")
    
    # Method 3: iptables
    try:
        subprocess.run(
            f"sudo iptables -I INPUT -s {ip} -j DROP",
            shell=True,
            capture_output=True,
            text=True
        )
        results.append("iptables")
        print(f"✅ Added {ip} to iptables")
    except Exception as e:
        print(f"❌ iptables error: {e}")
        results.append(f"iptables_error: {e}")
    
    return results

if __name__ == "__main__":
    try:
        input_data = sys.stdin.read()
        data = json.loads(input_data)
        
        src_ip = data.get('srcip')
        if not src_ip and 'data' in data:
            src_ip = data['data'].get('srcip')
        
        if src_ip:
            methods = block_ip(src_ip)
            response = {
                "success": True,
                "ip": src_ip,
                "methods": methods,
                "timestamp": datetime.now().isoformat()
            }
        else:
            response = {
                "success": False,
                "error": "No IP found in request"
            }
        
        print(json.dumps(response))
    except Exception as e:
        print(json.dumps({"success": False, "error": str(e)}))
```

#### 2. webhook_server.py

```python
#!/usr/bin/env python3
from flask import Flask, request, jsonify
import subprocess
import json
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    try:
        data = request.json
        app.logger.info(f"Received: {data}")
        
        result = subprocess.run(
            ['python3', '/home/kumar/block_ip.py'],
            input=json.dumps(data),
            capture_output=True,
            text=True
        )
        
        response = json.loads(result.stdout)
        app.logger.info(f"Response: {response}")
        return jsonify(response), 200
        
    except Exception as e:
        app.logger.error(f"Error: {e}")
        return jsonify({"success": False, "error": str(e)}), 500

@app.route('/health', methods=['GET'])
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

#### 3. Extract IP Code (Shuffle Workflow)

```python
import json

def execute(data):
    # Get the webhook data
    webhook_data = data.get('webhook', {})
    body = webhook_data.get('body', {})
    
    # If body is a string, parse it
    if isinstance(body, str):
        try:
            body = json.loads(body)
        except:
            body = {}
    
    # Try multiple locations for the IP
    src_ip = None
    
    # Method 1: Direct srcip
    if not src_ip:
        src_ip = body.get('srcip')
    
    # Method 2: In data field
    if not src_ip and 'data' in body:
        src_ip = body['data'].get('srcip')
    
    # Method 3: In agent field
    if not src_ip and 'agent' in body:
        src_ip = body['agent'].get('ip')
    
    # Method 4: In alert field
    if not src_ip and 'alert' in body:
        src_ip = body['alert'].get('srcip')
    
    # Get rule info
    rule_id = body.get('rule', {}).get('id', 'unknown')
    description = body.get('rule', {}).get('description', 'unknown')
    
    if src_ip:
        return {
            'attacker_ip': src_ip,
            'rule_id': rule_id,
            'description': description,
            'success': True
        }
    else:
        return {
            'success': False,
            'error': 'No IP found in data',
            'received_data': body
        }
```

---

## 11. TROUBLESHOOTING GUIDE

### Issue 1: Docker Permission Denied

**Error:**

```text
permission denied while trying to connect to the Docker API
```

**Solution:**

```bash
sudo usermod -aG docker $USER
newgrp docker
# OR logout and login
```

### Issue 2: ARM64 Platform Mismatch

**Error:**

```text
The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)
```

**Solution:**

```bash
sudo apt install qemu-user-static binfmt-support -y
sudo docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

### Issue 3: OpenSearch Out of Memory

**Error:**

```text
Restarting (137) 22 seconds ago
```

**Solution:**

```bash
# Increase VM RAM to 4GB+
# Or limit OpenSearch memory
export OPENSEARCH_JAVA_OPTS="-Xms512m -Xmx1g"
sudo docker compose down
sudo docker compose up -d
```

### Issue 4: Orborus Crashing on ARM64

**Error:**

```text
shuffle-orborus: Restarting (255)
```

**Solution:**

```bash
# Stop Orborus
sudo docker stop shuffle-orborus

# Use Worker instead
sudo docker compose up -d worker

# Or use external Flask server
```

### Issue 5: Flask "externally-managed-environment"

**Error:**

```text
error: externally-managed-environment
```

**Solution:**

```bash
# Use virtual environment
python3 -m venv soar-env
source soar-env/bin/activate
pip install flask
```

### Issue 6: Shuffle Tools Not Activating

**Error:**

- No "Execute python" option
- Clicking Shuffle Tools does nothing

**Solution:**

```bash
# Use HTTP node instead
# Call external Flask server
```

### Issue 7: UFW Not Blocking

**Error:**

- `sudo ufw status` shows inactive
- IP not being blocked

**Solution:**

```bash
# Enable UFW
sudo ufw enable
sudo ufw allow ssh
```

### Issue 8: Permission Denied for hosts.deny

**Error:**

```text
PermissionError: [Errno 13] Permission denied: '/etc/hosts.deny'
```

**Solution:**

```bash
sudo chmod 666 /etc/hosts.deny
```

### Issue 9: Python Can't Run Sudo

**Error:**

```text
sudo: no tty present and no askpass program specified
```

**Solution:**

```bash
sudo visudo -f /etc/sudoers.d/soar
# Add: ALL ALL=(ALL) NOPASSWD: /usr/sbin/ufw, /sbin/iptables
```

### Issue 10: Nodes Not Connecting in Shuffle

**Error:**

- No lines between nodes
- Workflow not executing

**Solution:**

1. Drag from BOTTOM circle of first node
2. Drag to TOP circle of second node
3. You should see a line with arrow

---

## 12. GLOSSARY

### Key Terms

| Term | Definition |
| --- | --- |
| **SOAR** | Security Orchestration, Automation, and Response |
| **SIEM** | Security Information and Event Management |
| **ARM64** | 64-bit ARM architecture (used in M4 Mac) |
| **AMD64** | 64-bit x86 architecture (Intel/AMD) |
| **Docker** | Containerization platform |
| ** |  |



-----------------------------------------------







###  Quick Demo (Optional)

Add this after the Executive Summary:

```markdown
## 🎬 Quick Demo

### One-Line Test
```bash
curl -X POST http://192.168.31.114:3001/api/v1/hooks/webhook_9c32feda-a654-4da0-b118-34335e7ee430 \
  -H "Content-Type: application/json" -d '{"srcip": "192.168.100.50"}'
```

### Expected Output
```json
{"success": true, "execution_id": "ef722e2a-7f55-40db-b58f-74605623b30f"}
```

### Verification
```bash
✅ hosts.deny: ALL: 192.168.100.50
✅ UFW: DENY 192.168.100.50
✅ iptables: DROP 192.168.100.50
```
```

---

### 3. Add "Why This Project Matters" Section

Add this before the Conclusion:

```markdown
## 💡 Why This Project Matters

### For Security Teams
- **Time Savings**: Reduces incident response from hours to milliseconds
- **Cost Reduction**: Eliminates manual monitoring costs
- **Consistency**: 100% reliable automated response

### For Organizations
- **24/7 Protection**: Never misses an attack
- **Scalability**: Handles unlimited threats
- **Compliance**: Automated audit trails

### For Your Career
- **Demonstrates**: SOAR, Python, Docker, Security skills
- **Shows**: End-to-end project ownership
- **Proves**: Ability to solve real-world problems
```

---

###  Next Steps

Add this at the end:

```markdown
## 🚀 Next Steps (Future Enhancements)

| Priority | Enhancement | Status |
|----------|-------------|--------|
| **P1** | Wazuh Integration | 📝 Planned |
| **P2** | Slack/Email Notifications | 📝 Planned |
| **P3** | Dashboard & Analytics | 📝 Planned |
| **P4** | Whitelist Management | 📝 Planned |
| **P5** | AI-powered Anomaly Detection | 📝 Planned |

### How You Can Contribute
1. ⭐ **Star** this repository
2. 🐛 **Report** issues
3. 🔧 **Submit** pull requests
4. 📝 **Suggest** improvements
```

---

###  Skills Demonstrated 

```markdown
## 🛠️ Skills Demonstrated

| Category | Technologies | Proficiency |
|----------|--------------|-------------|
| **SOAR** | Shuffle Automation | ✅ Advanced |
| **Backend** | Python, Flask | ✅ Advanced |
| **DevOps** | Docker, Docker Compose | ✅ Intermediate |
| **Security** | UFW, iptables, hosts.deny | ✅ Advanced |
| **Linux** | Ubuntu 22.04, System Administration | ✅ Advanced |
| **Architecture** | ARM64, Virtualization (UTM) | ✅ Intermediate |
| **Automation** | Bash, Python Scripting | ✅ Advanced |
| **Documentation** | Technical Writing | ✅ Advanced |
```

---


