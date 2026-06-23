# 🌐 BAKLAVA / NEXUS CORE
### Comprehensive Server Deployment Guide for OpenMediaVault (OMV)

> **BaklavaApp** is your ultimate mobile command center for server administration. Securely manage Linux nodes via SSH/SFTP, monitor services, run scripts, and perform diagnostics with a sleek, cyberpunk UI. Take full control of your infrastructure from anywhere via Nexus Core integration.

---

## Table of Contents
1. [Part 1: Core OMV Environment & SSH Access](#part-1-core-omv-environment--ssh-access)
2. [Part 2: The 3-Phase Speedtest Daemon (Port 8098)](#part-2-the-3-phase-speedtest-daemon-port-8098)
3. [Part 3: Apache Guacamole Remote UX Gateway (Docker)](#part-3-apache-guacamole-remote-ux-gateway-docker)
4. [Part 4: Extended Media Transfer (SFTP) Setup](#part-4-extended-media-transfer-sftp-setup)
5. [Part 5: Firewall (UFW) & Intruder Mitigation (Fail2Ban)](#part-5-firewall-ufw--intruder-mitigation-fail2ban)
6. [Part 6: Ecosystem Diagnostics](#part-6-ecosystem-diagnostics)

---

## Part 1: Core OMV Environment & SSH Access

OpenMediaVault manages permissions and system services via a dedicated web panel. For the Baklava mobile application to securely execute terminal commands, monitor daemons, and manage APT packages, the host permissions must be properly established.

### Step 1.1: Activating SSH shell in OMV GUI
1. Log in to the OpenMediaVault web administration panel.
2. Navigate to **Services** -> **SSH**.
3. Ensure the **Enable** checkbox is toggled **ON**.
4. Configure the advanced parameters:
   * **Permit root login:** `Enabled` *(Required for native daemon/service control via the app)*.
   * **Password authentication:** `Enabled` *(Or disabled if you strictly rely on imported SSH Private Keys)*.
5. Click **Save**, then click the yellow **Apply** banner at the top of the screen to commit changes.

### Step 1.2: Installing essential host dependencies
Log into your OMV server terminal as `root` and execute the following one-liner to fetch all required network and processing binaries:

```bash
apt-get update && apt-get install -y \
  python3 \
  python3-pip \
  ufw \
  fail2ban \
  ethtool \
  curl \
  sed \
  coreutils
```

---

## Part 2: The 3-Phase Speedtest Daemon (Port 8098)

The mobile app requires a dedicated HTTP endpoint on port `8098` to process the symmetric throughput test (**Ping**, **150MB Download**, **150MB Upload**). 

### Step 2.1: Generating the 150MB dummy payload
Run the commands below to create the working directory and generate a high-entropy 150MB binary file using `/dev/urandom`:

```bash
mkdir -p /opt/nexus/speedtest
dd if=/dev/urandom of=/opt/nexus/speedtest/payload.bin bs=1M count=150
chmod -R 755 /opt/nexus/speedtest
```

### Step 2.2: Creating the native Python HTTP Bridge
Create the daemon file `/opt/nexus/speedtest/st.py`:

```bash
nano /opt/nexus/speedtest/st.py
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
            
            # Read and discard incoming stream to calculate pure upload throughput
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
        # Mute standard access logs to prevent journald I/O bottleneck
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
nano /etc/systemd/system/nexus-speedtest.service
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
systemctl daemon-reload
systemctl enable nexus-speedtest.service
systemctl start nexus-speedtest.service
```
*(Verify live status via: `systemctl status nexus-speedtest.service`)*

---

## Part 3: Apache Guacamole Remote UX Gateway (Docker)

The mobile app drives its **Remote UI module** via HTML5 Guacamole web sockets. We will deploy this stack using Docker Compose.

### Step 3.1: Generate PostgreSQL initialization schema
Prepare the directory and extract the native database schema:

```bash
mkdir -p /opt/nexus/guacamole/init
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgres > /opt/nexus/guacamole/init/initdb.sql
```

### Step 3.2: Create the Docker Compose stack
Create `/opt/nexus/guacamole/docker-compose.yml`:

```bash
nano /opt/nexus/guacamole/docker-compose.yml
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
docker compose up -d
```
> **OPSEC Note:** The web gateway is now mapped to `http://<YOUR_SERVER_IP>:8080/guacamole/`. Log in immediately using default credentials (**guacadmin** / **guacadmin**) and change the master password.

---

## Part 4: Extended Media Transfer (SFTP) Setup

To allow BaklavaApp to mount and browse remote file trees, set up an isolated mount point in the OMV Web GUI:

1. **Create Target Storage:** * Navigate to **Storage** -> **Shared Folders**.
   * Click **`+` (Add)**, name it `NEXUS_MEDIA`, select your volume, and apply.
2. **Assign User Access:**
   * Navigate to **Users** -> **Users**.
   * Create your dedicated mobile user (e.g., `AdminIT`). 
   * Under the **Shared Folders** tab, grant `Read/Write` permissions to `NEXUS_MEDIA`.
3. **Verify SFTP Subsystem:**
   * Open `/etc/ssh/sshd_config` and ensure the internal SFTP handler is explicitly declared:
   ```text
   Subsystem sftp internal-sftp
   ```

---

## Part 5: Firewall (UFW) & Intruder Mitigation (Fail2Ban)

Lock down the host to protect open ports while permitting raw traffic for the app's diagnostic sockets.

### Step 5.1: Apply UFW routing rules
Execute this rule chain in the terminal:

```bash
# Reset default policies
ufw default deny incoming
ufw default allow outgoing

# Port 22: Master SSH & SFTP Subsystem
ufw allow 22/tcp

# Port 8080: Apache Guacamole HTML5 Tunnel
ufw allow 8080/tcp

# Port 8098: 3-Phase Speedtest Daemon Bridge
ufw allow 8098/tcp

# Commit and activate firewall
ufw --force enable
```

### Step 5.2: Configure SSH Fail2Ban Jail
Create a local jail file `/etc/fail2ban/jail.local`:

```bash
nano /etc/fail2ban/jail.local
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
systemctl enable fail2ban
systemctl restart fail2ban
```

---

## Part 6: Ecosystem Diagnostics

If the mobile app reports an `UNKNOWN` or `TIMEOUT` state for any sub-module, use the following real-time terminal probes to inspect the host:

* **Watch live I/O on the Speedtest daemon:**
  ```bash
  journalctl -u nexus-speedtest.service -f
  ```
* **Check if your Phone's IP got accidentally banned by Fail2Ban:**
  ```bash
  fail2ban-client status sshd
  ```
* **Tail the Guacamole remote desktop handshake output:**
  ```bash
  docker logs -f nexus_guac_web
  ```

***

*Your OpenMediaVault infrastructure is now fully hardened, service-isolated, and ready to pair with BaklavaApp.*
