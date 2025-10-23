# InfiniBand MOFED Driver Installation in Compute Node Containers

## Overview

This document describes how Mellanox OpenFabrics Enterprise Distribution (MOFED) InfiniBand drivers are integrated into compute node container images during the build process. MOFED provides high-performance RDMA (Remote Direct Memory Access) capabilities for InfiniBand and Ethernet networks, which are essential for HPC cluster interconnect performance.

## What is MOFED?

### Mellanox OFED (OpenFabrics Enterprise Distribution)

MOFED is Mellanox/NVIDIA's tested and certified distribution of OpenFabrics software stack for InfiniBand and Ethernet networking. It provides:

- **Kernel Drivers** - Low-level hardware interface (mlx4, mlx5, mlx6 modules)
- **User-Space Libraries** - libibverbs, librdmacm, libmlx5, etc.
- **RDMA Protocols** - Support for RDMA over Converged Ethernet (RoCE), iWARP
- **Performance Tools** - ibstat, ibv_devinfo, perftest utilities
- **MPI Integration** - Optimized for MPI communication patterns
- **NFS over RDMA** - High-performance NFS using RDMA transport

### Why InfiniBand for HPC?

InfiniBand provides critical capabilities for HPC workloads:

| Capability | Ethernet | InfiniBand (MOFED) |
|------------|----------|-------------------|
| **Latency** | ~1-10 µs | ~0.5-1 µs |
| **Bandwidth** | 10-100 Gb/s | 100-400 Gb/s (HDR/NDR) |
| **CPU Overhead** | High | Very Low (RDMA) |
| **MPI Performance** | Good | Excellent |
| **Scalability** | Limited | Excellent (Subnet Manager) |
| **Hardware Cost** | Lower | Higher |

**RDMA (Remote Direct Memory Access)** allows direct memory-to-memory transfers without CPU involvement, critical for:
- Tightly-coupled MPI applications
- High-throughput data movement
- Low-latency communication
- Parallel filesystem performance

## MOFED in Container Build Process

### Role Location and Structure

The MOFED installation is handled by the `mofed` role:

```
ansible/roles/mofed/
└── tasks/
    ├── main.yml      # Entry point
    ├── Rocky.yml     # RHEL/Rocky-specific installation
    └── Ubuntu.yml    # Ubuntu-specific installation
```

### Build Playbook Integration

MOFED is included in all compute node builds. Example from `rocky-8.10-compute.yml`:

```yaml
- name: rocky-8.10-compute.yml
  hosts: localhost
  become: yes
  vars:
    bootstrap: "docker"
    build_from: "docker.io/rockylinux/rockylinux:8.10"
    image_name: "rocky-8.10-compute"
    distro: "rocky8"
    kernel_version: "4.18.0-553.56.1.el8_10.x86_64"
    nvidia_version: "580.65.06"
    mofed_version: "23.10-5.1.4.0"           # ← MOFED version specification
    mirror_server: "http://core014.inband/"
    slurm_version: "24.11.6-1"
  
  roles:
    - common/bootstrap
    - create_repos
    - create_users
    - packages
    - mkdisk
    - dracut
    - munge
    - sssd
    - nhc
    - mofed                                   # ← MOFED role executed here
    - nvidia
    - chrony
    # ... additional roles
```

### MOFED Version Matrix

Different container variants use different MOFED versions:

| Container Image | OS Version | Kernel Version | MOFED Version |
|----------------|------------|----------------|---------------|
| rocky-8.10-compute | Rocky 8.10 | 4.18.0-553.56.1 | 23.10-5.1.4.0 |
| rocky-9.5-compute | Rocky 9.5 | 5.14.0-503.40.1 | 24.10-2.1.8.0 |
| ubuntu-22.04-compute | Ubuntu 22.04 | 6.8.0-1019-nvidia | 24.10-2.1.8.0 |
| ubuntu-24.04-compute | Ubuntu 24.04 | (varies) | 24.10-2.1.8.0 |
| rocky-8.10-login | Rocky 8.10 | 4.18.0-553.56.1 | 24.10-2.1.8.0 |
| rocky-8.10-debug | Rocky 8.10 | 4.18.0-553.56.1 | 24.10-2.1.8.0 |

