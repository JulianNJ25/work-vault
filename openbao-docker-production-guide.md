# OpenBao — Docker Production Deployment Guide
### Bare-Metal / Single Server, Security-Hardened, DevOps-Ready

---

## Overview & Scope

This guide covers deploying OpenBao as a Docker container on a standalone Linux server without an orchestration platform. The approach prioritizes security hardening, operational repeatability, and a clean DevOps workflow — laying the foundation for future automation.

**What this guide covers:**
- Directory layout and file permission model
- TLS certificate provisioning
- OpenBao configuration for production
- Docker container setup and security hardening
- Host network and firewall configuration
- Systemd service for container lifecycle management
- Backup and snapshot strategy
- Deployment workflow and upgrade procedure
- Audit and observability basics

**Assumptions:**
- OS: Ubuntu 22.04 LTS or Debian 12 (adjust package names as needed)
- Docker Engine installed (not Docker Desktop)
- You have `sudo` access on the server
- Domain or internal hostname resolves to this server (e.g., `bao.internal.company.com`)

---

## 1. System Preparation

### 1.1 Create a Dedicated System User

Never run OpenBao as root. Create a locked system user that owns all OpenBao resources.

```bash
sudo useradd \
  --system \
  --shell /usr/sbin/nologin \
  --create-home \
  --home-dir /opt/openbao \
  --comment "OpenBao service account" \
  openbao
```

Add this user to the `docker` group so it can manage the container (alternatively, use a wrapper script run by root — see Section 7):

```bash
sudo usermod -aG docker openbao
```

### 1.2 Establish the Directory Layout

A clean, predictable layout is critical for auditability and future automation.

```
/opt/openbao/
├── config/          # OpenBao HCL configuration files
├── data/            # Raft integrated storage (persistent volume)
├── tls/             # TLS certificates and private key
├── logs/            # Audit log output
├── scripts/         # Operational scripts (backup, health-check, etc.)
└── snapshots/       # Raft snapshot backups
```

Create the structure:

```bash
sudo mkdir -p /etc/openbao/{config,data,tls,logs,scripts,snapshots}

# Set ownership
sudo chown -R openbao:openbao /etc/openbao

# Set permissions — strict, no world-readable paths
sudo chmod 750 /opt/openbao
sudo chmod 750 /opt/openbao/config
sudo chmod 700 /opt/openbao/data      # Storage: owner only
sudo chmod 700 /opt/openbao/tls       # TLS keys: owner only
sudo chmod 750 /opt/openbao/logs
sudo chmod 750 /opt/openbao/scripts
sudo chmod 700 /opt/openbao/snapshots
```

---

## 2. TLS Certificates

OpenBao **must** run with TLS in production. Plaintext HTTP is only acceptable in isolated dev environments.

### 2.1 Option A — Internal CA / Self-Signed (for internal-only deployments)

```bash
# Generate CA key and certificate
openssl genrsa -out /tmp/ca-key.pem 4096
openssl req -new -x509 -days 3650 -key /tmp/ca-key.pem \
  -out /tmp/ca-cert.pem \
  -subj "/C=US/O=YourCompany/CN=Internal CA"

# Generate OpenBao server key
openssl genrsa -out /tmp/bao-key.pem 4096

# Create CSR with SAN (Subject Alternative Names)
cat > /tmp/bao-san.cnf <<EOF
[req]
default_bits       = 4096
distinguished_name = req_distinguished_name
req_extensions     = v3_req
prompt             = no

[req_distinguished_name]
C  = US
O  = YourCompany
CN = bao.internal.company.com

[v3_req]
subjectAltName = @alt_names

[alt_names]
DNS.1 = bao.internal.company.com
DNS.2 = localhost
IP.1  = 127.0.0.1
IP.2  = <YOUR_SERVER_IP>
EOF

openssl req -new -key /tmp/bao-key.pem \
  -out /tmp/bao-csr.pem \
  -config /tmp/bao-san.cnf

# Sign with internal CA
openssl x509 -req -days 825 \
  -in /tmp/bao-csr.pem \
  -CA /tmp/ca-cert.pem \
  -CAkey /tmp/ca-key.pem \
  -CAcreateserial \
  -out /tmp/bao-cert.pem \
  -extfile /tmp/bao-san.cnf \
  -extensions v3_req

# Install certificates
sudo cp /tmp/bao-cert.pem /opt/openbao/tls/fullchain.pem
sudo cp /tmp/ca-cert.pem /opt/openbao/tls/ca.pem     # Full chain for verification
sudo cp /tmp/bao-key.pem /opt/openbao/tls/privkey.pem

# Secure the private key
sudo chown openbao:openbao /opt/openbao/tls/*
sudo chmod 640 /opt/openbao/tls/fullchain.pem
sudo chmod 640 /opt/openbao/tls/ca.pem
sudo chmod 600 /opt/openbao/tls/privkey.pem   # Private key: owner read only

# Clean temp files
rm -f /tmp/bao-key.pem /tmp/ca-key.pem /tmp/bao-csr.pem /tmp/bao-san.cnf
```

