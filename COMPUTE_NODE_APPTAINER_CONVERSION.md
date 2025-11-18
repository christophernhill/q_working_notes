# Converting Compute Node Containers for Apptainer Launch from Running Host

## Overview

This document analyzes the modifications required to adapt the current Warewulf network-boot compute node containers to run as Apptainer instances launched from a running host operating system. This represents a significant architectural shift from stateless diskless nodes to persistent container-based compute nodes.

## Current Architecture vs. Proposed Architecture

### Current: Network Boot (Stateless)

```
Physical Hardware
    ↓ PXE Boot
Kernel + Initramfs (RAM)
    ↓ HTTP Download
Complete OS Container → RAM tmpfs
    ↓ Dracut modules
System Overlays + Local Disk Setup
    ↓ pivot_root
Container becomes root filesystem
    ↓
Systemd Init → Services
```

**Characteristics:**
- No persistent host OS
- Entire OS in RAM (8-32GB+)
- Boots from network every time
- Stateless by design
- Hardware has direct access to kernel

### Proposed: Apptainer Instance (Persistent)

```
Physical Hardware
    ↓ Normal Boot
Host OS (Rocky/Ubuntu) on Local Disk
    ↓ Systemd Service
Apptainer Instance Start
    ↓ Container Launch
Compute Container runs isolated
    ↓ Nested namespace
Services run inside container
```

**Characteristics:**
- Persistent host OS on disk
- Container runs as isolated process
- Container filesystem can be in RAM or on disk
- Services run in container namespace
- Hardware access mediated by Apptainer

## Architectural Challenges

### 1. Init System Conflict

**Problem:**
- Current container expects to run systemd as PID 1
- Apptainer containers cannot run systemd as init by default
- Services are managed by systemd inside the container

**Current Approach:**
```yaml
# nodes.conf
init: /sbin/init  # Systemd becomes PID 1
```

**Solutions:**

#### Option A: Boot Flag (Preferred)
Use Apptainer's `--boot` flag to run systemd inside the container:

```bash
apptainer instance start --boot \
  --writable-tmpfs \
  container.sif compute-node
```

This requires:
- Apptainer compiled with systemd support
- Container must have systemd
- Host must allow user namespaces with systemd

#### Option B: Supervisor Script
Replace systemd with a custom supervisor that starts services:

```bash
#!/bin/bash
# /sbin/container-init

# Start services in order
/usr/sbin/chronyd
/usr/sbin/sshd
/usr/sbin/munged
/usr/sbin/slurmd

# Keep running
while true; do sleep 3600; done
```

Set as `%runscript` in container definition:
```singularity
%runscript
    exec /sbin/container-init
```

#### Option C: Hybrid Approach
Run a minimal init inside container:
- Use `dumb-init` or `tini` as PID 1
- Launch services via shell scripts
- Monitor and restart services

### 2. Network Configuration

**Problem:**
- Current setup uses kernel network configuration from iPXE/dracut
- Multiple network interfaces (Ethernet, InfiniBand)
- Warewulf applies network overlays dynamically

**Current Approach:**
```yaml
network_devices:
  internal:
    type: ethernet
    device: boot0
    ipaddr: 10.1.34.1
  ib:
    type: infiniband
    device: ib0
    ipaddr: 172.16.34.1
```

These are configured by:
1. Kernel command line
2. Dracut network modules
3. NetworkManager with overlays

**Solutions:**

#### Option A: Host Network Mode (Simple)
Use host's network stack directly:

```bash
apptainer instance start --boot \
  --network none \  # Disable container networking
  container.sif compute-node
```

The container sees all host interfaces as-is.

**Pros:**
- Simple, no configuration needed
- Direct hardware access
- Works with InfiniBand

**Cons:**
- Less isolation
- Container can affect host network
- Hostname conflicts possible

#### Option B: Bridge/Macvlan (Isolated)
Give container its own IP like OOD deployment:

```bash
apptainer instance start --boot \
  --network bridge \
  --network-args "portmap=22:2222/tcp" \
  container.sif compute-node
```