**Note:** Rocky 8.10 compute nodes use an older MOFED version (23.10) while other variants use the newer 24.10 release. This may be due to:
- Hardware compatibility requirements
- Testing and validation status
- Known issues with specific kernel/MOFED combinations

### Version String Format

MOFED versions follow the format: `MAJOR.MINOR-BUILD.PATCH.SUBPATCH`

**Example:** `24.10-2.1.8.0`
- **24.10** - MOFED 24.10 release (October 2024)
- **2.1.8.0** - Build and patch level

## Rocky Linux/RHEL Installation Process

### Role Entry Point

**File:** `roles/mofed/tasks/main.yml`
```yaml
- name: install-kernel
  tags: build
  block:
  - name: Include distro-specific tasks
    include_tasks: "{{ ansible_distribution }}.yml"
```

The role dispatcher includes distro-specific tasks based on the detected distribution.

### Rocky/RHEL Installation Steps

**File:** `roles/mofed/tasks/Rocky.yml`
```yaml
- name: install mofed
  tags: build
  vars:
    mofed_url: "{{mirror_server}}/mellanox/rocky{{ ansible_distribution_major_version }}/"
    mofed_package: "MLNX_OFED_LINUX-{{mofed_version}}-rhel{{ ansible_distribution_version }}-x86_64"
  block:
    - name: download mofed package
      ansible.builtin.get_url:
        url: "{{mofed_url}}/{{mofed_package}}.tgz"
        dest: "/tmp/{{mofed_package}}.tgz"
        force: false

    - name: extract mofed package
      ansible.builtin.unarchive:
        src: "/tmp/{{mofed_package}}.tgz"
        dest: "/tmp"
        remote_src: false

    - name: install mofed
      ansible.builtin.shell:
        cmd: "./mlnxofedinstall --distro rhel{{ ansible_distribution_version }} --skip-repo --kernel {{kernel_version}} --add-kernel-support --hpc --with-nfsrdma --without-fw-update"
        chdir: "/tmp/{{mofed_package}}"
```

### Installation Step Breakdown

#### Step 1: Variable Configuration

```yaml
vars:
  mofed_url: "{{mirror_server}}/mellanox/rocky{{ ansible_distribution_major_version }}/"
  mofed_package: "MLNX_OFED_LINUX-{{mofed_version}}-rhel{{ ansible_distribution_version }}-x86_64"
```

**Example Resolution for Rocky 8.10:**
- `mirror_server`: `http://core014.inband/` (local package mirror)
- `mofed_url`: `http://core014.inband/mellanox/rocky8/`
- `mofed_package`: `MLNX_OFED_LINUX-23.10-5.1.4.0-rhel8.10-x86_64`
- Full download URL: `http://core014.inband/mellanox/rocky8/MLNX_OFED_LINUX-23.10-5.1.4.0-rhel8.10-x86_64.tgz`

**Mirror Server:** The `core014.inband` server is a local package mirror that caches:
- MOFED releases from NVIDIA
- OS packages
- NVIDIA GPU drivers
- Custom packages (mkdisk, warewulf-dracut)

This provides:
- Faster downloads during builds
- Consistent package versions
- Reduced external bandwidth
- Offline build capability

#### Step 2: Package Download

```yaml
- name: download mofed package
  ansible.builtin.get_url:
    url: "{{mofed_url}}/{{mofed_package}}.tgz"
    dest: "/tmp/{{mofed_package}}.tgz"
    force: false
```

Downloads the MOFED tarball to `/tmp/` inside the container being built.

