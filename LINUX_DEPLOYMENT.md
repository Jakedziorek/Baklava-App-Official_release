# 🐧 BAKLAVA / NEXUS CORE
### Comprehensive Generic Debian & Linux Deployment Guide

> [!WARNING]  
> **ADVANCED KNOWLEDGE REQUIRED**  
> Unlike OpenMediaVault (OMV), which provides a cohesive abstraction layer for system management, deploying on a generic Linux distribution requires manual intervention at the kernel, service, and network policy levels. 
> You must be comfortable with CLI operations, manual editing of system configuration files, and troubleshooting service dependencies.

---

## Table of Contents
1. [Part 1: Base Environment & Dependency Installation](#part-1-base-environment--dependency-installation)
2. [Part 2: Deploying the 3-Phase Speedtest Daemon](#part-2-deploying-the-3-phase-speedtest-daemon)
3. [Part 3: Apache Guacamole Remote UX Gateway (Docker)](#part-3-apache-guacamole-remote-ux-gateway-docker)
4. [Part 4: Manual Extended Media Transfer (SFTP) Setup](#part-4-manual-extended-media-transfer-sftp-setup)
5. [Part 5: Firewall & Intruder Mitigation (UFW/Firewalld & Fail2Ban)](#part-5-firewall--intruder-mitigation)
6. [Part 6: Ecosystem Diagnostics & Known Limitations](#part-6-ecosystem-diagnostics--known-limitations)

---

## Part 1: Base Environment & Dependency Installation

You must manually install the required runtimes. Use the package manager appropriate for your specific distribution.

**For Debian / Ubuntu:**
```bash
sudo apt-get update && sudo apt-get install -y \
  python3 \
  python3-pip \
  ufw \
  fail2ban \
  curl \
  systemd \
  docker.io \
  docker-compose
```

**For CentOS / RHEL / Fedora:**
```bash
sudo dnf install -y \
  python3 \
  python3-pip \
  firewalld \
  fail2ban \
  curl \
  systemd \
  docker \
  docker-compose

sudo systemctl enable --now docker
```

---

## Part 2: Deploying the 3-Phase Speedtest Daemon

The mobile app requires a dedicated HTTP endpoint on port `8098` to process the symmetric throughput test.

### Step 2.1: Generating the 150MB dummy payload
Run the commands below to create the working directory and generate a high-entropy 150MB binary file:

```bash
sudo mkdir -p /opt/nexus/speedtest
sudo dd if=/dev/urandom of=/opt/nexus/speedtest/payload.bin bs=1M count=150
sudo chmod -R 755 /opt/nexus/speedtest
```

### Step 2.2: Creating the native Python HTTP Bridge
Create the daemon file `/opt/nexus/speedtest/st.py`:

```bash
sudo nano /opt/nexus/speedtest/st.py
```

Paste the following pristine, ASCII-only Python script inside:

```python
import os
import http.server
import socketserver
import sys

PORT = 8098
DIRECTORY = "/opt/nexus/speedtest"

class NexusSpeedtestHandler(http.server.SimpleHTTPRequestHandler):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, directory=DIRECTORY, **kwargs)

    def do_POST(self):
        try:
            content_length = int(self.headers.get('Content-Length', 0))
            bytes_read = 0
            chunk_size = 64 * 1024
            
            while bytes_read < content_length:
                remain = content_length - bytes_read
                to_read = chunk_size if remain > chunk_size else remain
                chunk = self.rfile.read(to_read)
                if not chunk:
                    break
                bytes_read += len(chunk)
                
            self.send_response(200)
            self.send_header('Content-Type', 'text/plain')
            self.send_header('Connection', 'close')
            self.end_headers()
            self.wfile.write(b"UPLOAD_COMPLETE")
        except Exception as e:
            sys.stderr.write(f"Error handling POST: {str(e)}\n")

    def log_message(self, format, *args):
        pass

class UnlinkTCPServer(socketserver.TCPServer):
    allow_reuse_address = True

if __name__ == "__main__":
    os.chdir(DIRECTORY)
    with UnlinkTCPServer(('0.0.0.0', PORT), NexusSpeedtestHandler) as httpd:
        sys.stdout.write(f"Nexus Network Daemon active on port {PORT}\n")
        try:
            httpd.serve_forever()
        except KeyboardInterrupt:
            httpd.shutdown()
            sys.stdout.write("Daemon safely terminated\n")
```

### Step 2.3: Registering the Systemd Service
Create the configuration file `/etc/systemd/system/nexus-speedtest.service`:

```bash
sudo nano /etc/systemd/system/nexus-speedtest.service
```

Paste the definition block:

```ini
[Unit]
Description=Nexus Network Throughput Speedtest Daemon
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/nexus/speedtest
ExecStart=/usr/bin/python3 /opt/nexus/speedtest/st.py
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Reload the systemd daemon manager and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable nexus-speedtest.service
sudo systemctl start nexus-speedtest.service
```

---

## Part 3: Apache Guacamole Remote UX Gateway (Docker)

### Step 3.1: Generate PostgreSQL initialization schema
Prepare the directory and extract the native database schema:

```bash
sudo mkdir -p /opt/nexus/guacamole/init
sudo docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgres > /opt/nexus/guacamole/init/initdb.sql
```

### Step 3.2: Create the Docker Compose stack
Create `/opt/nexus/guacamole/docker-compose.yml`:

```bash
sudo nano /opt/nexus/guacamole/docker-compose.yml
```

Paste the multi-container configuration:

```yaml
version: '3.8'

services:
  guacd:
    image: guacamole/guacd
    container_name: nexus_guacd
    restart: always
    volumes:
      - /opt/nexus/guacamole/drive:/drive:rw
      - /opt/nexus/guacamole/record:/record:rw

  postgres:
    image: postgres:15-alpine
    container_name: nexus_guac_db
    restart: always
    environment:
      POSTGRES_DB: guacamole_db
      POSTGRES_USER: guacamole_user
      POSTGRES_PASSWORD: SecretSecurePassword2026
    volumes:
      - /opt/nexus/guacamole/dbdata:/var/lib/postgresql/data:rw
      - /opt/nexus/guacamole/init:/docker-entrypoint-initdb.d:ro

  guacamole:
    image: guacamole/guacamole
    container_name: nexus_guac_web
    restart: always
    ports:
      - "8080:8080"
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRES_HOSTNAME: postgres
      POSTGRES_DATABASE: guacamole_db
      POSTGRES_USER: guacamole_user
      POSTGRES_PASSWORD: SecretSecurePassword2026
    depends_on:
      - guacd
      - postgres
```

### Step 3.3: Spin up the Gateway
Navigate to the root directory and start the stack in detached mode:

```bash
cd /opt/nexus/guacamole
sudo docker compose up -d
```
> **Note:** Access the gateway at `http://<YOUR_SERVER_IP>:8080/guacamole/`. Default credentials are **guacadmin** / **guacadmin**. Change them immediately.

---

## Part 4: Manual Extended Media Transfer (SFTP) Setup

Without OMV's GUI, you must create a dedicated user and directory for SFTP transfers manually.

### Step 4.1: Create a dedicated user and directory
Run the following to create the target folder and a user (e.g., `AdminIT`):

```bash
sudo mkdir -p /srv/NEXUS_MEDIA
sudo useradd -m -d /srv/NEXUS_MEDIA -s /bin/false AdminIT
sudo passwd AdminIT
sudo chown -R AdminIT:AdminIT /srv/NEXUS_MEDIA
```
*(You will be prompted to set a secure password. Use this password in the BaklavaApp SFTP Setup Screen).*

### Step 4.2: Enforce the SFTP Subsystem
Open your SSH configuration file:

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure this line exists and is uncommented:
```text
Subsystem sftp internal-sftp
```

Restart the SSH service:
```bash
sudo systemctl restart sshd
```

---

## Part 5: Firewall & Intruder Mitigation

### Step 5.1: Routing Rules (Choose UFW or Firewalld)

**Option A: UFW (Debian / Ubuntu)**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 8080/tcp
sudo ufw allow 8098/tcp
sudo ufw --force enable
```

**Option B: Firewalld (CentOS / RHEL)**
```bash
sudo firewall-cmd --permanent --add-port=22/tcp
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --add-port=8098/tcp
sudo firewall-cmd --reload
```

### Step 5.2: Configure SSH Fail2Ban Jail
Create a local jail file `/etc/fail2ban/jail.local`:

```bash
sudo nano /etc/fail2ban/jail.local
```

Paste the strict defensive policy:

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 5
findtime = 600
bantime = 3600
```

Restart and enable the security daemon:

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
```

---

## Part 6: Ecosystem Diagnostics & Known Limitations

### Real-Time Diagnostics
* **Watch live I/O on the Speedtest daemon:** `journalctl -u nexus-speedtest.service -f`
* **Check Fail2Ban blocks:** `fail2ban-client status sshd`
* **Tail the Guacamole container logs:** `docker logs -f nexus_guac_web`

### Architectural Limitations
Because BaklavaApp is optimized for standardized environments (Debian/OMV), running it on heavily modified distributions (e.g., Arch Linux, Alpine) may cause the following sub-modules to fail:

| App Module | Requirement | Failure Symptom |
| :--- | :--- | :--- |
| **Daemon Director** | Requires `systemd`. | `systemctl` toggles will return execution errors on OpenRC/SysVinit. |
| **Journal Log Viewer** | Requires `journalctl`. | Will return empty arrays on systems using legacy `/var/log/syslog`. |
| **APT Manager** | Requires `apt` structure. | Will show "UNKNOWN" on `dnf`, `yum`, or `pacman` based distributions. |