Or use macvlan like Warewulf does:

```bash
apptainer instance start --boot \
  --network macvlan_compute001 \
  container.sif compute-node
```

**Pros:**
- Network isolation
- Container has own IP
- Similar to Warewulf model

**Cons:**
- Requires network configuration
- InfiniBand passthrough more complex
- DHCP complications

#### Option C: Hybrid Network
Use host network but configure inside container:

```bash
apptainer instance start --boot \
  --network none \
  -B /sys/class/net \
  -B /etc/sysconfig/network-scripts \
  container.sif compute-node
```

Container configures interfaces using host's network subsystem.

**Recommended:** Option A (host network) for simplicity and InfiniBand compatibility.

### 3. Hardware Access

**Problem:**
- Current setup has direct kernel access to hardware
- GPUs, InfiniBand, RAID controllers need direct access
- Apptainer mediates hardware access

**Required Hardware Access:**

#### GPU (NVIDIA)
```bash
apptainer instance start --boot \
  --nv \  # NVIDIA GPU support
  container.sif compute-node
```

Requires:
- NVIDIA driver on host matches container
- Or use `--nvccli` for automatic injection

#### InfiniBand
```bash
apptainer instance start --boot \
  -B /dev/infiniband \
  -B /sys/class/infiniband \
  container.sif compute-node
```

Requires:
- RDMA subsystem on host
- MOFED drivers on host
- Container can use host's IB stack

#### RAID Controllers
```bash
apptainer instance start --boot \
  --device /dev/md0 \
  --device /dev/nvme0n1 \
  --device /dev/nvme1n1 \
  -B /dev/disk \
  container.sif compute-node
```

Requires:
- Block devices passed through
- udev rules available
- mdadm on host or in container

### 4. Filesystem and Storage

**Problem:**
- Current setup uses tmpfs for root, local RAID for scratch
- MkDisk configures storage at boot time
- Apptainer needs different storage strategy

**Current Storage Layout:**
```
/ (tmpfs, 16GB) - OS root filesystem
/system (md0, RAID1) - Persistent storage
/scratch (md1, RAID0) - High-performance scratch
/shared (NFS) - Cluster shared storage
/home (NFS) - User home directories
```

**Solutions:**

#### Option A: Overlay + Bind Mounts
```bash
apptainer instance start --boot \
  --overlay /local/compute-overlay.img \  # Writable layer
  -B /raid/system:/system \               # Local RAID
  -B /raid/scratch:/scratch \             # Scratch space
  -B /shared:/shared \                    # NFS shares
  -B /home:/home \                        # Home dirs
  container.sif compute-node
```

**Storage Preparation Script:**
```bash
#!/bin/bash
# setup-compute-storage.sh

# Create overlay image (one-time)
if [ ! -f /local/compute-overlay.img ]; then
  apptainer overlay create --size 10240 /local/compute-overlay.img
fi

# Configure RAID arrays
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/nvme0n1 /dev/nvme1n1
mdadm --create /dev/md1 --level=0 --raid-devices=4 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1 /dev/nvme5n1

# Format and mount
mkfs.ext4 /dev/md0
mkfs.ext4 /dev/md1
mkdir -p /raid/system /raid/scratch
mount /dev/md0 /raid/system
mount /dev/md1 /raid/scratch

# Setup directories
mkdir -p /raid/system/{log,tmp}
mkdir -p /raid/scratch/fscache
```

#### Option B: Bind Mount Root
Use container as template, bind mount persistent root:

```bash
apptainer instance start --boot \
  -B /persistent/compute-root:/var \
  -B /persistent/compute-root:/tmp \
  -B /raid/scratch:/scratch \
  container.sif compute-node
```

Provides persistence for logs and temporary files.

#### Option C: Hybrid Approach
Container read-only with specific writable paths:

```bash
apptainer instance start --boot \
  --writable-tmpfs \                # In-memory writes
  -B /raid/system/log:/var/log \    # Persistent logs
  -B /raid/system/tmp:/tmp \        # Persistent temp
  -B /raid/scratch:/scratch \       # Scratch space
  container.sif compute-node
```

**Recommended:** Option C for consistency with current stateless model.

### 5. Service Dependencies and Ordering

**Problem:**
- Current systemd units have dependencies on hardware and network
- Some services need to start after storage is ready
- Container init must respect dependencies

**Current Dependencies:**
```ini
[Unit]
After=network.target
After=time-sync.target
Requires=munge.service

[Service]
Type=notify
ExecStart=/usr/sbin/slurmd
```

**Solutions:**

#### With --boot Flag
Services managed by systemd inside container work normally:

```bash
systemctl status slurmd
systemctl restart munge
```

Dependencies resolved by container's systemd.

#### Without --boot Flag
Create a service manager script:

```bash
#!/bin/bash
# /usr/local/bin/service-manager.sh

# Wait for network
while ! ping -c1 -W1 10.1.224.50 &>/dev/null; do
  sleep 1
done

# Start chronyd for time sync
/usr/sbin/chronyd -d &
sleep 2

# Start munge
/usr/sbin/munged --foreground &
sleep 2

# Start slurmd
/usr/sbin/slurmd -D &

# Monitor processes
while true; do
  # Check if services are running
  pgrep slurmd || /usr/sbin/slurmd -D &
  sleep 60
done
```

**Recommended:** Use `--boot` flag for proper systemd support.

### 6. Host OS Requirements

**Problem:**
- Need a base OS on the physical host
- Must be compatible with container
- Need to manage host OS separately

**Host OS Setup:**

#### Minimal Host OS Requirements
```yaml
# Host packages needed
packages:
  - apptainer-suid
  - kernel (matching container kernel modules)
  - network tools
  - storage tools (mdadm, lvm)
  - monitoring agents

services:
  - sshd (for remote access)
  - chronyd (time sync)
  - networking (basic)
  - monitoring
```

#### Host OS Build
Create a minimal host OS playbook:

```yaml
# host-os-minimal.yml
- name: Setup compute host OS
  hosts: localhost
  tasks:
    - name: Install minimal packages
      package:
        name:
          - apptainer-suid
          - openssh-server
          - chrony
          - mdadm
          - nfs-utils
          - kernel
        state: present
    
    - name: Disable unnecessary services
      systemd:
        name: "{{ item }}"
        enabled: no
      loop:
        - firewalld
        - postfix
        - bluetooth
    
    - name: Enable required services
      systemd:
        name: "{{ item }}"
        enabled: yes
      loop:
        - sshd
        - chronyd
    
    - name: Install compute container launcher
      copy:
        src: launch-compute-container.sh
        dest: /usr/local/bin/
        mode: '0755'
    
    - name: Install systemd service for compute container
      copy:
        dest: /etc/systemd/system/compute-node.service
        content: |
          [Unit]
          Description=Compute Node Container
          After=network.target
          
          [Service]
          Type=forking
          ExecStart=/usr/local/bin/launch-compute-container.sh start
          ExecStop=/usr/local/bin/launch-compute-container.sh stop
          
          [Install]
          WantedBy=multi-user.target
    
    - name: Enable compute node service
      systemd:
        name: compute-node
        enabled: yes
        daemon_reload: yes
```

### 7. Container Build Modifications

**Problem:**
- Current container assumes it IS the root filesystem
- Need modifications for running inside Apptainer instance

**Required Changes:**

#### 1. Remove Conflicting Components

Create a new role `setup_apptainer_env`:

```yaml
# roles/setup_apptainer_env/tasks/main.yml
- name: Remove components handled by host
  package:
    name:
      - systemd-udev    # Keep this one for hardware access
    state: present

- name: Keep network manager (might be needed)
  package:
    name: NetworkManager
    state: present

- name: Ensure systemd is properly configured
  copy:
    dest: /etc/systemd/system.conf
    content: |
      [Manager]
      RuntimeWatchdogSec=0
      ShutdownWatchdogSec=10min
```