**Package Size:** MOFED tarballs are typically 300-500 MB and contain:
- Source code for kernel modules
- Pre-compiled user-space libraries
- Installation scripts
- Documentation
- Tools and utilities

#### Step 3: Package Extraction

```yaml
- name: extract mofed package
  ansible.builtin.unarchive:
    src: "/tmp/{{mofed_package}}.tgz"
    dest: "/tmp"
    remote_src: false
```

Extracts to `/tmp/MLNX_OFED_LINUX-23.10-5.1.4.0-rhel8.10-x86_64/`

**Directory Structure:**
```
/tmp/MLNX_OFED_LINUX-23.10-5.1.4.0-rhel8.10-x86_64/
├── mlnxofedinstall          # Main installation script
├── docs/                    # Documentation
├── RPMS/                    # Pre-built RPM packages
│   ├── mlnx-ofed-kernel-modules-*.rpm
│   ├── mlnx-ofed-kernel-*.rpm
│   ├── libibverbs-*.rpm
│   ├── librdmacm-*.rpm
│   ├── infiniband-diags-*.rpm
│   └── ... (many more packages)
├── SRPMS/                   # Source RPMs
├── src/                     # Source code for kernel modules
└── uninstall.sh            # Uninstallation script
```

#### Step 4: MOFED Installation

```yaml
- name: install mofed
  ansible.builtin.shell:
    cmd: "./mlnxofedinstall --distro rhel{{ ansible_distribution_version }} --skip-repo --kernel {{kernel_version}} --add-kernel-support --hpc --with-nfsrdma --without-fw-update"
    chdir: "/tmp/{{mofed_package}}"
```

**Command Breakdown:**

| Flag | Purpose |
|------|---------|
| `--distro rhel8.10` | Specify target distribution |
| `--skip-repo` | Don't configure package repositories |
| `--kernel 4.18.0-553.56.1.el8_10.x86_64` | Build for specific kernel |
| `--add-kernel-support` | Build kernel modules from source |
| `--hpc` | Install HPC-specific packages and optimizations |
| `--with-nfsrdma` | Include NFS over RDMA support |
| `--without-fw-update` | Skip firmware updates (not needed in container) |

**The `mlnxofedinstall` Script:**

This is Mellanox's installation orchestrator that:
1. Detects system configuration
2. Selects appropriate packages
3. Resolves dependencies
4. Compiles kernel modules
5. Installs RPMs
6. Configures services

**Installation Output Includes:**
- Kernel module compilation (mlx4_core, mlx5_core, ib_core, etc.)
- User-space library installation
- Utility installation (ibstat, ibv_devinfo, perftest)
- Service configuration (openibd)

### Packages Installed

The `--hpc` flag installs a comprehensive set of packages:

**Kernel Modules:**
- `mlnx-ofed-kernel` - Core OFED kernel modules
- `mlx4_core` - ConnectX-3 hardware support
- `mlx5_core` - ConnectX-4/5/6/7 hardware support
- `ib_core` - InfiniBand core stack
- `ib_uverbs` - User-space verbs interface
- `rdma_cm` - RDMA Connection Manager
- `ib_ipoib` - IP over InfiniBand

**User-Space Libraries:**
- `libibverbs` - Verbs API for RDMA operations
- `librdmacm` - RDMA Connection Manager library
- `libmlx4` - User-space driver for ConnectX-3
- `libmlx5` - User-space driver for ConnectX-4/5/6/7
- `libibumad` - InfiniBand Management Datagram library
- `libibmad` - InfiniBand Management library

**NFS over RDMA:**
- `nfsrdma` - Kernel modules for NFS RDMA transport
- Configuration for `nfsd` RDMA support

**HPC Libraries and Tools:**
- `ucx` - Unified Communication X library (for MPI)
- `ucx-devel` - Development headers
- `sharp` - Scalable Hierarchical Aggregation and Reduction Protocol
- `hcoll` - Hierarchical Collectives library
- `openmpi` - Open MPI built with MOFED support (optional)