### 2.2 Option B — Let's Encrypt (for public-facing deployments)

If the server is publicly reachable, use Certbot:

```bash
sudo apt install certbot -y
sudo certbot certonly --standalone \
  -d bao.company.com \
  --agree-tos \
  --email ops@company.com

# Symlink into OpenBao's TLS directory
sudo ln -sf /etc/letsencrypt/live/bao.company.com/fullchain.pem \
  /opt/openbao/tls/fullchain.pem
sudo ln -sf /etc/letsencrypt/live/bao.company.com/privkey.pem \
  /opt/openbao/tls/privkey.pem

# Certbot auto-renews; add a post-renewal hook to reload OpenBao
sudo bash -c 'cat > /etc/letsencrypt/renewal-hooks/post/openbao-reload.sh <<EOF
#!/bin/bash
systemctl restart openbao-docker
EOF'
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/openbao-reload.sh
```

---

## 3. OpenBao Configuration

Create the main configuration file:

```bash
sudo -u openbao tee /opt/openbao/config/openbao.hcl > /dev/null <<'EOF'
# =============================================================================
# OpenBao Production Configuration
# Server: bao.internal.company.com
# =============================================================================

# ── Cluster Identity ──────────────────────────────────────────────────────────
cluster_name = "prod-openbao"

api_addr     = "https://bao.internal.company.com:8200"
cluster_addr = "https://bao.internal.company.com:8201"

# ── UI ────────────────────────────────────────────────────────────────────────
ui = true

# ── Storage: Integrated Raft ──────────────────────────────────────────────────
storage "raft" {
  path    = "/openbao/data"
  node_id = "prod-node-1"

  # Recommended for production servers
  performance_multiplier = 1
}

# ── Listener ──────────────────────────────────────────────────────────────────
listener "tcp" {
  address       = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"

  tls_cert_file = "/openbao/tls/fullchain.pem"
  tls_key_file  = "/openbao/tls/privkey.pem"

  # Enforce modern TLS — reject TLS 1.0 and 1.1
  tls_min_version = "tls12"

  # Restrict allowed cipher suites (TLS 1.2)
  tls_cipher_suites = "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"

  # Security headers — harden the HTTP layer
  custom_response_headers {
    "default" = {
      "Strict-Transport-Security" = ["max-age=31536000; includeSubDomains"]
      "X-Content-Type-Options"    = ["nosniff"]
      "X-Frame-Options"           = ["DENY"]
    }
  }
}

# ── Audit Device (configured in config file, production v2.4+) ────────────────
audit "file" "prod_audit" {
  path     = "file"
  options = {
    file_path = "/openbao/logs/audit.log"
    mode      = "0600"
  }
}

# ── Lease TTLs ────────────────────────────────────────────────────────────────
default_lease_ttl = "24h"
max_lease_ttl     = "720h"

# ── Logging ───────────────────────────────────────────────────────────────────
log_level  = "info"
log_format = "json"

# Log to file with rotation
log_file             = "/openbao/logs/openbao.log"
log_rotate_bytes     = 52428800   # 50 MB
log_rotate_max_files = 10

# ── Telemetry (optional — expose for Prometheus scraping) ─────────────────────
telemetry {
  disable_hostname          = false
  prometheus_retention_time = "30s"
  unauthenticated_metrics_access = false
}

# ── Miscellaneous hardening ───────────────────────────────────────────────────
# Prevent exposing raw storage
raw_storage_endpoint   = false
introspection_endpoint = false

# Audit device creation via API requires explicit opt-in
unsafe_allow_api_audit_creation = false
EOF
```

