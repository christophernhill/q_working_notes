# Apptainer Container Launchers and Monitoring

## Overview

This document describes how the three primary container types in the Quantori cluster infrastructure are launched and monitored on their respective host systems:

1. **OOD Containers** - Open OnDemand web portal services (ood001-004, ood-ha001-004)
2. **Login Containers** - SSH login nodes (login005-008, login-ha001-004)
3. **Compute Containers** - Warewulf-provisioned compute nodes (node4100, node3401, etc.)

Each deployment type uses Apptainer containers but with different launch mechanisms, monitoring approaches, and host architectures.

## Container Types and Host Mapping

### Host Systems Overview

The cluster uses physical "core" hosts to run containerized services:

| Host System | Role | Services Hosted |
|------------|------|-----------------|
| **core015** | Container Host | ood001, ood002, login005, login007, ww001 |
| **core016** | Container Host | ood-ha001, ood-ha002, login-ha001, login-ha002 |
| **core017** | Container Host | ood003, ood004, login006, login008 |
| **core018** | Container Host | ood-ha003, ood-ha004, login-ha003, login-ha004 |

### Service Distribution

**Production Environment:**
- **OOD Services**: 4 primary (ood001-004) + 4 HA proxies (ood-ha001-004)
- **Login Services**: 4 primary (login005-008) + 4 HA proxies (login-ha001-004)
- **Warewulf Services**: 1 primary (ww001)
- **Compute Nodes**: node4100, node3401, and others managed by ww001

## 1. OOD Container Launch and Monitoring

### Container Architecture

OOD containers run the Open OnDemand web portal, providing browser-based access to cluster resources including job submission, file management, and interactive applications.

**Container Image:** `ghcr.io/mit-orcd/ood-4.0:main_latest`

### Host Directory Structure

Each OOD container is deployed on a core host with the following directory layout:

```
/shared/services/ood/<servicename>/
├── bin/
│   ├── start.sh          # Container startup script
│   └── stop.sh           # Container shutdown script
├── containers/
│   └── container.sif     # OOD container image (~2GB)
├── overlay/
│   └── overlay.img       # Writable overlay (1GB default)
└── root/
    └── .ssh/
        ├── authorized_keys
        └── cluster        # SSH key for cluster access
```

### Container Launch Process

#### 1. Systemd Service Definition

Each OOD container is managed as a systemd service on its host:

**Service File:** `/etc/systemd/system/<servicename>.service`

```ini
[Unit]
Description=Container Service ood001
Wants=network-online.target
After=network-online.target
After=time-sync.target

[Service]
Type=forking
ExecStart=/shared/services/ood/ood001/bin/start.sh
ExecStop=/shared/services/ood/ood001/bin/stop.sh

[Install]
WantedBy=multi-user.target
```

#### 2. Container Start Script

**Script:** `/shared/services/ood/<servicename>/bin/start.sh`

```bash
#!/bin/bash

SCRIPT=$(realpath "$0")
DIRNAME=$(dirname "$SCRIPT")/../
OVERLAY=$DIRNAME/overlay/overlay.img

if [ -f $DIRNAME/bin/.service_env ]; then
    source $DIRNAME/bin/.service_env
fi

apptainer instance start \
  --boot \
  --network macvlan_ood001 \
  -B /home/systems/slurm/etc/slurm:/etc/slurm \
  -B /shared \
  -B /home \
  -B /orcd \
  -B $DIRNAME/root:/root \
  --overlay $OVERLAY \
  "$DIRNAME/containers/container.sif" ood001
```

**Key Parameters:**
- `--boot`: Runs systemd as PID 1 inside the container
- `--network macvlan_ood001`: Dedicated macvlan network giving container its own IP
- `-B /etc/slurm`: Bind mount for Slurm client configuration
- `-B /shared, /home, /orcd`: Cluster filesystem access
- `--overlay`: Writable layer for container-local persistence

#### 2a. Overlay Creation (First-Time Setup)

Before the container can be launched, a writable overlay filesystem must be created. This happens automatically during the initial deployment via the `master` role in Ansible.

