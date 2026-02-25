# Setup Guide (Complete Installation)

## Prerequisites

- Synology NAS with DSM 7.2+ and Docker/Container Manager installed
- Proxmox VE host accessible via SSH from Synology
- Active Backup for Business package installed on Synology
- Sufficient storage on the Synology (recommended: 2+ TB free)

## Directory Structure

```
<docker-dir>/proxmox-backup/              # Docker config
<docker-dir>/proxmox-backup/ssh-keys/     # SSH keys for tunnel
  id_tunnel                                # Private key (ed25519)
  id_tunnel.pub                            # Public key
  known_hosts                              # Host key of Proxmox

<backup-volume>/pbs-datastore/             # PBS backup data (chunkstore)
<backup-volume>/pbs-etc/                   # PBS configuration (persistent)
<backup-volume>/pbs-logs/                  # PBS logs
<backup-volume>/pbs-lib/                   # PBS lib (persistent)
```

---

## Step 1: Create Directories (Synology)

```bash
mkdir -p /path/to/docker/proxmox-backup/ssh-keys
mkdir -p /path/to/backup/pbs-datastore
mkdir -p /path/to/backup/pbs-etc
mkdir -p /path/to/backup/pbs-logs
mkdir -p /path/to/backup/pbs-lib
```

## Step 2: Generate SSH Keys (Synology)

```bash
ssh-keygen -t ed25519 \
  -f /path/to/docker/proxmox-backup/ssh-keys/id_tunnel \
  -N "" -C "synology-tunnel@proxmox"

# Populate known_hosts
ssh -o StrictHostKeyChecking=no -o BatchMode=yes <PROXMOX_HOST> echo 2>&1
grep <PROXMOX_HOST> ~/.ssh/known_hosts > /path/to/docker/proxmox-backup/ssh-keys/known_hosts
```

## Step 3: Create Tunnel User on Proxmox

On the Proxmox host as root:

```bash
useradd -m -s /usr/sbin/nologin tunnel-synology
mkdir -p /home/tunnel-synology/.ssh

cat > /home/tunnel-synology/.ssh/authorized_keys << 'EOF'
no-pty,permitlisten="8007",permitlisten="5510",command="/bin/false" <PASTE_PUBLIC_KEY_HERE>
EOF

chown -R tunnel-synology:tunnel-synology /home/tunnel-synology/.ssh
chmod 700 /home/tunnel-synology/.ssh
chmod 600 /home/tunnel-synology/.ssh/authorized_keys
```

The `authorized_keys` restrictions ensure the tunnel user can **only** forward ports 8007 and 5510 - no shell access, no other ports.

## Step 4: Create docker-compose.yml (Synology)

Copy the reference compose file from [docker-compose reference](08-docker-compose-reference.md) and adjust:

- `SSH_REMOTE_HOST` - your Proxmox hostname/IP
- `SSH_REMOTE_USER` - the tunnel user created in step 3
- Volume paths for PBS data, config, logs, lib
- Volume paths for SSH keys

## Step 5: Start Containers (Synology)

```bash
cd /path/to/docker/proxmox-backup
docker compose up -d
docker compose ps   # All 3 must show "Up"
```

## Step 6: Configure PBS (Synology)

```bash
# Create datastore
docker exec proxmox-backup-server \
  proxmox-backup-manager datastore create vm-backups /backups

# Create prune job (retention)
docker exec proxmox-backup-server \
  proxmox-backup-manager prune-job create daily-prune \
  --schedule "*-*-* 03:00:00" --store vm-backups \
  --keep-daily 30 --keep-weekly 8 --keep-monthly 6

# Garbage collection schedule
docker exec proxmox-backup-server \
  proxmox-backup-manager datastore update vm-backups \
  --gc-schedule "Sat *-*-* 04:00:00"

# Create backup user + set password
docker exec proxmox-backup-server \
  proxmox-backup-manager user create backup-client@pbs
docker exec proxmox-backup-server \
  proxmox-backup-manager user update backup-client@pbs --password <PASSWORD>

# Set ACL: backup permissions on datastore
docker exec proxmox-backup-server \
  proxmox-backup-manager acl update /datastore/vm-backups DatastoreBackup \
  --auth-id backup-client@pbs

# Set root password for web UI
docker exec proxmox-backup-server \
  proxmox-backup-manager user update root@pam --password <PASSWORD>

# Show fingerprint (needed for Proxmox storage config)
docker exec proxmox-backup-server \
  proxmox-backup-manager cert info | grep Fingerprint
```

## Step 7: Add PBS as Storage on Proxmox

On the Proxmox host:

```bash
pvesm add pbs synology-pbs \
  --server 127.0.0.1 --port 8007 \
  --datastore vm-backups \
  --username backup-client@pbs \
  --fingerprint "<PBS_FINGERPRINT>" \
  --password <PASSWORD>
```

## Step 8: Create Backup Job on Proxmox

**Web UI**: Datacenter > Backup > Add:
- Storage: `synology-pbs`
- Schedule: `01:00` (daily)
- Selection: All
- Mode: Snapshot
- Compression: ZSTD

**Or via CLI:**
```bash
vzdump --all --storage synology-pbs --mode snapshot --compress zstd --mailto root
```

## Step 9: Test Backup

```bash
# Backup a single VM (e.g., VM 101)
vzdump 101 --storage synology-pbs --mode snapshot --compress zstd
```

Verify:
- Proxmox Web UI: Storage > synology-pbs > Content
- PBS Web UI: `https://<SYNOLOGY_IP>:8007` > Datastore > vm-backups
