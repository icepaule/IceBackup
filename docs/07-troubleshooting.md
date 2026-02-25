# Troubleshooting

## Tunnel is Down

```mermaid
flowchart TD
    A[Tunnel not working] --> B{Check autossh logs}
    B -->|Connection refused| C[Check Proxmox SSH service]
    B -->|Permission denied| D[Check authorized_keys on Proxmox]
    B -->|Host key changed| E[Regenerate known_hosts]
    B -->|Timeout| F[Check network/firewall]
    C --> G[Restart tunnel container]
    D --> G
    E --> G
    F --> G

    style A fill:#ffebee
    style G fill:#e8f5e9
```

```bash
# Check tunnel logs
docker logs autossh-pbs --tail 20
docker logs autossh-abb --tail 20

# Test SSH connection manually (from Synology)
ssh -i /path/to/ssh-keys/id_tunnel \
  -o UserKnownHostsFile=/path/to/ssh-keys/known_hosts \
  tunnel-synology@<PROXMOX_HOST>

# Restart tunnel containers
docker compose restart autossh-pbs autossh-abb
```

## PBS Not Reachable

```bash
# Check container status
docker compose ps

# Test PBS directly (from Synology)
curl -k https://localhost:8007/api2/json/version

# Test via tunnel (from Proxmox)
curl -k https://127.0.0.1:8007/api2/json/version
```

If PBS container keeps restarting:
```bash
# Check PBS logs for startup errors
docker logs proxmox-backup-server --tail 100

# Common issues:
# - tmpfs not mounted (kernel 4.4 requirement)
# - Permission issues on data directories
# - Port 8007 already in use
```

## Backup Fails

```bash
# Check PBS logs
docker logs proxmox-backup-server --tail 50

# On Proxmox: check task log
cat /var/log/vzdump/*.log

# Check disk space
df -h /path/to/backup/pbs-datastore/

# Check datastore status
docker exec proxmox-backup-server \
  proxmox-backup-manager datastore show vm-backups
```

### Common Backup Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `connection refused` | Tunnel down | Restart autossh-pbs |
| `authentication failed` | Wrong password/fingerprint | Re-add storage with correct credentials |
| `datastore full` | No space left | Run prune + GC, or add storage |
| `snapshot failed` | VM agent not responding | Check QEMU Guest Agent |
| `timeout` | Network too slow | Increase timeout, check bandwidth |

## PBS Web UI Not Accessible

- URL must use **HTTPS**: `https://<SYNOLOGY_IP>:8007`
- Browser warning about self-signed certificate is **normal** - accept it
- Login: `root@pam` with realm **"Linux PAM standard authentication"**
- If login fails, reset password:
  ```bash
  docker exec proxmox-backup-server \
    proxmox-backup-manager user update root@pam --password <NEW_PASSWORD>
  ```

## Host Key Changed (After Proxmox Reinstall)

```bash
# On Synology: regenerate known_hosts
ssh -o StrictHostKeyChecking=no -o BatchMode=yes <PROXMOX_HOST> echo 2>&1
grep <PROXMOX_HOST> ~/.ssh/known_hosts > /path/to/ssh-keys/known_hosts

# Restart tunnels
docker compose restart autossh-pbs autossh-abb
```

## ABB Agent Registration Fails

```bash
# On Proxmox: check if tunnel port 5510 is listening
ss -tlnp | grep 5510

# If not listening, check autossh-abb container
docker logs autossh-abb --tail 20

# Try registration with verbose output
abb-cli -c register -s 127.0.0.1 -p 5510 -u <DSM_USER> -w <DSM_PASS>
```

## ABB Snapshot Errors

If ABB reports "Failed to take a snapshot" (Error Code: fffffffffffffffe):

1. **Check filesystem type**: ABB snapshots require LVM, btrfs, or ext4 with sufficient space
2. **Check available space**: Snapshots need free space in the volume group
3. **Workaround**: Disable snapshot in ABB task settings (reduces consistency guarantee)

```bash
# On the affected host, check filesystem
lsblk -f
df -h
lvs  # Check LVM status
```

## Container Won't Start on Synology (Kernel 4.4)

Synology uses an older kernel (4.4.x). If PBS container fails:

1. Ensure `tmpfs` is mounted on `/run` in the compose file
2. Check if the Docker image requires newer kernel features
3. Try pinning to a specific image version instead of `latest`

```yaml
# Required in docker-compose.yml for kernel 4.4
tmpfs:
  - /run
```