Set correct permissions:

```bash
sudo chown openbao:openbao /opt/openbao/config/openbao.hcl
sudo chmod 640 /opt/openbao/config/openbao.hcl
```

---

## 4. Docker Setup

### 4.1 Create a Dedicated Docker Network

Isolate OpenBao in its own Docker bridge network. This prevents unrelated containers from communicating with it by default.

```bash
docker network create \
  --driver bridge \
  --subnet 172.28.0.0/24 \
  --gateway 172.28.0.1 \
  --opt com.docker.network.bridge.name=br-openbao \
  openbao-net
```

This command creates a **custom Docker bridge network**, which acts as a virtual switch allowing containers connected to it to communicate with each other in an isolated environment.

It defines a private IP range (`172.28.0.0/24`), giving 256 total addresses (254 usable), ensuring predictable and controlled addressing for containers.

A gateway (`172.28.0.1`) is assigned, which acts as the internal router for containers in this network, allowing them to reach external networks if needed.

The underlying Linux bridge interface is explicitly named `br-openbao`, making it easier to identify and manage at the system level.

By placing OpenBao in this dedicated network, only containers explicitly attached to it can communicate with it, providing **network-level isolation and improved security**.
### 4.2 The Docker Run Command (Reference)

This is the canonical run command. In production it will be managed by systemd (Section 5), not run manually. It is documented here for clarity.

```bash
docker run \
  --name openbao \
  --hostname openbao \
  \
  # ── Restart policy ───────────────────────────────
  --restart unless-stopped \
  \
  # ── Network ──────────────────────────────────────
  --network openbao-net \
  --ip 172.28.0.10 \
  -p 8200:8200 \
  -p 8201:8201 \
  \
  # ── Volumes ──────────────────────────────────────
  -v /opt/openbao/config:/openbao/config:ro \
  -v /opt/openbao/data:/openbao/data \
  -v /opt/openbao/tls:/openbao/tls:ro \
  -v /opt/openbao/logs:/openbao/logs \
  \
  # ── Security hardening ───────────────────────────
  --cap-drop ALL \
  --cap-add IPC_LOCK \
  --security-opt no-new-privileges:true \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  \
  # ── Resource limits ──────────────────────────────
  --memory 512m \
  --memory-swap 512m \
  --cpus 1.0 \
  \
  # ── Environment ──────────────────────────────────
  -e VAULT_LOG_LEVEL=info \
  \
  # ── User (UID must match openbao system user) ─────
  --user $(id -u openbao):$(id -g openbao) \
  \
  quay.io/openbao/openbao:2.5.0 \
  server \
  -config=/openbao/config/openbao.hcl
```

**Security flags explained:**

| Flag | Reason |
|---|---|
| `--cap-drop ALL` | Remove all Linux capabilities by default |
| `--cap-add IPC_LOCK` | Required — allows OpenBao to lock memory (prevents secrets from being swapped to disk) |
| `--security-opt no-new-privileges:true` | Prevents privilege escalation inside the container |
| `--read-only` | Root filesystem is read-only; only explicit volumes are writable |
| `--tmpfs /tmp` | Provides a writable, non-persistent `/tmp` with `noexec` |
| `--memory-swap 512m` | Sets swap equal to memory limit, effectively disabling swap (prevents secrets in swap) |
| `--user` | Runs as the non-root `openbao` system user |

### 4.3 Verify UID Alignment

The container process must run as the same UID that owns the host directories; otherwise volume writes will fail.

```bash
# Get the UID/GID of the openbao user
id openbao
# uid=999(openbao) gid=999(openbao) groups=999(openbao)

# Verify ownership on data directory
ls -la /opt/openbao/
```

If the UID on the host and inside the image differ, you may need to specify the numeric UID explicitly in `--user 999:999`.

---