**Overlay Creation Process:**

The Ansible playbook checks if the overlay already exists and creates it if needed:

```yaml
# From roles/master/tasks/main.yml

- name: Check if overlay.img exists
  stat:
    path: "{{ install_path }}/{{ servicename }}/overlay/overlay.img"
  register: overlay_img_stat

- name: Create overlay.img if it does not exist
  command: >
    apptainer overlay create -S -s {{ hostvars[service].apptainer_options.overlay_size | default(1024) }}
    {{ install_path }}/{{ servicename }}/overlay/overlay.img
  when: not overlay_img_stat.stat.exists
```

**Command Breakdown:**

```bash
apptainer overlay create \
  -S \          # Create sparse file (doesn't use full disk space immediately)
  -s 1024 \     # Size in MB (default 1024 MB = 1 GB)
  /shared/services/ood/ood001/overlay/overlay.img
```

**Command Options:**
- `-S`: Creates a sparse file that grows as needed rather than allocating all space immediately
- `-s <size>`: Size in megabytes (MB)
  - OOD containers: Default 1024 MB (1 GB)
  - Warewulf containers: 100000 MB (100 GB) for storing compute node images

**Manual Overlay Creation:**

If you need to create or recreate an overlay manually:

```bash
# Stop the container first
systemctl stop ood001.service

# Remove old overlay (if exists)
rm /shared/services/ood/ood001/overlay/overlay.img

# Create new overlay (1GB)
apptainer overlay create -S -s 1024 /shared/services/ood/ood001/overlay/overlay.img

# For larger overlay (10GB example)
apptainer overlay create -S -s 10240 /shared/services/ood/ood001/overlay/overlay.img

# Start the container
systemctl start ood001.service
```

**What the Overlay Stores:**

The overlay filesystem provides a writable layer on top of the read-only container image. Changes written inside the container are stored here:

- **Configuration changes**: Modified files in `/etc/` (if changed inside container)
- **Application data**: Data written to system directories
- **Package installations**: If packages are installed inside the running container
- **Temporary state**: Service-specific runtime data
- **Logs**: If services write logs to local filesystem (though many logs go to bind-mounted `/var/log`)

**Important Notes:**

1. **Persistence**: The overlay persists across container restarts. Data in the overlay survives when the container is stopped and restarted.

2. **Container Updates**: When deploying a new container image, the overlay is preserved unless `deploy_forced=true` is used, which removes and recreates the entire service directory including the overlay.

3. **Size Planning**:
   - **OOD**: 1GB is usually sufficient since most data is on bind mounts
   - **Login**: 1GB default, may need more if users install software
   - **Warewulf**: 100GB to store multiple compute node OS images

4. **Sparse Files**: Using the `-S` flag means the overlay file only uses actual disk space for data written to it, not the full allocated size.

5. **Backup Considerations**: The overlay contains container-specific state. To fully backup a container service, you need both the container image URL and the overlay file.

**Example: Checking Overlay Usage**

```bash
# Check overlay file size (apparent vs actual)
ls -lh /shared/services/ood/ood001/overlay/overlay.img

# Check actual disk usage
du -h /shared/services/ood/ood001/overlay/overlay.img

# Example output:
-rw-r--r-- 1 root root 1.0G Nov 15 10:30 overlay.img  # Apparent size
1.0G    overlay.img                                     # Actual usage (du)
# If sparse and not fully used:
256M    overlay.img                                     # Actual usage might be less
```

#### 3. Network Configuration

Each OOD container gets its own IP address via macvlan networking:

**Network Config:** `/etc/apptainer/network/macvlan_<servicename>.conflist`

The container appears as a separate host on the network:
- **DNS Name**: `ood001.inband`
- **Primary Access**: `https://orcd-ood.mit.edu` (via HAProxy)
- **Direct Access**: `https://ood001.inband`

#### 4. Container Stop Script

**Script:** `/shared/services/ood/<servicename>/bin/stop.sh`