**Diagnostic Tools:**
- `infiniband-diags` - IB diagnostic utilities
- `perftest` - Performance testing tools
- `ibutils2` - Advanced management utilities
- `mft` - Mellanox Firmware Tools

**Management:**
- `opensm` - OpenSM subnet manager
- `ibacm` - InfiniBand Address Cache Manager

### Post-Installation

After installation completes:

**Services Enabled:**
- `openibd` - Loads InfiniBand kernel modules at boot
- (opensm is typically NOT enabled on compute nodes - runs on management node)

**Configuration Files Created:**
- `/etc/infiniband/openib.conf` - OFED configuration
- `/etc/modprobe.d/mlnx.conf` - Module parameters
- `/etc/udev/rules.d/*-mlnx.rules` - Device naming rules

**Kernel Modules Available:**
```bash
$ ls /lib/modules/4.18.0-553.56.1.el8_10.x86_64/extra/
mlnx-ofa_kernel/
├── drivers/infiniband/core/
│   ├── ib_core.ko
│   ├── ib_uverbs.ko
│   ├── rdma_cm.ko
│   └── ...
├── drivers/infiniband/hw/mlx4/
│   ├── mlx4_ib.ko
│   └── ...
├── drivers/infiniband/hw/mlx5/
│   ├── mlx5_ib.ko
│   └── ...
├── drivers/net/ethernet/mellanox/mlx4/
│   ├── mlx4_core.ko
│   └── mlx4_en.ko
└── drivers/net/ethernet/mellanox/mlx5/core/
    ├── mlx5_core.ko
    └── ...
```

## Ubuntu Installation Process

### Ubuntu-Specific Steps

**File:** `roles/mofed/tasks/Ubuntu.yml`
```yaml
- name: install mofed
  tags: build
  vars:
    mofed_url: "{{mirror_server}}/mellanox/ubuntu{{ ansible_distribution_major_version }}/"
    mofed_package: "MLNX_OFED_LINUX-{{mofed_version}}-ubuntu{{ ansible_distribution_version }}-x86_64"

  block:
    - name: download mofed package
      ansible.builtin.get_url:
        url: "{{mofed_url}}/{{mofed_package}}.tgz"
        dest: "/tmp/{{mofed_package}}.tgz"
        force: false

    - name: extract mofed package
      ansible.builtin.unarchive:
        src: "/tmp/{{mofed_package}}.tgz"
        dest: "/tmp"
        remote_src: false

    - name: remove conflicting packages
      ansible.builtin.package:
        name:
          - librbd1
          - librados2
          - fio
          - librdmacm1t64
        state: absent

    - name: install mofed
      ansible.builtin.shell:
        cmd: "./mlnxofedinstall --distro ubuntu{{ ansible_distribution_version }} --skip-repo --kernel {{kernel_version}} --add-kernel-support --hpc --with-nfsrdma --without-fw-update"
        chdir: "/tmp/{{mofed_package}}"

    - name: enable openibd service
      ansible.builtin.shell:
        cmd: systemctl enable openibd

    - name: disable opensmd service
      ansible.builtin.shell:
        cmd: systemctl disable opensmd
```

### Key Differences from Rocky

#### 1. Conflicting Package Removal

```yaml
- name: remove conflicting packages
  ansible.builtin.package:
    name:
      - librbd1         # Ceph RADOS Block Device library
      - librados2       # Ceph RADOS library
      - fio             # Flexible I/O tester
      - librdmacm1t64   # Ubuntu's RDMA CM library
    state: absent
```

**Why Remove These?**

- **librbd1/librados2** - Ceph libraries that may conflict with MOFED's versions
- **fio** - May have been built against system RDMA libraries
- **librdmacm1t64** - Ubuntu's distribution RDMA library conflicts with MOFED's version

MOFED provides its own versions of these libraries optimized for Mellanox hardware.

#### 2. Explicit Service Management

