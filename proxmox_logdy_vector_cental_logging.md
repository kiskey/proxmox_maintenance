# Agentless Real-Time Centralized Log Infrastructure (Logdy + Vector)

A highly optimized, enterprise-grade, **100% in-memory (zero disk-write amplification)** logging pipeline. This architecture aggregates logs from nested Docker containers and standalone application log files inside LXCs without requiring agents, sidecars, or custom scripts within the containers themselves.

---

# Architecture Overview

```text
                                  [ PROXMOX HOST ]
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  [ LXC 101: Docker ]        [ LXC 102: App ]      [ Master Symlink Mapper ]  │
│  ┌──────────────────┐       ┌──────────────┐      ┌───────────────────────┐  │
│  │ Authelia, etc.   │       │ myapp/       │      │ update-lxc-symlinks.sh│  │
│  │ /run/docker.sock │       │ error.log    │      └───────────┬───────────┘  │
│  └────────┬─────────┘       └──────┬───────┘                  │              │
│           │                        │                          │ Resolves     │
│           ▼                        ▼                          │ Active PIDs  │
│     /run/docker-nested-101.sock  /run/lxc-102-myapp-error.log │              │
│     (or /dev/null if offline)    (or /dev/null if offline)    ▼              │
│           │                        │                                         │
│           └──────────────┬─────────┘                                         │
│                          ▼                                                   │
│                 [ Vector Host Daemon ]                                       │
│                          │                                                   │
│                          ▼                                                   │
│                     TCP 8123                                                 │
└──────────────────────────┼───────────────────────────────────────────────────┘
                           ▼
                [ Central Logdy Server ]
              ┌───────────────────────────┐
              │  Alpine LXC (RAM Buffers) │
              │  Web UI Port: 8080        │
              └───────────────────────────┘
```

---

# Design Principles

## RAM-Resident Logging

Logdy stores active log buffers entirely in memory using a Go-based ring buffer.

Benefits:

- No database dependency
- No local log storage
- Minimal SSD wear
- Fast in-memory filtering

## Agentless Collection

Vector runs on the Proxmox host and accesses container resources through:

```text
/proc/<PID>/root
```

This avoids:

- Container modifications
- Sidecar containers
- Additional log agents

## Offline Container Resiliency

If an LXC is offline during boot or service startup:

- The mapping script automatically redirects the host-side path to `/dev/null`
- `vector validate` succeeds
- Vector starts normally
- No boot-order dependency failures occur

---

# Phase 1: Setup Logdy Central Receiver (Alpine LXC)

Logdy receives logs on TCP port `8123` and serves the web dashboard on port `8080`.

---

## 1. Install Alpine Compatibility Layer

Logdy binaries are typically linked against `glibc`.

Since Alpine uses `musl`, install compatibility support:

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

# Listen for Vector on TCP 8123
# Serve Web UI on TCP 8080
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

Verify:

```text
http://<LOGDY_LXC_IP>:8080
```

---

# Phase 2: Install Vector on the Proxmox Host

Vector acts as the centralized log router.

---

## 1. Configure Official Repository

```bash
# Install prerequisites
apt-get update
apt-get install -y apt-transport-https curl gnupg

# Import Datadog GPG key
curl -sSf https://keys.datadoghq.com/DATADOG_APT_KEY_CURRENT.public \
  | gpg --dearmor \
  | tee /usr/share/keyrings/datadog-archive-keyring.gpg >/dev/null

# Register repository
echo "deb [signed-by=/usr/share/keyrings/datadog-archive-keyring.gpg] https://apt.vector.dev/ stable vector-0" \
  | tee /etc/apt/sources.list.d/vector.list

# Install Vector
apt-get update
apt-get install -y vector
```

---

# Phase 3: Dynamic PID Mapping

Create a single self-healing mapping service that dynamically exposes files and sockets from LXCs.

Adding future mappings only requires another entry in the `MAP` array.

---

## 1. Create Master Symlink Mapper

Create:

```bash
nano /usr/local/bin/update-lxc-symlinks.sh
```

Paste:

```bash
#!/bin/bash

# ============================================================================
# MASTER LXC SYMLINK MAPPER
# ============================================================================
# Format:
# VMID:SOURCE_PATH_INSIDE_CONTAINER:PERSISTENT_HOST_LINK
# ============================================================================

MAP=(
  "101:/run/docker.sock:/run/docker-nested-101.sock"
  "102:/var/log/myapp/error.log:/run/lxc-102-myapp-error.log"
)

echo "Initializing dynamic LXC mappings..."

for ENTRY in "${MAP[@]}"; do
    IFS=":" read -r VMID SOURCE_PATH TARGET_LINK <<< "$ENTRY"

    PID=$(lxc-info -n "$VMID" -p -H 2>/dev/null)

    # Remove stale mapping
    rm -f "$TARGET_LINK"

    if [ ! -z "$PID" ] && [ "$PID" != "-1" ]; then

        ln -s "/proc/$PID/root$SOURCE_PATH" "$TARGET_LINK"

        echo "[ACTIVE] LXC $VMID -> $SOURCE_PATH -> $TARGET_LINK"

    else

        ln -s "/dev/null" "$TARGET_LINK"

        echo "[OFFLINE] LXC $VMID -> $TARGET_LINK -> /dev/null"

    fi
done
```

Make executable:

```bash
chmod +x /usr/local/bin/update-lxc-symlinks.sh
```

---

## 2. Configure Vector Service Overrides

Create the override directory:

```bash
mkdir -p /etc/systemd/system/vector.service.d/
```

Create:

```bash
cat << 'EOF' > /etc/systemd/system/vector.service.d/override.conf
[Service]
User=root
Group=root

# Clear vendor-defined pre-start actions
ExecStartPre=

# Step 1: Build namespace mappings
ExecStartPre=/usr/local/bin/update-lxc-symlinks.sh

# Step 2: Validate Vector configuration
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

  # Docker logs from LXC 101
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

  # Application logs from LXC 102
  lxc_102_app_logs:
    type: file

    include:
      - "/run/lxc-102-myapp-error.log"

    read_from: end

sinks:

  logdy_central:
    type: socket

    inputs:
      - docker_nested_logs
      - lxc_102_app_logs

    address: "192.168.1.40:8123"
    mode: tcp

    encoding:
      codec: json
```

### Notes

- Replace `192.168.1.40` with your Logdy server IP.
- Docker and file sources are merged into a single output stream.
- JSON encoding preserves metadata for dashboard filtering.

---

# Phase 5: Validation and Operations

## Validate Configuration

```bash
vector validate --config-dir /etc/vector/
```

## Restart Vector

```bash
systemctl restart vector
```

## Check Status

```bash
systemctl status vector
```

## View Live Logs

```bash
journalctl -u vector -f
```

---

# Phase 6: Configure Logdy Dashboard

Open:

```text
http://<LOGDY_LXC_IP>:8080
```

---

## Enable JSON Parsing

1. Enable **Auto-Parse JSON**.
2. Open:

   ```text
   Settings → Columns
   ```

---

## Recommended Columns

### Container / File

| Setting | Value |
|----------|----------|
| Header | Container / File |
| JSON Path | `container_name` or `file` |

### Message

| Setting | Value |
|----------|----------|
| Header | Log Message |
| JSON Path | `message` |

Disable the default **Raw Line** column for a cleaner interface.

---

## Example Searches

Filter Docker container errors:

```text
container_name:"authelia" AND level:"error"
```

Filter a specific application log:

```text
file:"/var/log/myapp/error.log"
```

Search all error messages:

```text
level:"error"
```

---

# Operational Commands

## Refresh Namespace Mappings

```bash
/usr/local/bin/update-lxc-symlinks.sh
```

## Validate Configuration

```bash
vector validate --config-dir /etc/vector/
```

## Restart Vector

```bash
systemctl restart vector
```

## Restart Logdy

```bash
rc-service logdy restart
```

---

# Benefits

- Agentless log collection
- Works with Docker and non-Docker LXCs
- Zero database dependency
- RAM-only active storage
- Minimal SSD wear
- Automatic PID remapping after container restarts
- Boot-safe offline container handling
- Centralized real-time dashboard
- Structured JSON filtering and search
- Easily extensible through the `MAP` array