## 5. Systemd Service (Container Lifecycle Management)

Systemd is the correct way to manage long-running services on a Linux server. It handles start-on-boot, restart-on-failure, and log capture via journald.

```bash
sudo tee /etc/systemd/system/openbao-docker.service > /dev/null <<'EOF'
[Unit]
Description=OpenBao Vault - Manejo de Secretos (Docker)
Documentation=https://openbao.org/docs/
Requires=docker.service network-online.target
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Group=root

# Clean up any stale container before starting
ExecStartPre=-/usr/bin/docker rm -f openbao

ExecStart=/usr/bin/docker run \
  --name openbao \
  --hostname openbao \
  --restart no \
  --network openbao-net \
  --ip 172.28.0.10 \
  -p 8200:8200 \
  -p 8201:8201 \
  -v /opt/openbao/config:/openbao/config:ro \
  -v /opt/openbao/data:/openbao/data \
  -v /opt/openbao/tls:/openbao/tls:ro \
  -v /opt/openbao/logs:/openbao/logs \
  --cap-drop ALL \
  --cap-add IPC_LOCK \
  --security-opt no-new-privileges:true \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --memory 512m \
  --memory-swap 512m \
  --cpus 1.0 \
  --user 997:983 \
  -e VAULT_LOG_LEVEL=info \
  quay.io/openbao/openbao:2.5.0 \
  server \
  -config=/openbao/config/openbao.hcl

ExecStop=/usr/bin/docker stop -t 30 openbao
ExecStopPost=-/usr/bin/docker rm -f openbao

# Restart behaviour
Restart=on-failure
RestartSec=10
StartLimitInterval=60
StartLimitBurst=3

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=openbao

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable openbao-docker.service
sudo systemctl start openbao-docker.service

# Verify it's running
sudo systemctl status openbao-docker.service

# Follow logs
sudo journalctl -u openbao-docker.service -f
```

**Important:** Note that `--restart no` is set in the `ExecStart` command. Systemd owns the restart logic — letting Docker also retry would create conflicts. Systemd's `Restart=on-failure` handles this correctly.

---

## 6. Host Network & Firewall Configuration

### 6.1 UFW (Ubuntu Firewall)

```bash
# Allow OpenBao API port from trusted networks only
# Replace with your actual admin/application CIDR
sudo ufw allow from 10.0.0.0/8 to any port 8200 proto tcp comment "OpenBao API"
sudo ufw allow from 192.168.0.0/16 to any port 8200 proto tcp comment "OpenBao API local"

# Port 8201 is for cluster-to-cluster communication (only if running HA)
# If single-node, you may block this externally
sudo ufw allow from 10.0.0.0/8 to any port 8201 proto tcp comment "OpenBao cluster"

# If you need SSH access, ensure it's still allowed
sudo ufw allow from <YOUR_ADMIN_IP> to any port 22 proto tcp

# Enable the firewall
sudo ufw enable
sudo ufw status verbose
```

### 6.2 iptables (if not using UFW)

```bash
# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow OpenBao from trusted range only
iptables -A INPUT -p tcp --dport 8200 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 8201 -s 10.0.0.0/8 -j ACCEPT

# Drop everything else
iptables -A INPUT -j DROP

# Persist rules
apt install iptables-persistent -y
netfilter-persistent save
```

### 6.3 Docker and UFW/iptables Interaction Warning

Docker bypasses UFW by default, writing its own iptables rules. If OpenBao's port is published with `-p 8200:8200`, it will be accessible from any IP regardless of UFW rules. To prevent this:

**Option 1 (recommended):** Bind Docker to localhost only and use a reverse proxy (nginx) that enforces access controls:

```bash
# Bind only to localhost in the docker run command:
-p 127.0.0.1:8200:8200
-p 127.0.0.1:8201:8201
```

Then expose it through nginx or HAProxy with proper ACLs.

**Option 2:** Use the `DOCKER-USER` iptables chain, which Docker respects:

```bash
# Restrict port 8200 to trusted CIDR via DOCKER-USER chain
iptables -I DOCKER-USER -p tcp --dport 8200 ! -s 10.0.0.0/8 -j DROP
iptables -I DOCKER-USER -p tcp --dport 8201 ! -s 10.0.0.0/8 -j DROP

# Save
netfilter-persistent save
```