```yaml
- name: enable openibd service
  ansible.builtin.shell:
    cmd: systemctl enable openibd

- name: disable opensmd service
  ansible.builtin.shell:
    cmd: systemctl disable opensmd
```

**Service Configuration:**

- **openibd** - Enabled to load IB modules at boot
- **opensmd** - Disabled because:
  - Subnet Manager should run on management/core nodes, not compute nodes
  - Multiple SMs can cause fabric instability
  - Compute nodes are Subnet Managed Agents (clients), not managers

#### 3. Package Format

Ubuntu uses `.deb` packages instead of `.rpm`, but `mlnxofedinstall` handles this automatically:

**Example Package Name:**
- Rocky: `MLNX_OFED_LINUX-24.10-2.1.8.0-rhel8.10-x86_64.tgz`
- Ubuntu: `MLNX_OFED_LINUX-24.10-2.1.8.0-ubuntu22.04-x86_64.tgz`

## Integration with Compute Node Boot

### Kernel Module Loading

When compute nodes boot via Warewulf:

1. **Initramfs Phase:**
   - Basic network drivers loaded for PXE/HTTP boot
   - Container rootfs downloaded to RAM

2. **Early Boot (systemd):**
   - `openibd.service` starts
   - Loads InfiniBand kernel modules:
     ```bash
     modprobe mlx5_core
     modprobe mlx5_ib
     modprobe ib_ipoib
     modprobe rdma_cm
     modprobe ib_umad
     ```

3. **Module Verification:**
   ```bash
   $ lsmod | grep mlx
   mlx5_ib               389120  0
   ib_uverbs             155648  1 mlx5_ib
   ib_core               413696  4 rdma_cm,ib_ipoib,ib_uverbs,mlx5_ib
   mlx5_core            1605632  1 mlx5_ib
   mlxfw                  32768  1 mlx5_core
   tls                   102400  1 mlx5_core
   pci_hyperv_intf        16384  1 mlx5_core
   psample                20480  1 mlx5_core
   ```

### Network Interface Configuration

InfiniBand interfaces are configured via Warewulf overlays:

**Node Configuration:** `inventory/prod/host_vars/node3401.yml`
```yaml
network_devices:
  internal:
    type: ethernet
    device: boot0
    hwaddr: 6C:92:CF:EE:90:80
    ipaddr: 10.1.34.1
    netmask: 255.255.0.0
    primary: true
  
  ib:
    type: infiniband           # ← InfiniBand interface
    device: ib5                # ← Device name (may vary: ib0, ib5, etc.)
    ipaddr: 172.16.34.1        # ← IP on IB network
    netmask: 255.255.0.0
    primary: false
```

**Warewulf applies this configuration as:**

1. **NetworkManager Connection:**
   - Creates connection profile for `ib5`
   - Sets IP address `172.16.34.1/16`
   - Configures MTU (typically 65520 for IB)

2. **Interface Activation:**
   ```bash
   $ ip link show ib5
   5: ib5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65520 qdisc mq state UP mode DEFAULT group default qlen 256
       link/infiniband 80:00:00:48:fe:80:00:00:00:00:00:00:6c:92:cf:ee:90:80:00:00 brd 00:ff:ff:ff:ff:12:40:1b:ff:ff:00:00:00:00:00:00:ff:ff:ff:ff
   
   $ ip addr show ib5
   5: ib5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65520 qdisc mq state UP group default qlen 256
       link/infiniband 80:00:00:48:fe:80:00:00:00:00:00:00:6c:92:cf:ee:90:80:00:00 brd 00:ff:ff:ff:ff:12:40:1b:ff:ff:00:00:00:00:00:00:ff:ff:ff:ff
       inet 172.16.34.1/16 brd 172.16.255.255 scope global ib5
          valid_lft forever preferred_lft forever
   ```

3. **IPoIB Mode:**
   - Default: **Datagram mode** (MTU 2044)
   - Can be configured for **Connected mode** (MTU 65520, lower latency)