#### 2. Modify Dracut Configuration

Since we're not using network boot, simplify dracut:

```yaml
# Don't build network boot initramfs
# Keep kernel for debugging but don't use it

- name: Build minimal initramfs
  command:
    cmd: "dracut --force --kver {{ kernel_version }}"
  tags: build
```

Or skip dracut entirely:

```yaml
# Replace dracut role with nothing
# roles/dracut becomes empty or removed
```

#### 3. Add Container Entry Point

Create `%runscript` or modify init:

```singularity
%runscript
    #!/bin/bash
    # Container entry point for Apptainer
    
    # Source environment
    source /etc/profile.d/modules.sh
    
    # Start systemd if running with --boot
    if [ -f /run/systemd/system ]; then
        exec /lib/systemd/systemd
    fi
    
    # Otherwise run service manager
    exec /usr/local/bin/service-manager.sh
```

#### 4. Create Apptainer-Specific Playbook

```yaml
# rocky-8.10-compute-apptainer.yml
- name: Build compute node for Apptainer launch
  hosts: localhost
  become: yes
  vars:
    bootstrap: "docker"
    build_from: "docker.io/rockylinux/rockylinux:8.10"
    image_name: "rocky-8.10-compute-apptainer"
    distro: "rocky8"
    kernel_version: "4.18.0-553.56.1.el8_10.x86_64"
    nvidia_version: "580.65.06"
    mofed_version: "23.10-5.1.4.0"
    slurm_version: "24.11.6-1"
  
  roles:
    - common/bootstrap
    - create_repos
    - create_users
    - packages
    - munge
    - sssd
    - nhc
    - mofed
    - nvidia
    - chrony
    - slurmd
    - setup_timezone
    - common/setup_home
    - common/setup_lmod
    - setup_apptainer_env  # New role
    - common/final_cleanup
    - common/build_info
    - cleanup
  
  # Note: Removed roles
  # - mkdisk (handled by host)
  # - dracut (not needed)
  # - xiraid (handled by host if needed)
  # - ignition (not needed)
  # - setup_container_env (replaced)
```

## Complete Deployment Solution

### Step 1: Build Modified Container

```bash
cd orcd-quantori-platform-work/ansible

# Build Apptainer-compatible compute container
ansible-playbook rocky-8.10-compute-apptainer.yml

# Push to registry
apptainer push rocky-8.10-compute-apptainer.sif \
  oras://ghcr.io/mit-orcd/rocky-8.10-compute-apptainer:latest
```

### Step 2: Setup Host OS

Install minimal OS on physical hardware:

```bash
# Install Rocky 8.10 on bare metal
# Minimal installation profile

# Run host setup
ansible-playbook -i compute-hosts.yml host-os-minimal.yml
```

### Step 3: Deploy Container Launcher

Create `/usr/local/bin/launch-compute-container.sh`:

```bash
#!/bin/bash
# Launch compute node as Apptainer instance

CONTAINER_URL="oras://ghcr.io/mit-orcd/rocky-8.10-compute-apptainer:latest"
CONTAINER_PATH="/local/containers/compute.sif"
OVERLAY_PATH="/local/overlay/compute.img"
INSTANCE_NAME="compute-node"

setup_storage() {
    # Setup RAID arrays
    if [ ! -b /dev/md0 ]; then
        mdadm --create /dev/md0 --level=1 --raid-devices=2 \
          /dev/nvme0n1 /dev/nvme1n1
        mkfs.ext4 /dev/md0
    fi
    
    if [ ! -b /dev/md1 ]; then
        mdadm --create /dev/md1 --level=0 --raid-devices=4 \
          /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1 /dev/nvme5n1
        mkfs.ext4 /dev/md1
    fi
    
    # Mount arrays
    mkdir -p /raid/system /raid/scratch
    mount /dev/md0 /raid/system
    mount /dev/md1 /raid/scratch
    
    # Create directories
    mkdir -p /raid/system/{log,tmp}
    mkdir -p /raid/scratch/fscache
    
    # Create overlay
    if [ ! -f "$OVERLAY_PATH" ]; then
        mkdir -p $(dirname "$OVERLAY_PATH")
        apptainer overlay create --size 10240 "$OVERLAY_PATH"
    fi
}

pull_container() {
    if [ ! -f "$CONTAINER_PATH" ]; then
        mkdir -p $(dirname "$CONTAINER_PATH")
        apptainer pull "$CONTAINER_PATH" "$CONTAINER_URL"
    fi
}

start_container() {
    setup_storage
    pull_container
    
    apptainer instance start \
        --boot \
        --nv \
        --writable-tmpfs \
        --overlay "$OVERLAY_PATH" \
        --network none \
        -B /etc/slurm \
        -B /etc/munge \
        -B /var/run/munge \
        -B /var/lib/sss \
        -B /raid/system/log:/var/log \
        -B /raid/system/tmp:/tmp \
        -B /raid/scratch:/scratch \
        -B /shared:/shared \
        -B /home:/home \
        -B /orcd:/orcd \
        -B /dev/infiniband \
        -B /sys/class/infiniband \
        "$CONTAINER_PATH" \
        "$INSTANCE_NAME"
}

stop_container() {
    apptainer instance stop "$INSTANCE_NAME"
}

case "$1" in
    start)
        start_container
        ;;
    stop)
        stop_container
        ;;
    restart)
        stop_container
        sleep 2
        start_container
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```

### Step 4: Enable and Start Service

```bash
# Enable service
systemctl enable compute-node.service

# Start compute node
systemctl start compute-node.service

# Check status
systemctl status compute-node.service

# View container logs
journalctl -u compute-node.service -f

# Access container
apptainer shell instance://compute-node

# Run commands in container
apptainer exec instance://compute-node squeue
```

## Configuration Management

### Ansible Integration

Update inventory structure:

```yaml
# inventory/prod/host_vars/compute001.yml
ansible_host: compute001.inband
ansible_user: root

# Host OS configuration
host_os: rocky8
host_packages:
  - apptainer-suid
  - mdadm
  - kernel

# Container configuration
container_url: oras://ghcr.io/mit-orcd/rocky-8.10-compute-apptainer:latest
container_overlay_size: 10240

# Storage configuration
raid_config:
  - level: 1
    devices: [/dev/nvme0n1, /dev/nvme1n1]
    mount: /raid/system
  - level: 0
    devices: [/dev/nvme2n1, /dev/nvme3n1, /dev/nvme4n1, /dev/nvme5n1]
    mount: /raid/scratch

# Network configuration (on host)
network:
  eth0:
    ipaddr: 10.1.34.1
    netmask: 255.255.0.0
    gateway: 10.1.224.51
  ib0:
    ipaddr: 172.16.34.1
    netmask: 255.255.0.0
```

### Deployment Playbook

```yaml
# deploy-apptainer-compute.yml
- name: Deploy Apptainer-based compute nodes
  hosts: compute
  tasks:
    - name: Update host OS
      package:
        name: "*"
        state: latest
    
    - name: Deploy launcher script
      template:
        src: launch-compute-container.sh.j2
        dest: /usr/local/bin/launch-compute-container.sh
        mode: '0755'
    
    - name: Deploy systemd service
      template:
        src: compute-node.service.j2
        dest: /etc/systemd/system/compute-node.service
    
    - name: Enable service
      systemd:
        name: compute-node
        enabled: yes
        daemon_reload: yes
    
    - name: Pull new container if updated
      command: apptainer pull --force {{ container_path }} {{ container_url }}
      when: container_updated | default(false)
    
    - name: Restart compute node
      systemd:
        name: compute-node
        state: restarted
      when: container_updated | default(false)
```

## Comparison: Network Boot vs. Apptainer Launch

