# Compute Node Container Build and Network Boot Process

## Overview

This document describes how compute node container images are built and deployed using Warewulf 4 for network booting (diskless operation). The system uses stateless compute nodes that boot entirely from network-provided images loaded into RAM, with local disks configured dynamically for scratch storage.

## Architecture Summary

The compute node deployment architecture consists of:

1. **Container Image Build** - Apptainer/Singularity containers built from Linux distributions
2. **Warewulf Server** - Runs as an Apptainer container providing provisioning services
3. **Network Boot** - PXE/iPXE boot process loads kernel and initramfs
4. **Image Deployment** - Container rootfs loaded into RAM via HTTP
5. **Runtime Configuration** - Overlays applied for node-specific configuration
6. **Local Storage** - RAID arrays configured via mkdisk during boot

## Phase 1: Container Image Build

### Build Environment

Compute node containers are built using the `orcd-quantori-platform-work` repository. The build process creates Apptainer/Singularity container images (.sif files) that contain a complete Linux operating system with all required software.

### Build Playbooks

Multiple compute node variants are available:

- **Rocky Linux 8.10 Compute** (`rocky-8.10-compute.yml`)
- **Rocky Linux 9.5 Compute** (`rocky-9.5-compute.yml`)
- **Ubuntu 22.04 Compute** (`ubuntu-22.04-compute.yml`)
- **Ubuntu 24.04 Compute** (`ubuntu-24.04-compute.yml`)

Each playbook specifies:
```yaml
vars:
  bootstrap: "docker"
  build_from: "docker.io/rockylinux/rockylinux:8.10"
  image_name: "rocky-8.10-compute"
  distro: "rocky8"
  kernel_version: "4.18.0-553.56.1.el8_10.x86_64"
  nvidia_version: "580.65.06"
  mofed_version: "23.10-5.1.4.0"
  slurm_version: "24.11.6-1"
```

### Bootstrap Process

The build uses a two-stage bootstrap defined in `roles/common/bootstrap/`:

#### Stage 1: Container Definition

A Singularity definition file is generated (`container.def.j2`):

```singularity
Bootstrap: docker
From: docker.io/rockylinux/rockylinux:8.10

%files
    .. /container_repo

%setup
    mkdir ${SINGULARITY_ROOTFS}/shared
    touch ${SINGULARITY_ROOTFS}/bootstrap.sh

%post
    /bin/bash /bootstrap.sh
    
    cd /container_repo/ansible
    source /opt/ansible/bin/activate
    ansible-playbook -v rocky-8.10-compute.yml -i localhost, -c local -l localhost --tags "build"
```

#### Stage 2: Bootstrap Script

The `bootstrap.sh` script prepares the container for Ansible execution:

1. **Install Python** - Required for Ansible
2. **Install Language Packs** - Locale support
3. **Create Virtual Environment** - `/opt/ansible`
4. **Install Ansible** - Latest version with ansible-runner

The bootstrap ensures Ansible can execute even in minimal base images.

#### Stage 3: Ansible Playbook Execution

Inside the container, the full playbook runs with the `build` tag, applying roles sequentially.

### Roles Applied During Build

The compute node build applies these roles (from `rocky-8.10-compute.yml`):

```yaml
roles:
  - common/bootstrap          # Initial setup
  - create_repos              # Package repository configuration
  - create_users              # System users/groups
  - packages                  # Base packages and utilities
  - mkdisk                    # Local disk configuration tool
  - dracut                    # Initramfs generation
  - munge                     # Authentication for Slurm
  - sssd                      # Identity management
  - nhc                       # Node Health Check
  - mofed                     # Mellanox OFED drivers
  - nvidia                    # NVIDIA GPU drivers
  - chrony                    # Time synchronization
  - xiraid                    # Xi RAID controller support
  - slurmd                    # Slurm compute daemon
  - setup_timezone            # Timezone configuration
  - common/enable_autofs      # Automount configuration
  - common/enable_fscache     # Filesystem caching
  - common/unmask_services    # Systemd service management
  - common/setup_home         # Home directory setup
  - common/setup_lmod         # Environment modules (Lmod)
  - common/disable_selinux    # SELinux configuration
  - common/final_cleanup      # Image optimization
  - common/build_info         # Build metadata
  - cleanup                   # Final cleanup and optimization
```

### Key Build Configurations

#### 1. Dracut and Initramfs Generation

The `dracut` role is critical for network boot functionality:

**Package Installation:**
```yaml
- name: install warewulf dracut package
  ansible.builtin.package:
    name: "warewulf-dracut-4.6.1"
    state: present
```

**Custom Dracut Modules:**

The build includes custom dracut modules in `/usr/lib/dracut/modules.d/`:

- **91preboot** - Executes Ansible playbooks during early boot
- **92mkdisk** - Configures local disk arrays during boot

**Initramfs Generation:**
```bash
dracut --force --no-hostonly --add wwinit --add preboot --add mkdisk --kver 4.18.0-553.56.1.el8_10.x86_64
```

This creates a generic (non-hostonly) initramfs that includes:
- `wwinit` - Warewulf initialization module
- `preboot` - Custom pre-boot configuration
- `mkdisk` - Disk configuration module

The resulting initramfs is copied to `/boot/initrd-${kernel_version}`.

#### 2. Container Environment Setup

The `setup_container_env` role prepares the container for stateless operation:

**For Rocky 8:**
```yaml
- name: disable NetworkManager
  ansible.builtin.command: systemctl disable NetworkManager

- name: remove autofs
  ansible.builtin.package:
    name: "autofs"
    state: absent

- name: remove udev
  shell: rpm -e systemd-udev --nodeps

- name: remove rpcbind
  ansible.builtin.package:
    name: "rpcbind"
    state: absent
```

These changes are necessary because:
- **NetworkManager** - Conflicts with Warewulf network management
- **autofs** - Handled differently in stateless nodes
- **udev** - Not needed when running from RAM
- **rpcbind** - Provided by Warewulf at runtime

#### 3. Hardware Drivers

**NVIDIA GPU Support:**
- Driver version specified in playbook (e.g., 580.65.06)
- Kernel modules compiled for specified kernel version
- CUDA libraries and tools included

**Mellanox InfiniBand (MOFED):**
- OFED version specified (e.g., 23.10-5.1.4.0)
- Kernel modules for IB networking
- User-space libraries and tools

**Xi RAID Controllers:**
- Proprietary RAID management utilities
- Monitoring and configuration tools

### Build Output

The build produces:
- **Container Image** - `rocky-8.10-compute.sif` (Apptainer SIF format)
- **Kernel** - `/boot/vmlinuz-${kernel_version}`
- **Initramfs** - `/boot/initrd-${kernel_version}`

### Image Storage and Distribution

Built images are:
1. **Pushed to GitHub Container Registry** (GHCR):
   ```
   oras://ghcr.io/mit-orcd/rocky-8.10-compute:main_latest
   ```

2. **Tagged with versions**:
   - `:main_latest` - Latest build from main branch
   - `:main_ab64` - Specific commit reference
   - Version tags for releases

## Phase 2: Warewulf Server Deployment

### Warewulf Container Build

Warewulf itself runs as a containerized service. The build is defined in `warewulf-4.5.8.yml`:

```yaml
vars:
  bootstrap: "docker"
  build_from: "docker.io/rockylinux/rockylinux:8.10"
  image_name: "warewulf-4.5.8"
  warewulf_version: "4.5.8"

roles:
  - common/bootstrap
  - warewulf
  - common/build_info
```

The Warewulf container includes:
- Warewulf 4.5.8 or 4.6.1 server components
- DHCP server
- TFTP server
- NFS server
- HTTP server for image provisioning

### Warewulf Deployment on Physical Host

The Warewulf server is deployed using `warewulf.yml` playbook from `orcd-quantori-warewulf-work`:

**Host Configuration Example** (`inventory/prod/host_vars/ww001.yml`):
```yaml
ansible_host: ww001
container_url: oras://ghcr.io/mit-orcd/warewulf-4.6.1:main_latest
servicename: ww001

apptainer_options:
  overlay_size: 100000  # Large overlay for container storage
  network: 
    - macvlan_ww001
    - ipmi_bridge_ww001  # For IPMI management
  mounts:
    - /etc/slurm
    - /shared
    - /home
    - /orcd
    - $DIRNAME/root:/root
  boot: true

import_containers:
  - name: rocky-8.10-compute
    url: ghcr.io/mit-orcd/rocky-8.10-compute:main_latest
  - name: rocky-9.5-compute
    url: ghcr.io/mit-orcd/rocky-9.5-compute:main_latest
  - name: ubuntu-22.04-compute
    url: ghcr.io/mit-orcd/ubuntu-22.04-compute:main_latest
```

### Warewulf Container Import Process

The `warewulf` role imports compute node containers:

```bash
# Pull the SIF image
apptainer pull /etc/warewulf/containers/${name}.sif oras://${url}

# Extract the rootfs from the SIF
apptainer sif dump 4 /etc/warewulf/containers/${name}.sif > /etc/warewulf/containers/${name}.img

# Unsquash to directory
unsquashfs -d /etc/warewulf/containers/${name} /etc/warewulf/containers/${name}.img

# Import into Warewulf
wwctl container import --force --build /etc/warewulf/containers/${name}
```

This process:
1. Downloads the Apptainer SIF file
2. Extracts the SquashFS filesystem (partition 4 of SIF)
3. Unsquashes to a directory
4. Imports into Warewulf's management system
5. Warewulf builds its own provisioning formats

After import, containers are stored in:
- `/var/lib/warewulf/chroots/${name}/` - Extracted rootfs
- `/var/lib/warewulf/provision/containers/${name}.img.gz` - Compressed image for network boot

### Warewulf Configuration

#### Server Configuration (`warewulf.conf`)

Generated from `warewulf.conf.j2`:

```yaml
ipaddr: 10.1.224.50
netmask: 255.255.0.0
network: 10.1.0.0

warewulf:
  port: 9873
  secure: false
  update interval: 60
  autobuild overlays: false
  host overlay: true

nfs:
  enabled: true
  export paths:
    - path: /shared
      export options: rw,sync,no_root_squash
    - path: /warewulf/secrets
      export options: ro,sync,no_root_squash

dhcp:
  enabled: true
  template: static

tftp:
  enabled: true
  tftproot: /var/lib/tftpboot
```

#### Node Profiles (`nodes.conf`)

The default profile applied to all nodes:

```yaml
nodeprofiles:
  default:
    comment: This profile is automatically included for each node
    cluster name: mit-hpc
    ipxe template: default
    
    runtime overlay:
      - hosts
      - ssh.authorized_keys
    
    system overlay:
      - wwinit
      - wwclient
      - fstab
      - hostname
      - ssh.host_keys
      - issue
      - resolv
      - udev.netname
      - ignition
      - NetworkManager
      - mkdisk
    
    kernel:
      args:
        - console=tty0
        - crashkernel=no
        - modprobe.blacklist=nouveau
        - net.ifnames=0
        - numa_balancing=disable
        - quiet
        - transparent_hugepage=never
        - vga=791
        - systemd.unified_cgroup_hierarchy=0
    
    init: /sbin/init
    root: initramfs
```

**Overlay Types:**

- **System Overlays** - Built into the provisioned image, applied before boot
- **Runtime Overlays** - Applied dynamically at boot time from Warewulf server

#### Individual Node Configuration

Each compute node has a configuration file generated from `node_config.j2`:

Example (`node3401.yml`):
```yaml
node3401:
  cluster name: mit-hpc
  image name: rocky-8.10-compute
  
  ipmi:
    ipaddr: 172.29.34.1
  
  profiles:
    - default
  
  system overlay:
    - hostname
  
  runtime overlay:
    - hosts
  
  network devices:
    internal:
      type: ethernet
      onboot: true
      device: boot0
      hwaddr: 6C:92:CF:EE:90:80
      ipaddr: 10.1.34.1
      netmask: 255.255.0.0
      gateway: 10.1.224.51
      mtu: 1500
      primary: true
    
    ib:
      type: infiniband
      device: ib5
      ipaddr: 172.16.34.1
      netmask: 255.255.0.0
      primary: false
  
  tags:
    mkdisk_profile: default
    mkdisk_fscache: /scratch/fscache
    mkdisk_movedirs: /var/log:/system/log,/tmp:/system/tmp
    mkdisk_wipeonboot: "true"
  
  primary network: internal
```

Nodes are imported into Warewulf:
```bash
wwctl node import /etc/warewulf/nodes/node3401.yml
```

### Warewulf Overlays

Overlays are filesystem additions that customize the base container image. They are stored in `/var/lib/warewulf/overlays/` and synchronized from the repository.

**Key Overlays:**

1. **wwinit** - Warewulf initialization scripts
2. **hostname** - Node-specific hostname configuration
3. **hosts** - `/etc/hosts` file
4. **fstab** - Filesystem mount table
5. **mkdisk** - Local disk configuration
6. **ssh.host_keys** - SSH host keys (persistent per-node)
7. **ssh.authorized_keys** - User SSH keys
8. **NetworkManager** - Network configuration
9. **resolv** - DNS resolver configuration