---

## 7. First-Time Initialization

After the container is running, the following steps are performed **once** to bootstrap the cluster.

### 7.1 Set Up the CLI Environment

On the host (or on any machine with the `bao` binary and network access):

```bash
export BAO_ADDR="https://bao.internal.company.com:8200"
export BAO_CACERT="/opt/openbao/tls/ca.pem"    # If using internal CA

# Verify the server is reachable and in sealed state
bao status
```

Expected output: `Initialized: false`, `Sealed: true`.

### 7.2 Initialize

```bash
bao operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -format=json | tee /tmp/init-output.json
```

**Immediately** extract and securely store the unseal keys and root token. If using PGP (strongly recommended):

```bash
bao operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -pgp-keys="keybase:alice,keybase:bob,keybase:carol,keybase:dave,keybase:eve" \
  -root-token-pgp-key="keybase:alice"
```

After saving the output securely, remove the temporary file:

```bash
shred -u /tmp/init-output.json
```

### 7.3 Unseal

```bash
# Provide 3 of the 5 unseal keys (run three times with different keys)
bao operator unseal <key-1>
bao operator unseal <key-2>
bao operator unseal <key-3>

# Verify
bao status
# Sealed: false  ← expected
```

### 7.4 Authenticate with Root Token and Bootstrap

```bash
export BAO_TOKEN="<initial-root-token>"

# Enable audit logging (belt-and-suspenders alongside config file audit)
bao audit list   # Confirm config file audit is active

# Enable a secondary audit device (syslog) for redundancy
bao audit enable syslog

# Enable the KV v2 secrets engine
bao secrets enable -path=secret kv-v2

# Set up your first auth method (example: AppRole)
bao auth enable approle

# Revoke the root token — this is mandatory before leaving this step
bao token revoke "$BAO_TOKEN"
unset BAO_TOKEN
```

From this point forward, all access is through scoped tokens from configured auth methods.

---

## 8. Operational Scripts

Place these in `/opt/openbao/scripts/` and make them executable.

### 8.1 Health Check Script

```bash
sudo tee /opt/openbao/scripts/health-check.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
# OpenBao health check — exits 0 if healthy, non-zero otherwise
set -euo pipefail

BAO_ADDR="${BAO_ADDR:-https://bao.internal.company.com:8200}"
BAO_CACERT="${BAO_CACERT:-/opt/openbao/tls/ca.pem}"

STATUS=$(curl -sk --cacert "$BAO_CACERT" "$BAO_ADDR/v1/sys/health")
INITIALIZED=$(echo "$STATUS" | grep -o '"initialized":[^,}]*' | cut -d: -f2)
SEALED=$(echo "$STATUS" | grep -o '"sealed":[^,}]*' | cut -d: -f2)

echo "OpenBao Health:"
echo "  Initialized : $INITIALIZED"
echo "  Sealed      : $SEALED"

if [ "$INITIALIZED" = "true" ] && [ "$SEALED" = "false" ]; then
  echo "  Status      : HEALTHY"
  exit 0
else
  echo "  Status      : UNHEALTHY"
  exit 1
fi
EOF
chmod 750 /opt/openbao/scripts/health-check.sh
chown openbao:openbao /opt/openbao/scripts/health-check.sh
```

### 8.2 Raft Snapshot Backup Script

