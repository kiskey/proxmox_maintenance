# Agentless Real-Time Centralized Log Infrastructure (Logdy + Vector)

A production-grade, ultra-thin, and **100% in-memory (zero disk-write amplification)** log-centralization system. This architecture runs completely in RAM to protect SSD longevity on Proxmox hypervisors while providing real-time, sub-millisecond dashboard filtering.

---

# System Architecture

```text
                                  [ PROXMOX HOST ]
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  [ LXC 101: Docker Host ]              [ Host-Level Logging Router ]         │
│  ┌───────────────────────┐             ┌─────────────────────────────┐       │
│  │  - Authelia           │             │  Vector (Rust Engine)       │       │
│  │  - Postgres           │             │                             │       │
│  │  - Stremio Add-ons    │             │  1. Scrapes nested socket   │       │
│  │                       │             │     via /proc namespace     │       │
│  │  /run/docker.sock ───┼────────────►│  2. Routes structured JSON  │       │
│  └───────────────────────┘             └──────────────┬──────────────┘       │
│                                                       │ (Port 8123 TCP)      │
└───────────────────────────────────────────────────────┼──────────────────────┘
                                                        ▼
                                           [ CENTRAL LOGDY SERVER ]
                                         ┌───────────────────────────┐
                                         │  Alpine LXC (RAM Buffers) │
                                         │  Web UI Port: 8080        │
                                         └───────────────────────────┘
```

## Components

### Central Receiver

Logdy running in `socket` mode inside a lightweight Alpine Linux LXC.

- Stores active log buffers entirely in RAM.
- Provides real-time web dashboard access.
- Eliminates database requirements.
- Avoids disk-write amplification.

### Log Shipper

Vector running directly on the Proxmox host as a root service.

- Maps the namespace of nested LXC containers.
- Auto-discovers selected Docker containers.
- Streams structured logs via TCP.
- Requires no agents inside application containers.

---

# Phase 1: Setup Logdy Central Receiver (Alpine LXC)

Logdy serves as the central high-performance ring buffer.

Because Alpine uses `musl` instead of `glibc`, install compatibility layers before running Logdy.

## 1. Install Compatibility Layer & Logdy

Download your preferred Logdy release and place it at:

```text
/usr/local/bin/logdy
```

Install compatibility packages:

```bash
apk update
apk add gcompat

chmod +x /usr/local/bin/logdy
```

---

## 2. Create OpenRC Service

Create:

```bash
nano /etc/init.d/logdy
```

Paste:

```sh
#!/sbin/openrc-run

name="logdy"
description="Logdy Real-time Centralized Log Server"

command="/usr/local/bin/logdy"

# Opens:
# - TCP 8123 for Vector ingestion
# - TCP 8080 for browser UI
command_args="socket 8123 --port 8080 --ui-ip 0.0.0.0 --no-analytics --no-updates"

command_background="yes"

pidfile="/run/${RC_SVCNAME}.pid"

output_log="/var/log/logdy.log"
error_log="/var/log/logdy.err"

depend() {
    need net
}
```

---

## 3. Enable and Start Logdy

```bash
chmod +x /etc/init.d/logdy

rc-update add logdy default

rc-service logdy start
```

Verify access:

```text
http://<YOUR_ALPINE_LOGDY_IP>:8080
```

---

# Phase 2: Install Vector on the Proxmox Host

Run Vector directly on the Proxmox host so a single service can aggregate logs from any Docker-enabled LXC.

---

## 1. Register the Official Repository

```bash
# Install prerequisites
apt-get update
apt-get install -y apt-transport-https curl gnupg

# Download Datadog GPG key
curl -sSf https://keys.datadoghq.com/DATADOG_APT_KEY_CURRENT.public \
  | gpg --dearmor \
  | tee /usr/share/keyrings/datadog-archive-keyring.gpg >/dev/null

# Register Vector repository
echo "deb [signed-by=/usr/share/keyrings/datadog-archive-keyring.gpg] https://apt.vector.dev/ stable vector-0" \
  | tee /etc/apt/sources.list.d/vector.list

# Install Vector
apt-get update
apt-get install -y vector
```

---

# Phase 3: Dynamic PID Mapping for Nested Docker

Container host PIDs change whenever an LXC restarts.

To maintain a stable Docker socket path for Vector, create a dynamic symlink.

