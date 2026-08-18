# Docker Swarm Storage

Storage strategies for stateful applications in Docker Swarm — from simple local volumes to distributed storage with NFS, GlusterFS, and Ceph.

## Storage Types

| Type | Use Case | Pros | Cons |
|------|----------|------|------|
| Local (default) | Single-node apps, temporary data | Simple, fast | No HA, data lost if node fails |
| Shared (NFS, CIFS) | Multi-node apps needing shared data | HA, persistent | Network overhead, potential bottleneck |
| Distributed (GlusterFS, Ceph) | Large-scale, high-performance | Scalable, redundant | Complex setup, resource intensive |

## NFS Storage Setup

### NFS Server

```bash
# Install NFS server
sudo apt update
sudo apt install -y nfs-kernel-server

# Create shared directories
sudo mkdir -p /srv/nfs/swarm/{volumes,configs,secrets}
sudo chown nobody:nogroup /srv/nfs/swarm
sudo chmod 755 /srv/nfs/swarm

# Configure exports
sudo tee /etc/exports << 'EOF'
/srv/nfs/swarm 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash,no_all_squash)
EOF

# Apply configuration
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server

# Open firewall ports
sudo ufw allow from 192.168.1.0/24 to any port nfs
```

### TrueNAS/FreeNAS Integration

If using TrueNAS/FreeNAS as your NFS server:

1. Create dataset: `tank/docker-swarm`
2. Set permissions: `chmod 755`, owner `root:wheel`
3. Create NFS share with:
   - Authorized Networks: `192.168.1.0/24`
   - Maproot User: `root`
   - Maproot Group: `wheel`

### NFS Client Setup (All Swarm Nodes)

Run on every swarm node:

```bash
# Install NFS client
sudo apt update
sudo apt install -y nfs-common

# Create mount points
sudo mkdir -p /mnt/nfs/swarm/{volumes,configs,secrets}

# Add to fstab for persistent mounting
echo "192.168.1.100:/srv/nfs/swarm /mnt/nfs/swarm nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# Mount the NFS share
sudo mount -a

# Verify mount
df -h /mnt/nfs/swarm
```

### Create Docker NFS Volumes

```bash
#!/bin/bash
# create-nfs-volumes.sh

NFS_SERVER="192.168.1.100"
NFS_PATH="/srv/nfs/swarm"

# Create volume directories on NFS server
ssh root@${NFS_SERVER} "mkdir -p ${NFS_PATH}/{postgres,redis,grafana,prometheus,elasticsearch}"

# Create Docker volumes using NFS
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=${NFS_SERVER},rw \
  --opt device=:${NFS_PATH}/postgres \
  postgres_data

docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=${NFS_SERVER},rw \
  --opt device=:${NFS_PATH}/redis \
  redis_data

docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=${NFS_SERVER},rw \
  --opt device=:${NFS_PATH}/grafana \
  grafana_data

docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=${NFS_SERVER},rw \
  --opt device=:${NFS_PATH}/prometheus \
  prometheus_data

docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=${NFS_SERVER},rw \
  --opt device=:${NFS_PATH}/elasticsearch \
  elasticsearch_data

echo "NFS volumes created successfully"
```

## GlusterFS Setup

For high-performance distributed storage across swarm nodes:

```bash
# Install GlusterFS on all nodes
sudo apt update
sudo apt install -y glusterfs-server

# Start and enable GlusterFS
sudo systemctl start glusterd
sudo systemctl enable glusterd

# On the first node, probe peers
sudo gluster peer probe 192.168.1.102
sudo gluster peer probe 192.168.1.103
sudo gluster peer probe 192.168.1.104

# Create a replicated volume (3-way replica)
sudo gluster volume create swarm-storage replica 3 \
  192.168.1.101:/data/glusterfs \
  192.168.1.102:/data/glusterfs \
  192.168.1.103:/data/glusterfs \
  force

# Start the volume
sudo gluster volume start swarm-storage

# Mount on all nodes
sudo mkdir -p /mnt/glusterfs
echo "localhost:/swarm-storage /mnt/glusterfs glusterfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

## Ceph Integration

For enterprise-grade distributed storage:

```bash
# Install ceph-common on all swarm nodes
sudo apt update
sudo apt install -y ceph-common

# Copy ceph.conf and keyring from existing Ceph cluster

