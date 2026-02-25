# Daily Operations

## Container Management

```bash
cd /path/to/docker/proxmox-backup

# Check status
docker compose ps

# Start all containers
docker compose up -d

# Stop all containers
docker compose down

# Restart individual containers
docker compose restart pbs
docker compose restart autossh-pbs
docker compose restart autossh-abb
```

## Checking Logs

```bash
# PBS server logs
docker logs proxmox-backup-server --tail 50

# Tunnel status
docker logs autossh-pbs --tail 20
docker logs autossh-abb --tail 20

# Follow logs in real-time
docker logs -f proxmox-backup-server
```

## Checking Backup Status

```bash
# On Proxmox: storage status
pvesm status --storage synology-pbs

# PBS: list datastore contents
docker exec proxmox-backup-server \
  proxmox-backup-client list --repository backup-client@localhost:vm-backups

# PBS: show snapshots for a specific VM
docker exec proxmox-backup-server \
  proxmox-backup-client snapshots --repository backup-client@localhost:vm-backups \
  --group vm/101
```

## Manual Backup (on Proxmox)

```bash
# Single VM
vzdump 101 --storage synology-pbs --mode snapshot --compress zstd

# Multiple VMs
vzdump 101,102,103 --storage synology-pbs --mode snapshot --compress zstd

# All VMs and CTs
vzdump --all --storage synology-pbs --mode snapshot --compress zstd
```

## Manual Prune / Garbage Collection

```bash
# Run prune manually
docker exec proxmox-backup-server \
  proxmox-backup-manager prune-job run daily-prune

# Run garbage collection manually
docker exec proxmox-backup-server \
  proxmox-backup-manager garbage-collection start vm-backups
```

## Storage Health

```bash
# Check disk usage on backup volume
df -h /path/to/backup/pbs-datastore/

# Check datastore details
docker exec proxmox-backup-server \
  proxmox-backup-manager datastore show vm-backups
```