```bash
sudo tee /opt/openbao/scripts/snapshot-backup.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
# Creates a Raft snapshot and rotates old ones
# Requires a valid BAO_TOKEN with sys/storage/raft/snapshot permissions
set -euo pipefail

BAO_ADDR="${BAO_ADDR:-https://bao.internal.company.com:8200}"
BAO_CACERT="${BAO_CACERT:-/opt/openbao/tls/ca.pem}"
SNAPSHOT_DIR="/opt/openbao/snapshots"
RETENTION_DAYS=14
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
SNAPSHOT_FILE="$SNAPSHOT_DIR/openbao_snapshot_$TIMESTAMP.snap"

if [ -z "${BAO_TOKEN:-}" ]; then
  echo "ERROR: BAO_TOKEN is not set. A token with snapshot permissions is required." >&2
  exit 1
fi

echo "[$(date)] Creating snapshot: $SNAPSHOT_FILE"

bao operator raft snapshot save \
  -address="$BAO_ADDR" \
  -ca-cert="$BAO_CACERT" \
  "$SNAPSHOT_FILE"

chmod 600 "$SNAPSHOT_FILE"
chown openbao:openbao "$SNAPSHOT_FILE"

echo "[$(date)] Snapshot complete. Size: $(du -sh "$SNAPSHOT_FILE" | cut -f1)"

# Rotate old snapshots
echo "[$(date)] Pruning snapshots older than $RETENTION_DAYS days..."
find "$SNAPSHOT_DIR" -name "openbao_snapshot_*.snap" \
  -mtime "+$RETENTION_DAYS" -delete

echo "[$(date)] Backup complete."
EOF
chmod 750 /opt/openbao/scripts/snapshot-backup.sh
chown openbao:openbao /opt/openbao/scripts/snapshot-backup.sh
```

Schedule snapshots via cron:

```bash
# Add to root or openbao crontab
# Run daily at 02:00 AM
echo "0 2 * * * openbao BAO_TOKEN=$(cat /etc/openbao/snapshot-token) /opt/openbao/scripts/snapshot-backup.sh >> /opt/openbao/logs/snapshot.log 2>&1" \
  | sudo tee -a /etc/cron.d/openbao-backup

# Or use a systemd timer (preferred)
```

### 8.3 Log Rotation

```bash
sudo tee /etc/logrotate.d/openbao > /dev/null <<'EOF'
/opt/openbao/logs/audit.log
/opt/openbao/logs/openbao.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0600 openbao openbao
    sharedscripts
    postrotate
        # Signal OpenBao to reopen log files
        docker kill --signal=HUP openbao 2>/dev/null || true
    endscript
}
EOF
```

---

## 9. Continuous Deployment Workflow

### 9.1 Version Pinning Strategy

**Never use the `latest` tag in production.** Always pin to a specific digest-verified version.

```bash
# Pull a specific version and verify
docker pull quay.io/openbao/openbao:2.5.0

# Record the digest for verification in CI/CD
docker inspect --format='{{index .RepoDigests 0}}' quay.io/openbao/openbao:2.5.0
# quay.io/openbao/openbao@sha256:abc123...
```

Store the pinned version in a `.env` file or config file tracked in version control:

```bash
# /opt/openbao/config/version.env
OPENBAO_IMAGE=quay.io/openbao/openbao
OPENBAO_VERSION=2.5.0
OPENBAO_DIGEST=sha256:<full-digest>
```

Update the systemd unit to use these variables, or reference them in a deploy script.

### 9.2 Upgrade Procedure

Upgrading OpenBao follows a deliberate, verified sequence:

```
1. Review the release notes and upgrade guide for the target version
2. Take a Raft snapshot backup
3. Pull the new image and verify its digest
4. Stop the container gracefully
5. Start with the new image
6. Verify health
7. If unhealthy: rollback (restore snapshot if needed)
```

**Upgrade script:**