Example overlay file (`overlays/host/rootfs/etc/profile.d/ssh_setup.sh.ww`):

This is a Warewulf template that generates SSH keys for users on first login:

```bash
#!/bin/sh
_UID=`id -u`
if [ $_UID -ge 500 -o $_UID -eq 0 ] && [ ! -f "$HOME/.ssh/config" -a ! -f "$HOME/.ssh/cluster" ]; then
    echo "Configuring SSH for cluster access"
    install -d -m 700 $HOME/.ssh
    ssh-keygen -t ed25519 -f $HOME/.ssh/cluster -N '' -C "Warewulf Cluster key"
    cat $HOME/.ssh/cluster.pub >> $HOME/.ssh/authorized_keys
    chmod 0600 $HOME/.ssh/authorized_keys
    
    echo "Host *" >> $HOME/.ssh/config
    echo "   IdentityFile ~/.ssh/cluster" >> $HOME/.ssh/config
    echo "   StrictHostKeyChecking=no" >> $HOME/.ssh/config
    chmod 0600 $HOME/.ssh/config
fi
```

This allows passwordless SSH between nodes in the cluster.

### Secrets Management

Warewulf manages secrets via a tmpfs mount:

```yaml
- name: Create /warewulf/secrets and /warewulf/secrets_local directories
  file:
    path: "{{ item }}"
    state: directory
    mode: '0700'
  loop:
    - /warewulf/secrets
    - /warewulf/secrets_local

- name: Create systemd mount unit
  copy:
    dest: /etc/systemd/system/warewulf-secrets.mount
    content: |
      [Mount]
      What=tmpfs
      Where=/warewulf/secrets
      Type=tmpfs
      Options=size=10M
```

The secrets are:
1. Stored encrypted on disk in `/warewulf/secrets_local/`
2. Copied to tmpfs at `/warewulf/secrets/` at boot
3. NFS-exported to compute nodes (read-only)
4. Available during the preboot phase for Ansible playbooks

## Phase 3: Network Boot Process

### Boot Sequence Overview

1. **Power On / IPMI Reset**
2. **PXE Boot** - NIC firmware initiates network boot
3. **DHCP Request** - Node requests IP and boot server
4. **iPXE Chainload** - TFTP provides iPXE bootloader
5. **Kernel Download** - HTTP download of kernel
6. **Initramfs Download** - HTTP download of initramfs
7. **Kernel Boot** - Linux kernel starts
8. **Initramfs Execution** - Dracut runs boot modules
9. **Container Download** - Warewulf provides rootfs
10. **Preboot Phase** - Ansible configuration
11. **MkDisk Phase** - Local disk setup
12. **Pivot Root** - Switch to container rootfs
13. **Init** - Systemd starts services

### Detailed Boot Process

#### Step 1-4: PXE and iPXE Boot

When the node powers on:

1. BIOS/UEFI initiates PXE boot
2. Node sends DHCP DISCOVER
3. Warewulf DHCP responds with:
   - IP address (e.g., 10.1.34.1)
   - Next-server (Warewulf IP)
   - Boot filename (iPXE binary)

4. Node downloads iPXE via TFTP
5. iPXE executes and requests boot configuration
6. Warewulf generates iPXE menu/script dynamically

#### Step 5-6: Kernel and Initramfs Download

iPXE script generated by Warewulf:

```ipxe
#!ipxe
kernel http://10.1.224.50:9873/provision/kernel/rocky-8.10-compute \
  console=tty0 crashkernel=no modprobe.blacklist=nouveau \
  net.ifnames=0 numa_balancing=disable quiet \
  transparent_hugepage=never vga=791 \
  systemd.unified_cgroup_hierarchy=0 \
  wwinit_uri=http://10.1.224.50:9873/provision/

initrd http://10.1.224.50:9873/provision/initrd/rocky-8.10-compute

boot
```

The kernel and initramfs are served via HTTP from Warewulf.

#### Step 7-8: Kernel Boot and Dracut Initialization

The Linux kernel boots and unpacks the initramfs into a RAM-based filesystem. Dracut starts and executes modules in order:

**Key Dracut Modules:**

1. **network** - Configure network interface (already up from iPXE)
2. **wwinit** - Warewulf initialization module
3. **preboot** - Custom pre-boot configuration (module 91)
4. **mkdisk** - Disk configuration (module 92)

#### Step 9: Container Download (wwinit module)

The `wwinit` module (provided by Warewulf) performs:

```bash
# Download compressed container image
curl http://10.1.224.50:9873/provision/container/rocky-8.10-compute \
  -o /tmp/container.img.gz

# Mount tmpfs for rootfs
mount -t tmpfs -o size=<calculated> tmpfs /sysroot

# Extract container to /sysroot
gunzip -c /tmp/container.img.gz | cpio -idmv -D /sysroot

# Apply system overlays
for overlay in wwinit wwclient fstab hostname ssh.host_keys issue resolv udev.netname ignition NetworkManager mkdisk; do
  curl http://10.1.224.50:9873/overlay/system/$overlay/node3401 | \
    cpio -idmv -D /sysroot
done
```

At this point, `/sysroot` contains:
- Complete Linux rootfs from the container
- Node-specific configuration from system overlays
- All software installed during container build

#### Step 10: Preboot Phase (91preboot module)

The `preboot.sh` script executes before switching to the new root:

```bash
# Create secrets mount point
mkdir $NEWROOT/secrets

# Mount secrets from Warewulf NFS
WAREWULF=$(echo "$wwinit_uri" | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+')
mount -t nfs $WAREWULF:/warewulf/secrets $NEWROOT/secrets

# Extract playbook name from build info
export PLAYBOOK=$(grep -oP '^Playbook:\s*\K.*' $NEWROOT/.buildinfo)

# Execute Ansible in chroot
chroot $NEWROOT bash << EOF
    source /opt/ansible/bin/activate
    cd /container_repo/ansible
    ansible-playbook -vvv -e @/secrets/secrets.yml $PLAYBOOK \
      -i localhost, -c local -l localhost \
      --tags "secret,preboot" &>> /var/log/dracut-preboot.log
EOF

# Unmount secrets
umount $NEWROOT/secrets
rmdir $NEWROOT/secrets
```

This allows the same Ansible playbook used for building to configure the node at boot time with secrets.

**Example preboot tasks:**
- Write Munge keys
- Configure cluster-specific settings
- Apply encryption certificates
- Configure monitoring agents

#### Step 11: MkDisk Phase (92mkdisk module)

The `run_mkdisk.sh` script configures local storage:

```bash
if [ -f "$NEWROOT/etc/mkdisk/mkdisk.conf" ]; then
    source $NEWROOT/etc/mkdisk/mkdisk.conf
    
    # Bind mount proc, sys, run, dev
    mount --bind /proc $NEWROOT/proc
    mount --bind /sys $NEWROOT/sys
    mount --bind /run $NEWROOT/run
    mount --bind /dev $NEWROOT/dev
    
    if [ "$WIPE_ON_BOOT" = "true" ]; then
        chroot $NEWROOT /bin/mkdisk -r &>> $NEWROOT/mkdisk.log
    else
        chroot $NEWROOT /bin/mkdisk -c &>> $NEWROOT/mkdisk.log
    fi
    
    # Cleanup bind mounts
    umount $NEWROOT/proc $NEWROOT/sys $NEWROOT/run $NEWROOT/dev
fi
```

**MkDisk Configuration** (from node tags):

```ini
# /etc/mkdisk/mkdisk.conf
WIPE_ON_BOOT=true
MKDISK_PROFILE=default
FSCACHE_DIR=/scratch/fscache
MOVE_DIRS=/var/log:/system/log,/tmp:/system/tmp
```

**MkDisk Profile** (`profiles.d/compute-node`):

```bash
disk_config[1]=""                    # Disk 1: unused
disk_config[2]="raid1:ext4:/system"  # Disks 2-3: RAID1 for system
disk_config[4]="raid0:ext4:/scratch" # Disks 4-N: RAID0 for scratch
```

MkDisk will:
1. Detect available NVMe/SATA disks
2. Create software RAID arrays (mdadm)
3. Format filesystems
4. Mount at specified paths
5. Move directories from RAM to persistent storage

Example result:
- `/system` - RAID1 mirror for critical data (logs, etc.)
- `/scratch` - RAID0 stripe for high-performance scratch space
- `/var/log` → `/system/log` - Logs persist across reboots
- `/tmp` → `/system/tmp` - Temp files on fast storage

#### Step 12-13: Pivot Root and Init

After all dracut modules complete:

```bash
# Dracut performs pivot_root
pivot_root /sysroot /sysroot/mnt

# Execute /sbin/init (systemd)
exec /sbin/init
```