```bash
#!/bin/bash

apptainer instance stop ood001
```

### Internal Services

Once the container boots, systemd inside the container starts these services:

1. **shibd** - Shibboleth daemon for federated authentication
2. **munge** - Authentication daemon for Slurm communication
3. **httpd** - Apache web server (port 443 with SSL)
4. **sshd** - SSH daemon for remote management

### OOD Deployment Example

For `ood001` running on `core015`:

```yaml
# From inventory/prod/hosts.yml
ood001:
  master_host: core015

# From inventory/prod/host_vars/ood001.yml
ansible_host: ood001.inband
container_url: oras://ghcr.io/mit-orcd/ood-4.0:main_latest
servicename: ood001
apptainer_options:
  network: 
    - macvlan_ood001
  mounts:
    - /home/systems/slurm/etc/slurm:/etc/slurm
    - /shared
    - /home
    - /orcd
    - $DIRNAME/root:/root
  boot: true
```

### OOD Monitoring

#### Host-Level Monitoring

From the host system (e.g., `core015`):

```bash
# Check systemd service status
systemctl status ood001.service

# View service logs
journalctl -u ood001.service -f

# List running Apptainer instances
apptainer instance list

# Example output:
INSTANCE NAME    PID      IP               IMAGE
ood001           123456   10.1.225.1       /shared/services/ood/ood001/containers/container.sif
```

#### Container-Level Monitoring

Access the running container:

```bash
# Execute commands inside the container
apptainer exec instance://ood001 systemctl status httpd

# Shell into the container
apptainer shell instance://ood001

# Inside container:
systemctl status shibd
systemctl status munge
systemctl status httpd
tail -f /var/log/httpd/ssl_error_log
tail -f /var/log/ondemand-nginx/error.log
```

#### Application-Level Monitoring

OOD-specific logs inside the container:

- **Apache Logs**: `/var/log/httpd/`
  - `ssl_access_log` - HTTPS requests
  - `ssl_error_log` - Apache errors
- **OOD Logs**: `/var/log/ondemand-nginx/`
  - `error.log` - OOD application errors
  - `access.log` - User access logs
- **Shibboleth Logs**: `/var/log/shibboleth/`
  - `shibd.log` - Authentication events

### OOD High-Availability Setup

The HA OOD containers (`ood-ha001` through `ood-ha004`) run HAProxy to load balance across the primary OOD instances:

```yaml
# From inventory/prod/hosts.yml
ha_ood:
  hosts:
    ood-ha001:
      master_host: core016
      priority: 200
    ood-ha002:
      master_host: core016
      priority: 100
  vars:
    frontends:
      internal:
        listen:
          port: 443
          ip: 10.1.225.31  # Virtual IP managed by keepalived
        hosts:
          - ood001
          - ood002
          - ood003
          - ood004
```

## 2. Login Container Launch and Monitoring

### Container Architecture

Login containers provide SSH access to the cluster with full development environments, authentication via Duo MFA, and access to cluster filesystems.

**Container Image:** `ghcr.io/mit-orcd/rocky-8.10-login:main_latest`

### Host Directory Structure

Similar to OOD containers, login containers use the same structure:

```
/shared/services/login/<servicename>/
├── bin/
│   ├── start.sh
│   └── stop.sh
├── containers/
│   └── container.sif     # Login container image (~4GB)
├── overlay/
│   └── overlay.img       # Writable overlay (1GB default)
└── root/
    └── .ssh/
        ├── authorized_keys
        └── cluster
```

### Container Launch Process

#### 1. Systemd Service

**Service File:** `/etc/systemd/system/<servicename>.service`

```ini
[Unit]
Description=Container Service login005
Wants=network-online.target
After=network-online.target
After=time-sync.target

[Service]
Type=forking
ExecStart=/shared/services/login/login005/bin/start.sh
ExecStop=/shared/services/login/login005/bin/stop.sh

[Install]
WantedBy=multi-user.target
```

#### 2. Container Start Script

**Script:** `/shared/services/login/<servicename>/bin/start.sh`