### Verifying InfiniBand at Runtime

On a booted compute node:

**Check HCA (Host Channel Adapter) Status:**
```bash
$ ibstat
CA 'mlx5_0'
    CA type: MT4123
    Number of ports: 1
    Firmware version: 20.35.1012
    Hardware version: 0
    Node GUID: 0x6c92cfee9080
    System image GUID: 0x6c92cfee9080
    Port 1:
        State: Active
        Physical state: LinkUp
        Rate: 100 Gb/sec (HDR)
        Base lid: 1234
        LMC: 0
        SM lid: 1
        Capability mask: 0x2651e848
        Port GUID: 0x6c92cfee9080
        Link layer: InfiniBand
```

**Check IB Devices:**
```bash
$ ibv_devinfo
hca_id: mlx5_0
    transport:                      InfiniBand (0)
    fw_ver:                         20.35.1012
    node_guid:                      6c92:cfee:9080
    sys_image_guid:                 6c92:cfee:9080
    vendor_id:                      0x02c9
    vendor_part_id:                 4123
    hw_ver:                         0x0
    board_id:                       MT_0000000223
    phys_port_cnt:                  1
        port:   1
            state:                  PORT_ACTIVE (4)
            max_mtu:                4096 (5)
            active_mtu:             4096 (5)
            sm_lid:                 1
            port_lid:               1234
            port_lmc:               0x00
            link_layer:             InfiniBand
```

**Performance Testing:**
```bash
# On node1 (server):
$ ib_write_bw

# On node2 (client):
$ ib_write_bw node1

# Results show:
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 TX depth        : 128
 CQ Moderation   : 100
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             98.52              98.50               0.187691
---------------------------------------------------------------------------------------
```

## NFS over RDMA Configuration

The `--with-nfsrdma` flag enables NFS to use RDMA transport.

### Server Configuration

NFS server needs to listen on RDMA:

```bash
# /etc/nfs.conf
[nfsd]
rdma=y
rdma-port=20049

# Start NFS with RDMA support
systemctl restart nfs-server
```

### Client Configuration

Compute nodes can mount NFS with RDMA:

```bash
# Traditional NFS (TCP)
mount -t nfs nfs-server:/export /mnt/data

# NFS over RDMA
mount -t nfs -o rdma,port=20049 nfs-server:/export /mnt/data
```

**Performance Comparison:**

| Transport | Bandwidth | Latency | CPU Usage |
|-----------|-----------|---------|-----------|
| NFS/TCP | ~1 GB/s | ~100 µs | High |
| NFS/RDMA | ~10 GB/s | ~10 µs | Low |

### Benefits of NFS/RDMA

1. **Zero-Copy** - Data moves directly between NFS server memory and client application memory
2. **Kernel Bypass** - RDMA bypasses kernel network stack
3. **Low CPU** - Offloads to HCA, frees CPU for computation
4. **High Throughput** - Saturates InfiniBand bandwidth
5. **Low Latency** - Sub-10µs read/write operations

## Hardware Compatibility

### Supported Mellanox/NVIDIA Adapters

MOFED supports multiple generations of InfiniBand adapters:

| Generation | Product Family | Speed | Notes |
|------------|---------------|-------|-------|
| ConnectX-3 | VPI | 40/56 Gb/s | FDR/FDR10, Ethernet 40GbE |
| ConnectX-4 | VPI | 100 Gb/s | EDR, Ethernet 100GbE |
| ConnectX-5 | VPI | 100 Gb/s | EDR, Enhanced features |
| ConnectX-6 | VPI | 200 Gb/s | HDR, Ethernet 200GbE |
| ConnectX-7 | VPI | 400 Gb/s | NDR, Ethernet 400GbE |

**VPI:** Virtual Protocol Interconnect - supports both InfiniBand and Ethernet on same hardware

### Detected Hardware

The kernel modules auto-detect installed hardware:

```bash
$ lspci | grep Mellanox
04:00.0 Infiniband controller: Mellanox Technologies MT28908 Family [ConnectX-6]
```

The `mlx5_core` driver binds to ConnectX-4/5/6/7 devices automatically.

## Troubleshooting

### Issue: Kernel Modules Not Loading

**Symptoms:**
- `ibstat` shows no devices
- `lsmod | grep mlx` shows nothing
- `/dev/infiniband` doesn't exist

**Solutions:**

1. **Check module files exist:**
   ```bash
   ls -l /lib/modules/$(uname -r)/extra/mlnx-ofa_kernel/
   ```

2. **Manual module load:**
   ```bash
   modprobe mlx5_core
   modprobe mlx5_ib
   ```

3. **Check dmesg for errors:**
   ```bash
   dmesg | grep mlx5
   ```

4. **Verify openibd service:**
   ```bash
   systemctl status openibd
   systemctl restart openibd
   ```

### Issue: Wrong MOFED Version

**Symptoms:**
- Performance issues
- Incompatibility with applications
- Missing features

**Check installed version:**
```bash
ofed_info -s
# Output: MLNX_OFED_LINUX-23.10-5.1.4.0
```

**Solution:**
Rebuild container with desired MOFED version by updating playbook variable.

### Issue: Firmware Mismatch

**Symptoms:**
- Driver loads but device doesn't activate
- Errors in dmesg about firmware

**Check firmware version:**
```bash
ibstat | grep "Firmware version"
```

**Note:** Firmware updates are typically NOT performed in containers. Physical hosts should update firmware via:
```bash
mst start
mlxfwmanager
```

### Issue: InfiniBand Port Not Active

**Symptoms:**
- `ibstat` shows "Physical state: LinkDown"
- No IB connectivity

**Diagnosis:**
```bash
# Check physical link
ibstat

# Check cable and switch
ibdiagnet

# Verify subnet manager
ibstat | grep "SM lid"
```

**Common Causes:**
- Cable not connected
- Switch port disabled
- No Subnet Manager running
- Incompatible speeds/widths

## Performance Optimization

### Module Parameters

MOFED modules can be tuned via `/etc/modprobe.d/mlnx.conf`:

```bash
# Example optimizations
options mlx5_core num_of_groups=1
options ib_ipoib send_queue_size=512 recv_queue_size=512
```

### MPI Configuration

For optimal MPI performance with MOFED:

**UCX (Unified Communication X):**
```bash
export UCX_NET_DEVICES=mlx5_0:1
export UCX_IB_GPU_DIRECT_RDMA=yes
export UCX_TLS=rc,ud,sm,self
```

**Open MPI:**
```bash
mpirun --mca btl openib,self,sm \
       --mca btl_openib_if_include mlx5_0 \
       -np 128 ./my_app
```

### Monitoring

**Real-time bandwidth monitoring:**
```bash
watch -n 1 'cat /sys/class/infiniband/mlx5_0/ports/1/counters/*_bytes'
```

**Performance counters:**
```bash
perfquery mlx5_0 1
```

## Summary

The MOFED driver installation process:

1. **Specified per container** via `mofed_version` variable
2. **Downloaded from local mirror** for speed and consistency
3. **Built for specific kernel** version during container build
4. **Includes HPC optimizations** via `--hpc` flag
5. **Supports NFS/RDMA** for high-performance storage
6. **Different versions** for different OS/kernel combinations
7. **Automatically loads** at boot via `openibd` service

The result is a container with full InfiniBand/RDMA capabilities that can:
- Achieve near-wire-speed MPI performance
- Use RDMA for low-latency communication
- Mount NFS with RDMA transport
- Support advanced HPC communication patterns
- Leverage GPU Direct RDMA for machine learning workloads

This infrastructure is critical for HPC cluster performance, providing the high-bandwidth, low-latency interconnect required for tightly-coupled parallel applications.