Systemd starts and brings up all services:
- Network configuration (from runtime overlays)
- Munge authentication
- Slurmd (Slurm compute daemon)
- SSH daemon
- Chrony (time sync)
- Monitoring agents
- NFS mounts (via autofs or fstab)

At this point, the compute node is fully operational.

### Runtime Overlays

After boot, Warewulf continues to provide runtime overlays dynamically:

```bash
# Warewulf client runs periodically
wwclient update

# Downloads and applies runtime overlays
curl http://10.1.224.50:9873/overlay/runtime/hosts/node3401 | cpio -idmv -D /
curl http://10.1.224.50:9873/overlay/runtime/ssh.authorized_keys/node3401 | cpio -idmv -D /
```

This allows:
- Dynamic updates to `/etc/hosts`
- User SSH key distribution
- Configuration changes without rebooting

## Summary of Boot Process

```
Power On
   ↓
PXE Boot → DHCP → iPXE
   ↓
Download Kernel & Initramfs (HTTP)
   ↓
Boot Kernel
   ↓
Dracut: wwinit module
   ├→ Download container rootfs (HTTP)
   ├→ Extract to RAM tmpfs
   └→ Apply system overlays
   ↓
Dracut: preboot module
   ├→ Mount secrets (NFS)
   ├→ Run Ansible (secret, preboot tags)
   └→ Unmount secrets
   ↓
Dracut: mkdisk module
   ├→ Detect local disks
   ├→ Create RAID arrays
   ├→ Format filesystems
   └→ Mount storage
   ↓
Pivot Root to Container Rootfs
   ↓
Init (Systemd)
   ├→ Network configuration
   ├→ NFS mounts
   ├→ Munge
   ├→ Slurmd
   ├→ SSH
   └→ Other services
   ↓
Node Ready - Accept Slurm Jobs
```

## Key Design Principles

### 1. Stateless Operation

- **No persistent OS state** - Everything in RAM
- **Reproducible** - Every boot is identical
- **Fast recovery** - Reboot = clean slate
- **No OS patching** - Update container, reboot nodes

### 2. Immutable Infrastructure

- **Container images** - Built once, used many times
- **Version control** - Images tagged and versioned
- **Rollback capability** - Change image name, reboot
- **Consistency** - All nodes run identical OS

### 3. Separation of Concerns

- **Container image** - OS and software (immutable)
- **System overlays** - Node-specific config (build time)
- **Runtime overlays** - Dynamic config (runtime)
- **Secrets** - Sensitive data (NFS, ephemeral)

### 4. Local Storage for Performance

- **OS in RAM** - Maximum speed, no I/O overhead
- **Local RAID** - High-performance scratch space
- **Ephemeral by default** - Clean on every boot
- **Optional persistence** - Directories moved to RAID

### 5. Centralized Management

- **Single Warewulf server** - Manages all nodes
- **Configuration as code** - Ansible inventory
- **Automated deployment** - No manual node setup
- **Dynamic updates** - Runtime overlay push

## Advantages of This Architecture

1. **Scalability** - Add nodes by editing inventory and rebooting
2. **Reliability** - Failed nodes simply reboot to working state
3. **Performance** - No disk I/O for OS, uses RAM
4. **Security** - Compromised node is clean after reboot
5. **Flexibility** - Multiple OS versions simultaneously
6. **Maintainability** - Update containers, not individual nodes
7. **Disaster Recovery** - Nodes self-heal on reboot

## Monitoring and Management

### Warewulf Commands

```bash
# List containers
wwctl container list

# List nodes
wwctl node list

# Show node configuration
wwctl node list node3401 -a

# Update node
wwctl node set node3401 --container rocky-9.5-compute

# Rebuild overlays
wwctl overlay build

# Power control (via IPMI)
wwctl power on node3401
wwctl power reset node3401
```

### Node Logs

During boot:
- **Kernel messages** - `dmesg`
- **Dracut logs** - `/run/initramfs/rdsosreport.txt`
- **Preboot log** - `/var/log/dracut-preboot.log`
- **MkDisk log** - `/mkdisk.log`

After boot:
- **System logs** - `journalctl`
- **Slurm logs** - `/var/log/slurm/slurmd.log`

### Health Monitoring

Node Health Check (NHC) runs periodically:
- Check hardware (memory, CPU, GPU)
- Verify filesystems mounted
- Test network connectivity
- Validate Slurm connectivity
- Mark node down if checks fail

Configuration: `/etc/nhc/nhc.conf` (in container)

This architecture provides a robust, scalable, and maintainable compute cluster infrastructure.