---

## 1. Create the Socket Mapping Script

```bash
nano /usr/local/bin/update-lxc-docker-sock.sh
```

Paste:

```bash
#!/bin/bash

VMID="101"
LINK_PATH="/run/docker-nested-101.sock"

# Retrieve active host PID of the LXC
PID=$(lxc-info -n $VMID -p -H 2>/dev/null)

if [ ! -z "$PID" ] && [ "$PID" != "-1" ]; then
    rm -f "$LINK_PATH"

    # Link directly into the LXC namespace
    ln -s "/proc/$PID/root/run/docker.sock" "$LINK_PATH"

    echo "Successfully mapped nested Docker socket of LXC $VMID (PID $PID) to $LINK_PATH"
else
    echo "LXC $VMID is currently stopped."
    exit 1
fi
```

Make executable:

```bash
chmod +x /usr/local/bin/update-lxc-docker-sock.sh
```

---

## 2. Configure Vector Service Overrides

Create the override directory:

```bash
mkdir -p /etc/systemd/system/vector.service.d/
```

Create the override:

```bash
cat << 'EOF' > /etc/systemd/system/vector.service.d/override.conf
[Service]
User=root
Group=root

# Clear existing validation ordering
ExecStartPre=

# Update nested Docker socket symlink
ExecStartPre=/usr/local/bin/update-lxc-docker-sock.sh

# Validate Vector configuration
ExecStartPre=/usr/bin/vector validate
EOF
```

Reload systemd:

```bash
systemctl daemon-reload
```

---

# Phase 4: Configure Vector Routing

Replace:

```text
/etc/vector/vector.yaml
```

with:

```yaml
sources:
  docker_nested_logs:
    type: docker_logs
    docker_host: "unix:///run/docker-nested-101.sock"

    include_containers:
      - "stremio-einthusan"
      - "stremio-einthusan-pg"
      - "authelia"
      - "easynews-plus-plus"
      - "stremio-recommends"
      - "stremio-reddit-recommends-onnx"
      - "bitmagnet"
      - "postgres"

sinks:
  logdy_central:
    type: socket

    inputs:
      - docker_nested_logs

    address: "192.168.1.40:8123"
    mode: tcp

    encoding:
      codec: json
```

### Notes

- `docker_host` points to the dynamically maintained socket symlink.
- `include_containers` limits collection to selected containers.
- `codec: json` preserves structured fields for Logdy parsing.
- Replace `192.168.1.40` with your Logdy server IP.

---

# Phase 5: Start and Validate

Validate configuration:

```bash
vector validate --config-dir /etc/vector/
```

Restart Vector:

```bash
systemctl restart vector
```

Check service status:

```bash
systemctl status vector
```

Follow logs:

```bash
journalctl -u vector -f
```

---

# Phase 6: Configure the Logdy Dashboard

Since Vector sends JSON payloads, configure Logdy to display structured fields.

## Dashboard Setup

1. Open:

   ```text
   http://<YOUR_ALPINE_LOGDY_IP>:8080
   ```

2. Enable **Auto-Parse JSON**.

3. Open:

   ```text
   Settings → Columns
   ```

4. Add a custom column:

   | Field      | Value            |
   |------------|------------------|
   | Header     | Container        |
   | JSON Path  | container_name   |

5. Add another custom column:

   | Field      | Value    |
   |------------|----------|
   | Header     | Log      |
   | JSON Path  | message  |

6. Disable the default **Raw Line** column.

7. Use structured filtering:

   ```text
   container_name:"authelia"
   ```

   or

   ```text
   container_name:"authelia" AND level:"error"
   ```

---

# Operational Commands

## Logdy

```bash
rc-service logdy start
rc-service logdy stop
rc-service logdy restart
rc-service logdy status
```

## Vector

```bash
systemctl start vector
systemctl stop vector
systemctl restart vector
systemctl status vector
```

## Live Logs

```bash
journalctl -u vector -f
```

```bash
tail -f /var/log/logdy.log
```

---

# Design Goals

- Agentless log collection
- Zero application changes
- RAM-only active buffering
- No database dependency
- Minimal SSD wear
- Lightweight Alpine deployment
- Single centralized dashboard
- Structured JSON log search
- Automatic recovery after container restarts
- Suitable for Proxmox + Docker nested LXC environments