```bash
#!/bin/bash

SCRIPT=$(realpath "$0")
DIRNAME=$(dirname "$SCRIPT")/../
OVERLAY=$DIRNAME/overlay/overlay.img

apptainer instance start \
  --boot \
  --network macvlan_login005 \
  -B /home/systems/slurm/etc/slurm:/etc/slurm \
  -B /shared \
  -B /home \
  -B /orcd \
  -B /nfs \
  -B /programs \
  -B $DIRNAME/root:/root \
  -B /sys/fs/cgroup \
  -B /dev/fuse \
  --overlay $OVERLAY \
  "$DIRNAME/containers/container.sif" login005
```

**Additional Bind Mounts for Login:**
- `-B /sys/fs/cgroup`: For systemd cgroup management
- `-B /dev/fuse`: For user-space filesystem tools (e.g., sshfs)
- `-B /nfs, /programs`: Additional cluster storage paths

### Internal Services

Login containers run these key services:

1. **sshd** - SSH daemon (port 22) for user login
2. **systemd-logind** - User session management
3. **rsyslog** - System logging
4. **telegraf** - Metrics collection (if configured)
5. **slurmd** (client tools only, no daemon)
6. **set-lo-vip** - Loopback VIP configuration (for HA)
7. **custom-routing** - Custom routing for dual-network setups (if configured)

### Login Deployment Example

For `login005` running on `core015`:

```yaml
# From inventory/prod/hosts.yml
login005:
  master_host: core015

# From inventory/prod/host_vars/login005.yml
ansible_host: login005.inband
ansible_port: 22
container_url: oras://ghcr.io/mit-orcd/rocky-8.10-login:main_latest
servicename: login005
apptainer_options:
  network: 
    - macvlan_login005
  mounts:
    - /home/systems/slurm/etc/slurm:/etc/slurm
    - /shared
    - /home
    - /orcd
    - /nfs
    - /programs
    - $DIRNAME/root:/root
    - /sys/fs/cgroup
    - /dev/fuse
  boot: true
sshd:
  internal:
    duo: false
    rootlogin: true
    port: 22
    host: 0.0.0.0
```

### Login Monitoring

#### Host-Level Monitoring

From the host system (e.g., `core015`):

```bash
# Check systemd service status
systemctl status login005.service

# View service logs
journalctl -u login005.service -f

# List running instances
apptainer instance list

# Monitor resource usage
apptainer exec instance://login005 top
```

#### Container-Level Monitoring

```bash
# Check SSH service
apptainer exec instance://login005 systemctl status sshd

# Check active user sessions
apptainer exec instance://login005 who
apptainer exec instance://login005 w

# View authentication logs
apptainer exec instance://login005 tail -f /var/log/secure

# Check Slurm connectivity
apptainer exec instance://login005 sinfo
apptainer exec instance://login005 squeue
```

#### User Session Monitoring

Inside the container:

```bash
# List logged-in users
who

# Show user session details
loginctl list-sessions

# Monitor user resource usage
systemctl status user-*.slice

# Check failed login attempts
grep "Failed password" /var/log/secure
```

### Login High-Availability Setup

Login HA containers (`login-ha001` through `login-ha004`) use HAProxy for load balancing SSH connections:

```yaml
# From inventory/prod/hosts.yml
ha_login:
  hosts:
    login-ha001:
      master_host: core016
      priority: 200
    login-ha002:
      master_host: core016
      priority: 100
  vars:
    frontends:
      internal:
        listen:
          port: 22
          ip: 10.1.225.30  # Virtual IP (orcd-login.inband)
        hosts:
          - host: login005
            endpoint: internal
          - host: login006
            endpoint: internal
      external:
        listen:
          port: 22
          ip: 18.13.54.160  # Public IP (orcd-login.mit.edu)
        hosts:
          - host: login007
            endpoint: external
          - host: login008
            endpoint: external
```