| Aspect | Network Boot (Current) | Apptainer Launch (Proposed) |
|--------|------------------------|----------------------------|
| **Boot Source** | Network (PXE/HTTP) | Local disk |
| **Boot Time** | ~2-3 minutes | ~30-60 seconds |
| **Network Dependency** | Required for boot | Not required |
| **Root Filesystem** | Entire OS in RAM | Container in RAM or overlay |
| **Persistence** | Fully stateless | Host OS persistent, container ephemeral |
| **Updates** | Reboot downloads new image | Pull + restart container |
| **Hardware Access** | Direct kernel access | Mediated by Apptainer |
| **Storage Config** | mkdisk at boot | Host script or container init |
| **Networking** | Kernel + overlays | Host or container |
| **Complexity** | High (Warewulf, dracut, etc.) | Medium (Apptainer + systemd) |
| **Failure Recovery** | Reboot restores to known state | Restart container |
| **Management** | Centralized (Warewulf) | Distributed (Ansible) |
| **Scalability** | Excellent (1000s of nodes) | Good (100s of nodes) |
| **Infrastructure** | Requires Warewulf server | No special infrastructure |

## Advantages of Apptainer Approach

1. **Faster Boot** - No network download of rootfs
2. **Offline Capability** - Nodes can boot without network
3. **Simpler Infrastructure** - No Warewulf server required
4. **Easier Debugging** - Can access host OS and container
5. **Flexible Storage** - More options for persistent data
6. **Gradual Rollout** - Update nodes individually
7. **Host OS Access** - Can SSH to host and container

## Disadvantages of Apptainer Approach

1. **Host OS Maintenance** - Must manage base OS on each node
2. **Less Stateless** - Host OS can diverge over time
3. **Storage Overhead** - Need disk space for host OS + container
4. **Complexity Split** - Configuration across host and container
5. **Lost Warewulf Features** - No centralized overlay management
6. **Network Complexity** - More complex than Warewulf overlays
7. **Scale Limits** - Harder to manage 1000+ nodes

## Hybrid Approach (Best of Both)

Consider a hybrid approach:

**Option 1: Warewulf Boot + Apptainer Runtime**
- Boot via Warewulf to RAM (stateless host)
- Launch compute workloads in Apptainer containers
- Separate infrastructure from applications

**Option 2: Disk Boot + Warewulf Management**
- Install minimal OS on disk
- Use Warewulf for container distribution only
- Apply overlays at container startup

**Option 3: Two-Tier Architecture**
- Critical nodes: Network boot (highly available)
- Development nodes: Apptainer launch (more flexible)

## Recommendations

### When to Use Network Boot (Current Architecture)

✅ Large-scale production clusters (100+ nodes)
✅ Need absolute consistency across nodes
✅ Stateless operation is critical
✅ Rapid recovery from failures
✅ Centralized management preferred
✅ Nodes are homogeneous

### When to Use Apptainer Launch

✅ Smaller clusters (<100 nodes)
✅ Heterogeneous hardware
✅ Need persistent host OS for debugging
✅ Network boot infrastructure not available
✅ Frequent container updates without reboots
✅ Mixed workload environments (some containers, some bare metal)

### Conclusion

The current Warewulf network boot architecture is well-suited for large-scale, homogeneous compute clusters where stateless operation and centralized management are priorities. Converting to Apptainer-launched containers would simplify some aspects (no Warewulf server, faster boots) but add complexity in others (host OS management, storage configuration).

For most production HPC environments, **the network boot approach is superior**. However, the Apptainer approach could be valuable for:
- Development and testing environments
- Small satellite clusters
- Heterogeneous or multi-tenant systems
- Environments where network boot infrastructure is unavailable

If pursuing the Apptainer approach, focus on:
1. Robust host OS provisioning and management
2. Comprehensive container launch automation
3. Monitoring of both host and container
4. Clear separation of host vs. container responsibilities
5. Documentation of troubleshooting procedures

The good news is that the container images can be shared between both approaches with minimal modifications, allowing a hybrid deployment strategy based on specific needs.