```bash
sudo tee /opt/openbao/scripts/upgrade.sh > /dev/null <<'UPGRADE'
#!/usr/bin/env bash
# OpenBao upgrade script
# Usage: ./upgrade.sh <new-version> <expected-digest>
set -euo pipefail

NEW_VERSION="${1:?Usage: $0 <version> <digest>}"
EXPECTED_DIGEST="${2:?Usage: $0 <version> <digest>}"
IMAGE="quay.io/openbao/openbao:$NEW_VERSION"
BAO_ADDR="${BAO_ADDR:-https://bao.internal.company.com:8200}"
BAO_CACERT="${BAO_CACERT:-/opt/openbao/tls/ca.pem}"

echo "=== OpenBao Upgrade: $(cat /opt/openbao/config/version.env | grep VERSION) → $NEW_VERSION ==="

echo "[1/6] Creating pre-upgrade snapshot..."
/opt/openbao/scripts/snapshot-backup.sh

echo "[2/6] Pulling new image: $IMAGE"
docker pull "$IMAGE"

echo "[3/6] Verifying image digest..."
ACTUAL_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' "$IMAGE" | cut -d@ -f2)
if [ "$ACTUAL_DIGEST" != "$EXPECTED_DIGEST" ]; then
  echo "ERROR: Digest mismatch!"
  echo "  Expected : $EXPECTED_DIGEST"
  echo "  Actual   : $ACTUAL_DIGEST"
  exit 1
fi
echo "  Digest verified: $ACTUAL_DIGEST"

echo "[4/6] Stopping current container..."
sudo systemctl stop openbao-docker.service

echo "[5/6] Updating version reference..."
sed -i "s|OPENBAO_VERSION=.*|OPENBAO_VERSION=$NEW_VERSION|" /opt/openbao/config/version.env
sed -i "s|OPENBAO_DIGEST=.*|OPENBAO_DIGEST=$EXPECTED_DIGEST|" /opt/openbao/config/version.env

# Update the image tag in the systemd unit
sudo sed -i "s|quay.io/openbao/openbao:[^ ]*|$IMAGE|" \
  /etc/systemd/system/openbao-docker.service
sudo systemctl daemon-reload

echo "[6/6] Starting with new image..."
sudo systemctl start openbao-docker.service

# Wait for it to come up
sleep 10
if /opt/openbao/scripts/health-check.sh; then
  echo "=== Upgrade successful to $NEW_VERSION ==="
else
  echo "=== UPGRADE FAILED — rolling back ==="
  sudo systemctl stop openbao-docker.service
  sudo sed -i "s|quay.io/openbao/openbao:[^ ]*|quay.io/openbao/openbao:<PREVIOUS_VERSION>|" \
    /etc/systemd/system/openbao-docker.service
  sudo systemctl daemon-reload
  sudo systemctl start openbao-docker.service
  echo "Rollback initiated. Check logs: journalctl -u openbao-docker.service"
  exit 1
fi
UPGRADE
chmod 750 /opt/openbao/scripts/upgrade.sh
chown openbao:openbao /opt/openbao/scripts/upgrade.sh
```

### 9.3 Configuration Changes

Configuration changes follow a strict review flow before being applied:

```
1. Edit config file in version-controlled repository
2. Validate HCL syntax: bao config validate /opt/openbao/config/openbao.hcl
3. Peer review (pull request / code review)
4. Apply to staging/test environment first
5. Merge and deploy to production
6. Send SIGHUP to reload config (for supported settings)
   OR perform a rolling restart for settings that require it
```

To reload OpenBao's configuration without a full restart (for settings that support it):

```bash
docker kill --signal=HUP openbao
```

For changes that require a restart (e.g., listener, storage, seal):

```bash
sudo systemctl restart openbao-docker.service
# Then unseal if using Shamir
```

---

## 10. Security Hardening Summary

### 10.1 Host OS Hardening

```bash
# Disable core dumps (prevent memory from being written to disk)
echo "* hard core 0" | sudo tee -a /etc/security/limits.conf
echo "fs.suid_dumpable = 0" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Prevent swap from containing sensitive data (disable swap entirely if possible)
sudo swapoff -a
# Comment out swap in /etc/fstab to persist

# Enable automatic security updates
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure unattended-upgrades

# Audit daemon (track file access on sensitive paths)
sudo apt install auditd -y
sudo auditctl -w /opt/openbao/config -p rwxa -k openbao-config
sudo auditctl -w /opt/openbao/tls -p rwxa -k openbao-tls
sudo auditctl -w /opt/openbao/data -p rwxa -k openbao-data
```

### 10.2 Docker Daemon Hardening

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "icc": false,
  "no-new-privileges": true,
  "live-restore": false,
  "userland-proxy": false,
  "log-driver": "journald",
  "log-opts": {
    "tag": "{{.Name}}"
  }
}
EOF