**Load Balancing:**
- Internal users connect to `orcd-login.inband` (10.1.225.30)
- External users connect to `orcd-login.mit.edu` (18.13.54.160)
- HAProxy distributes connections across backend login containers
- Keepalived manages VIP failover between HA pairs

## 3. Compute Container Launch and Monitoring (Warewulf)

### Container Architecture

Compute node containers are **stateless diskless nodes** that boot from the network using Warewulf provisioning. Unlike OOD and Login containers that run as Apptainer instances on core hosts, compute containers **become the entire operating system** of the physical compute node.

**Container Images:**
- `ghcr.io/mit-orcd/rocky-8.10-compute:main_latest`
- `ghcr.io/mit-orcd/rocky-9.5-compute:main_latest`
- `ghcr.io/mit-orcd/ubuntu-22.04-compute:main_latest`

### Warewulf Container (ww001)

The Warewulf service itself runs as an Apptainer container on a core host (e.g., `core015`):

```yaml
# From inventory/prod/host_vars/ww001.yml
ansible_host: ww001
servicename: ww001
master_host: core015
container_url: oras://ghcr.io/mit-orcd/warewulf-4.6.1:main_latest
apptainer_options:
  overlay_size: 100000  # 100GB overlay for container storage
  network:
    - macvlan_ww001
    - ipmi_bridge_ww001  # For IPMI access to compute nodes
  mounts:
    - /etc/slurm
    - /shared
    - /home
    - /orcd
    - $DIRNAME/root:/root
  boot: true
managed_nodes:
  - node4100
  - node3401
```

**Directory Structure:**

```
/shared/services/ww/<servicename>/
├── bin/
│   ├── start.sh
│   └── stop.sh
├── containers/
│   └── container.sif     # Warewulf container (~8GB)
├── overlay/
│   └── overlay.img       # Large overlay (100GB) for compute images
└── root/
    └── .ssh/
        ├── authorized_keys
        └── cluster
```

### Warewulf Container Launch

The Warewulf container is launched the same way as OOD/Login:

```bash
# /shared/services/ww/ww001/bin/start.sh
apptainer instance start \
  --boot \
  --network macvlan_ww001,ipmi_bridge_ww001 \
  -B /etc/slurm \
  -B /shared \
  -B /home \
  -B /orcd \
  -B $DIRNAME/root:/root \
  --overlay overlay.img \
  "$DIRNAME/containers/container.sif" ww001
```

### Compute Node Boot Process

Once the Warewulf container is running, it provisions physical compute nodes:

#### 1. PXE Boot Sequence

```
Physical Compute Node (e.g., node4100)
    ↓ Power On
BIOS/UEFI → PXE Boot
    ↓ DHCP Request
Warewulf DHCP Response (IP + boot info)
    ↓ TFTP Download
iPXE Bootloader
    ↓ HTTP Download
Kernel + Initramfs (from ww001)
    ↓ Boot Kernel
Dracut Initramfs Scripts
    ↓ HTTP Download
Container Image → tmpfs in RAM
    ↓ Mount Overlays
System Overlays + Runtime Overlays
    ↓ pivot_root
Container becomes root filesystem
    ↓ exec
systemd as PID 1 → Starts Services
```

#### 2. Container Import and Configuration

Inside the ww001 container, compute images are imported:

```bash
# Import container images
wwctl container import docker://ghcr.io/mit-orcd/rocky-8.10-compute:main_latest rocky-8.10-compute
wwctl container import docker://ghcr.io/mit-orcd/rocky-9.5-compute:main_latest rocky-9.5-compute

# Configure node
wwctl node set node4100 \
  --container rocky-8.10-compute \
  --ipaddr 10.1.34.100 \
  --netmask 255.255.0.0 \
  --gateway 10.1.224.51

# Add system overlays (persistent configs)
wwctl overlay import system /path/to/overlay

# List configured nodes
wwctl node list
```

#### 3. Node Configuration Example

Each compute node has a configuration file in the Warewulf container:

**File:** `/etc/warewulf/nodes/node4100.yml`