# Create RBD pool for Docker volumes
sudo ceph osd pool create docker-volumes 64
sudo ceph osd pool application enable docker-volumes rbd

# Create RBD images for volumes
sudo rbd create --size 10G docker-volumes/postgres-data
sudo rbd create --size 5G docker-volumes/redis-data
sudo rbd create --size 20G docker-volumes/grafana-data
```

## Stack Examples

### PostgreSQL with NFS

```yaml
version: "3.8"
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - type: volume
        source: postgres_nfs_data
        target: /var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == nfs

volumes:
  postgres_nfs_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/srv/nfs/swarm/postgres"
```

### High-Availability Database

```yaml
version: "3.8"
services:
  postgres-primary:
    image: postgres:15
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.database == primary
    networks:
      - db_network

  postgres-replica:
    image: postgres:15
    environment:
      PGUSER: replicator
      POSTGRES_PASSWORD: rep_password
    volumes:
      - postgres_replica_data:/var/lib/postgresql/data
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.database == replica
    networks:
      - db_network

volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/srv/nfs/swarm/postgres-primary"

  postgres_replica_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/srv/nfs/swarm/postgres-replica"

networks:
  db_network:
    driver: overlay
    internal: true
```

## Backup Strategy

```bash
#!/bin/bash
# backup-volumes.sh

NFS_PATH="/mnt/nfs/swarm"
BACKUP_PATH="/backup/docker-swarm"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p "${BACKUP_PATH}/${DATE}"

# Backup each volume
for volume in postgres redis grafana prometheus; do
    echo "Backing up ${volume}..."
    tar -czf "${BACKUP_PATH}/${DATE}/${volume}.tar.gz" -C "${NFS_PATH}" "${volume}"
done

# Cleanup old backups (keep 7 days)
find "${BACKUP_PATH}" -type d -mtime +7 -exec rm -rf {} +

echo "Backup completed: ${BACKUP_PATH}/${DATE}"
```

## Monitoring Storage

### Disk Usage

```bash
#!/bin/bash
# storage-monitor.sh

echo "=== Docker Storage Usage ==="
docker system df -v

echo -e "\n=== NFS Mount Status ==="
mount | grep nfs

echo -e "\n=== Disk Space ==="
df -h
```

### Health Check Script

```bash
#!/bin/bash
# storage-health.sh

# Check NFS connectivity
if ! showmount -e 192.168.1.100 >/dev/null 2>&1; then
    echo "WARNING: NFS server not accessible"
    exit 1
fi

# Check mount points
if ! mountpoint -q /mnt/nfs/swarm; then
    echo "ERROR: NFS not mounted"
    exit 1
fi

# Check disk space
usage=$(df /mnt/nfs/swarm | awk 'NR==2 {print $5}' | sed 's/%//')
if [ "$usage" -gt 85 ]; then
    echo "WARNING: Storage usage high: ${usage}%"
fi

echo "Storage health: OK"
```

## Troubleshooting

### NFS Mount Issues

```bash
# Debug NFS connectivity
showmount -e NFS_SERVER_IP
rpcinfo -p NFS_SERVER_IP

# Check NFS logs
sudo journalctl -u nfs-kernel-server
tail -f /var/log/syslog | grep nfs
```

### Volume Permission Issues

```bash
# Fix ownership for common services
sudo chown -R 999:999 /mnt/nfs/swarm/postgres
sudo chown -R 1000:1000 /mnt/nfs/swarm/grafana
sudo chown -R 65534:65534 /mnt/nfs/swarm/prometheus
```

### Performance Testing

```bash
# Test NFS write performance
dd if=/dev/zero of=/mnt/nfs/swarm/test.dat bs=1M count=1000
rm /mnt/nfs/swarm/test.dat

# Check network latency to NFS server
ping -c 10 NFS_SERVER_IP
```

## Best Practices

| Area | Recommendation |
|------|----------------|
| Performance | Use SSDs for database storage, separate data and log volumes |
| Filesystem | Tune with `noatime`, `data=writeback` where appropriate |
| Security | Use VPN or private networks for storage traffic |
| Access control | Implement proper user/group permissions per service |
| Encryption | Use encrypted volumes for sensitive data |
| Backups | Automate with cron, encrypt at rest and in transit |
| Monitoring | Alert on usage > 85%, check mount health regularly |
| Placement | Use node labels and constraints to pin stateful services |