sudo systemctl restart docker
```

| Setting | Effect |
|---|---|
| `"icc": false` | Disables inter-container communication by default |
| `"no-new-privileges": true` | Default to no privilege escalation |
| `"live-restore": false` | Containers stop when Docker daemon stops (safer for stateful apps) |
| `"userland-proxy": false` | Uses iptables directly instead of userland proxy |

### 10.3 Regular Security Checks

```bash
# Check for image vulnerabilities (install trivy)
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin
trivy image quay.io/openbao/openbao:2.5.0

# Review who has access to the Docker socket (should be minimal)
ls -la /var/run/docker.sock
getent group docker

# Check audit logs periodically
tail -f /opt/openbao/logs/audit.log | python3 -m json.tool | less
```

---

## 11. Observability

### 11.1 Container Logs via Journald

```bash
# Live container logs
sudo journalctl -u openbao-docker.service -f

# Last 100 lines
sudo journalctl -u openbao-docker.service -n 100

# Structured log output (since logs are JSON format)
sudo journalctl -u openbao-docker.service -o json | jq '.MESSAGE | fromjson'
```

### 11.2 Health Check Integration

Add a healthcheck to the container (this also feeds into `docker ps` output):

```bash
# Add to docker run or systemd ExecStart:
--health-cmd="bao status -address=https://127.0.0.1:8200 -ca-cert=/openbao/tls/ca.pem 2>&1 | grep -q 'Sealed.*false'" \
--health-interval=30s \
--health-timeout=10s \
--health-retries=3 \
--health-start-period=30s
```

### 11.3 Prometheus Metrics

OpenBao exposes metrics at `/v1/sys/metrics?format=prometheus`. Configure Prometheus to scrape it:

```yaml
# prometheus.yml scrape config
scrape_configs:
  - job_name: 'openbao'
    scheme: https
    tls_config:
      ca_file: /path/to/ca.pem
    bearer_token: '<metrics-token>'
    static_configs:
      - targets: ['bao.internal.company.com:8200']
    metrics_path: '/v1/sys/metrics'
    params:
      format: ['prometheus']
```

Create a minimal metrics-reader policy and token:

```bash
bao policy write metrics-reader - <<EOF
path "sys/metrics" {
  capabilities = ["read"]
}
EOF

bao token create \
  -policy=metrics-reader \
  -ttl=0 \
  -no-default-policy \
  -orphan \
  -display-name=prometheus-scraper
```

---

## 12. Directory & Files Reference

```
/opt/openbao/
├── config/
│   ├── openbao.hcl          # Main server configuration
│   └── version.env          # Pinned image version and digest
├── data/                    # Raft storage (never delete; back up via snapshot only)
├── tls/
│   ├── fullchain.pem        # Server certificate (+ intermediate chain)
│   ├── ca.pem               # CA certificate (for client verification)
│   └── privkey.pem          # Private key (chmod 600, owner only)
├── logs/
│   ├── audit.log            # Structured audit trail (every API call)
│   └── openbao.log          # Application log
├── scripts/
│   ├── health-check.sh      # Health check
│   ├── snapshot-backup.sh   # Raft snapshot
│   └── upgrade.sh           # Version upgrade procedure
└── snapshots/               # Raft snapshot files (chmod 600)

/etc/systemd/system/
└── openbao-docker.service   # Systemd unit

/etc/logrotate.d/
└── openbao                  # Log rotation config

/etc/docker/
└── daemon.json              # Docker daemon hardening
```

---

## 13. Quick-Reference: Day-to-Day Operations

| Task | Command |
|---|---|
| Start service | `systemctl start openbao-docker` |
| Stop service | `systemctl stop openbao-docker` |
| View live logs | `journalctl -u openbao-docker -f` |
| Check health | `/opt/openbao/scripts/health-check.sh` |
| Unseal | `bao operator unseal` |
| Take snapshot | `/opt/openbao/scripts/snapshot-backup.sh` |
| Reload config (SIGHUP) | `docker kill --signal=HUP openbao` |
| Upgrade version | `/opt/openbao/scripts/upgrade.sh <version> <digest>` |
| Check seal status | `bao status` |
| List audit devices | `bao audit list` |

---

*This guide establishes the baseline for a secure, production-grade OpenBao deployment on a bare server. The next phase of maturity — scripting the full initialization flow — will build on this directory layout and operational model, making automation straightforward.*