```yaml
node4100:
  cluster name: prod
  image name: rocky-8.10-compute
  ipmi:
    ipaddr: 192.168.1.100
  profiles:
    - compute-base
  system overlay:
    - wwinit
    - generic
  runtime overlay:
    - compute-runtime
  network devices:
    internal:
      type: ethernet
      device: boot0
      ipaddr: 10.1.34.100
      netmask: 255.255.0.0
      gateway: 10.1.224.51
      primary: true
    ib:
      type: infiniband
      device: ib0
      ipaddr: 172.16.34.100
      netmask: 255.255.0.0
      primary: false
  primary network: internal
```

#### 4. Boot Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Physical Compute Node (node4100)                            │
│                                                              │
│ ┌────────────────────────────────────────────────────────┐ │
│ │ Kernel (in RAM)                                         │ │
│ │   ↓                                                      │ │
│ │ Container Filesystem (in RAM - tmpfs)                   │ │
│ │   ├─ /usr, /bin, /sbin (read-only from container)      │ │
│ │   ├─ /etc (overlays applied from ww001)                │ │
│ │   ├─ /var (tmpfs - ephemeral)                          │ │
│ │   ├─ /tmp (tmpfs - ephemeral)                          │ │
│ │   ├─ /scratch (local RAID - persistent)                │ │
│ │   ├─ /shared (NFS from cluster)                        │ │
│ │   ├─ /home (NFS from cluster)                          │ │
│ │   └─ /orcd (NFS from cluster)                          │ │
│ │                                                          │ │
│ │ Services:                                               │ │
│ │   ├─ slurmd (Slurm compute daemon)                     │ │
│ │   ├─ munged (authentication)                           │ │
│ │   ├─ sshd (remote access)                              │ │
│ │   ├─ nvidia driver (GPU nodes)                         │ │
│ │   └─ telegraf (monitoring)                             │ │
│ └────────────────────────────────────────────────────────┘ │
│                                                              │
│ Network Interfaces:                                         │
│   eth0 (boot0): 10.1.34.100 - Primary network              │
│   ib0: 172.16.34.100 - InfiniBand                          │
└─────────────────────────────────────────────────────────────┘
                           ↑
                           │ PXE Boot, Image Download
                           ↓
┌─────────────────────────────────────────────────────────────┐
│ Warewulf Container (ww001) on core015                      │
│                                                              │
│ Services:                                                   │
│   ├─ warewulfd (provisioning daemon)                       │
│   ├─ dhcpd (IP assignment)                                 │
│   ├─ tftpd (iPXE boot files)                               │
│   └─ httpd (kernel, initramfs, container image serving)    │
│                                                              │
│ Container Storage:                                          │
│   /var/lib/warewulf/                                       │
│     ├─ chroots/                                            │
│     │   ├─ rocky-8.10-compute/                            │
│     │   ├─ rocky-9.5-compute/                             │
│     │   └─ ubuntu-22.04-compute/                          │
│     ├─ overlays/                                           │
│     │   ├─ system/                                         │
│     │   └─ runtime/                                        │
│     └─ ipxe/                                               │
└─────────────────────────────────────────────────────────────┘
```

### Compute Node Characteristics

**Stateless Operation:**
- No persistent OS on local disks
- Entire OS lives in RAM (8-32GB depending on node)
- Rebooting downloads fresh image from ww001
- Perfect consistency across all nodes
- Failed nodes recover by rebooting

**Storage:**
- **Root filesystem**: RAM tmpfs (ephemeral)
- **Local scratch**: RAID arrays configured at boot (ephemeral per boot)
- **Shared storage**: NFS mounts from cluster (persistent)

**Services:**
- **slurmd**: Compute daemon for job execution
- **munged**: Authentication for Slurm communication
- **sshd**: Remote access for debugging
- **nvidia driver**: GPU support (on GPU nodes)
- **MOFED driver**: InfiniBand support
- **telegraf**: Metrics collection

### Warewulf Monitoring

#### Warewulf Container Monitoring

From the Warewulf container host (e.g., `core015`):

```bash
# Check Warewulf container status
systemctl status ww001.service

