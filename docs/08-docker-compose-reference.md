# docker-compose.yml Reference

Annotated reference compose file. **Adjust all paths and hostnames before use.**

```yaml
version: '3.8'

services:
  # ============================================================
  # Proxmox Backup Server
  # Web UI + API on port 8007
  # ============================================================
  pbs:
    image: ayufan/proxmox-backup-server:latest
    container_name: proxmox-backup-server
    restart: unless-stopped
    ports:
      - "8007:8007"                          # PBS Web UI + API
    volumes:
      - /path/to/backup/pbs-datastore:/backups      # Backup data (chunkstore)
      - /path/to/backup/pbs-etc:/etc/proxmox-backup  # PBS config (persistent)
      - /path/to/backup/pbs-logs:/var/log/proxmox-backup  # Logs
      - /path/to/backup/pbs-lib:/var/lib/proxmox-backup   # Lib (persistent)
    tmpfs:
      - /run                                 # Required for Synology kernel 4.4
    environment:
      - TZ=Europe/Berlin
    labels:
      - "com.centurylinklabs.watchtower.enable=false"  # Don't auto-update
    networks:
      - pbs-net
    stop_grace_period: 60s                   # Allow graceful shutdown

  # ============================================================
  # Reverse SSH Tunnel for PBS
  # Makes PBS reachable from Proxmox as localhost:8007
  # ============================================================
  autossh-pbs:
    image: jnovack/autossh:latest
    container_name: autossh-pbs
    restart: unless-stopped
    environment:
      - SSH_REMOTE_HOST=<PROXMOX_HOST>       # Proxmox hostname or IP
      - SSH_REMOTE_PORT=22
      - SSH_REMOTE_USER=tunnel-synology      # Restricted SSH user
      - SSH_TARGET_HOST=proxmox-backup-server # PBS container (Docker DNS)
      - SSH_TARGET_PORT=8007
      - SSH_TUNNEL_PORT=8007                 # Port on Proxmox localhost
      - SSH_BIND_IP=127.0.0.1               # Only bind to localhost
      - SSH_MODE=-R                          # Reverse tunnel
      - SSH_KEY_FILE=/id_rsa
      - SSH_KNOWN_HOSTS_FILE=/known_hosts
      - SSH_STRICT_HOST_IP_CHECK=false
      - SSH_SERVER_ALIVE_INTERVAL=10         # Detect dead connections
      - SSH_SERVER_ALIVE_COUNT_MAX=3         # Max 30s to detect
    volumes:
      - ./ssh-keys/id_tunnel:/id_rsa:ro
      - ./ssh-keys/known_hosts:/known_hosts:ro
    labels:
      - "com.centurylinklabs.watchtower.enable=false"
    depends_on:
      - pbs
    networks:
      - pbs-net

  # ============================================================
  # Reverse SSH Tunnel for ABB Agent
  # Makes Synology port 5510 reachable from Proxmox as localhost:5510
  # ============================================================
  autossh-abb:
    image: jnovack/autossh:latest
    container_name: autossh-abb
    restart: unless-stopped
    environment:
      - SSH_REMOTE_HOST=<PROXMOX_HOST>       # Same Proxmox host
      - SSH_REMOTE_PORT=22
      - SSH_REMOTE_USER=tunnel-synology
      - SSH_TARGET_HOST=host.docker.internal  # Synology host
      - SSH_TARGET_PORT=5510                  # ABB agent port
      - SSH_TUNNEL_PORT=5510                  # Port on Proxmox localhost
      - SSH_BIND_IP=127.0.0.1
      - SSH_MODE=-R
      - SSH_KEY_FILE=/id_rsa
      - SSH_KNOWN_HOSTS_FILE=/known_hosts
      - SSH_STRICT_HOST_IP_CHECK=false
      - SSH_SERVER_ALIVE_INTERVAL=10
      - SSH_SERVER_ALIVE_COUNT_MAX=3
    extra_hosts:
      - "host.docker.internal:host-gateway"  # Resolve to Docker host IP
    volumes:
      - ./ssh-keys/id_tunnel:/id_rsa:ro
      - ./ssh-keys/known_hosts:/known_hosts:ro
    labels:
      - "com.centurylinklabs.watchtower.enable=false"
    networks:
      - pbs-net

networks:
  pbs-net:
    name: pbs-net
```

## Key Configuration Points

### PBS Container

| Setting | Purpose |
|---------|---------|
| `tmpfs: /run` | Required on Synology (kernel 4.4 doesn't support certain fs operations) |
| `stop_grace_period: 60s` | Allow PBS to finish writing before shutdown |
| Watchtower disabled | Prevent uncontrolled updates of backup infrastructure |

### Autossh Containers

| Setting | Purpose |
|---------|---------|
| `SSH_BIND_IP=127.0.0.1` | Only bind tunnel to localhost (security) |
| `SSH_SERVER_ALIVE_INTERVAL=10` | Check connection every 10 seconds |
| `SSH_SERVER_ALIVE_COUNT_MAX=3` | Reconnect after 30s of no response |
| `SSH_MODE=-R` | Reverse tunnel (remote port forwarding) |
| `SSH_STRICT_HOST_IP_CHECK=false` | Don't check IP against known_hosts (hostname sufficient) |

### Network

All containers share the `pbs-net` network, allowing:
- `autossh-pbs` to reach `proxmox-backup-server` via Docker DNS
- `autossh-abb` to reach the Synology host via `host.docker.internal`