# Access Warewulf container
apptainer shell instance://ww001

# Inside ww001 container:
wwctl node list
wwctl node status node4100
wwctl container list
wwctl overlay list
```

#### Compute Node Monitoring

From the Warewulf container:

```bash
# Check node power status
wwctl node power status node4100

# Power on node
wwctl node power on node4100

# Power off node
wwctl node power off node4100

# View node configuration
wwctl node list -a node4100

# Check provisioning logs
journalctl -u warewulfd -f
tail -f /var/log/httpd/access_log  # Image download requests
```

#### From Cluster Management

Once compute nodes are booted:

```bash
# Check node status via Slurm
sinfo -N node4100
scontrol show node node4100

# SSH directly to compute node
ssh node4100

# On compute node:
systemctl status slurmd
top
nvidia-smi  # GPU nodes
ibstat      # InfiniBand status
df -h
cat /proc/meminfo
```

#### Monitoring Node Boot Process

```bash
# From ww001 container, watch DHCP requests
tail -f /var/log/messages | grep dhcpd

# Watch HTTP downloads of kernel/initramfs/container
tail -f /var/log/httpd/access_log

# Example boot sequence logs:
# DHCP: node4100 requests IP
# TFTP: node4100 downloads iPXE
# HTTP: node4100 downloads kernel
# HTTP: node4100 downloads initramfs
# HTTP: node4100 downloads container image (can be 2-8GB)
```

### Compute Node vs Container-Based Services

| Aspect | Compute Nodes (ww-managed) | OOD/Login Containers |
|--------|---------------------------|----------------------|
| **Launch Method** | PXE network boot | `apptainer instance start` |
| **Host OS** | None (diskless) | Persistent Linux on core hosts |
| **Container Role** | **IS** the entire OS | Isolated service in namespace |
| **Root Filesystem** | RAM tmpfs | Container image + overlay |
| **Systemd** | PID 1 of physical node | PID 1 inside container namespace |
| **Networking** | Native (kernel direct) | macvlan (isolated) |
| **Persistence** | None (stateless) | Host + overlay persistent |
| **Updates** | Reboot = fresh image | Container restart or live update |
| **Management** | Centralized (Warewulf) | Individual (systemd + Ansible) |
| **Scale** | 1000s of nodes | 10s of containers |

## Deployment and Management

### Deploying OOD/Login Containers

**Playbooks:**
- `ood.yml` - Deploy OOD containers
- `ha_ood.yml` - Deploy HA OOD proxy containers
- `login.yml` - Deploy login containers
- `ha_login.yml` - Deploy HA login proxy containers

**Example: Deploy OOD Container**

```bash
cd orcd-quantori-warewulf-work

# Deploy specific OOD instance
ansible-playbook -i inventory/prod/hosts.yml ood.yml -l ood001

# Deploy all OOD instances
ansible-playbook -i inventory/prod/hosts.yml ood.yml

# Force redeploy (rebuild from scratch)
ansible-playbook -i inventory/prod/hosts.yml ood.yml -e deploy_forced=true
```

**Deployment Process:**

1. **Master Role** (on core host):
   - Install Apptainer with SUID support
   - Create directory structure
   - Generate network configuration
   - Pull container image from GHCR
   - Create overlay filesystem
   - Generate start/stop scripts
   - Create systemd service
   - Start and enable service

2. **Service Role** (inside container):
   - Configure application (OOD or login-specific)
   - Deploy configuration files
   - Set up SSL certificates
   - Configure authentication (Shibboleth, SSSD, etc.)
   - Start internal services

### Deploying Warewulf and Compute Nodes

**Playbooks:**
- `warewulf.yml` - Deploy Warewulf container
- Various playbooks in `orcd-quantori-platform-work` to build compute images

**Example: Deploy Warewulf**

```bash
cd orcd-quantori-warewulf-work

# Deploy Warewulf container
ansible-playbook -i inventory/prod/hosts.yml warewulf.yml -l ww001

# Inside ww001 container, provision compute nodes:
wwctl node set node4100 --container rocky-8.10-compute
wwctl node power cycle node4100
```

### Container Updates

#### Updating OOD/Login Containers

**Option 1: Live Updates (configuration only)**

```bash
# Update configs without container rebuild
ansible-playbook -i inventory/prod/hosts.yml ood.yml --tags live_updates

# Restart services inside container
apptainer exec instance://ood001 systemctl restart httpd
```

**Option 2: Full Container Update**

```bash
# Build new container image
cd orcd-quantori-platform-work/ansible
ansible-playbook ood-4.0.yml

# Push to registry
apptainer push ood-4.0.sif oras://ghcr.io/mit-orcd/ood-4.0:main_latest

# Redeploy
cd orcd-quantori-warewulf-work
ansible-playbook -i inventory/prod/hosts.yml ood.yml -e deploy_forced=true -l ood001
```

#### Updating Compute Nodes

```bash
# Build new compute image
cd orcd-quantori-platform-work/ansible
ansible-playbook rocky-8.10-compute.yml

# Push to GHCR
apptainer push rocky-8.10-compute.sif \
  oras://ghcr.io/mit-orcd/rocky-8.10-compute:main_latest

# Import into Warewulf (inside ww001 container)
wwctl container import docker://ghcr.io/mit-orcd/rocky-8.10-compute:main_latest \
  rocky-8.10-compute

# Reboot compute nodes to pick up new image
wwctl node power cycle node4100
```

## Summary Table

| Container Type | Host Systems | Launch Method | Management | Persistence | Monitoring |
|---------------|--------------|---------------|------------|-------------|------------|
| **OOD** | core015-018 | `systemd` → `apptainer instance start --boot` | Individual systemd services | Overlay + host dirs | `systemctl status`, `apptainer exec`, httpd logs |
| **Login** | core015-018 | `systemd` → `apptainer instance start --boot` | Individual systemd services | Overlay + host dirs | `systemctl status`, `who`, `/var/log/secure` |
| **HA Proxy** | core016, 018 | `systemd` → `apptainer instance start --boot` | Individual systemd services | Overlay + host dirs | `systemctl status`, HAProxy stats |
| **Warewulf** | core015 | `systemd` → `apptainer instance start --boot` | Systemd service on host | Large overlay (100GB) | `wwctl` commands, warewulfd logs |
| **Compute Nodes** | node4100, node3401, etc. | PXE boot → Container in RAM | Warewulf centralized | Stateless (RAM only) | `sinfo`, `scontrol`, SSH direct access |

## Key Differences Between Container Types

### OOD/Login (Apptainer Instance Model)
- **Purpose**: Long-running services (web portal, SSH gateway)
- **Lifecycle**: Start once, run continuously
- **Isolation**: Network namespace with macvlan
- **Users**: Multiple concurrent users per container
- **State**: Overlay for container changes, bind mounts for data
- **Updates**: Stop → pull new image → restart
- **Failure**: Systemd auto-restart

### Compute Nodes (Network Boot Model)
- **Purpose**: Job execution on physical hardware
- **Lifecycle**: Boot from network on every restart
- **Isolation**: None (container IS the OS)
- **Users**: Slurm job execution (many users, time-shared)
- **State**: Fully stateless (RAM only)
- **Updates**: Rebuild image, nodes reboot to get new version
- **Failure**: Reboot restores to known good state

## Additional Resources

- **OOD Details**: See `OOD_CONTAINER_ORGANIZATION_AND_DEPLOYMENT.md`
- **OOD Fakeroot**: See `OOD_FAKEROOT_CONTAINER_LAUNCH.md`
- **Compute Conversion**: See `COMPUTE_NODE_APPTAINER_CONVERSION.md`
- **Inventory Docs**: `orcd-quantori-warewulf-work/inventory/INVENTORY_DOCUMENTATION.md`
- **Platform Build**: `orcd-quantori-platform-work/ansible/` playbooks

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-16  
**Maintainer**: Cluster Operations Team

