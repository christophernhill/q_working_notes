# Open OnDemand (OOD) Container Organization and Deployment

## Overview

This document describes how the Open OnDemand (OOD) web portal is containerized and deployed as a persistent Apptainer/Singularity service on existing host systems. The deployment uses a two-phase approach: first building a container image, then deploying it as a systemd-managed service.

## Container Build Phase

### Base Image Construction

The OOD container is built from a Rocky Linux 8.10 base using the `orcd-quantori-platform-work` repository. The build process is defined in `ansible/ood-4.0.yml` and constructs a container image with the following components:

#### Key Components Installed

1. **OOD Core Package** (`ondemand` version 4.0)
   - Installed from the OSC OnDemand repository
   - Includes Ruby 3.3 and Node.js 20 modules
   - Web interface components and portal generator

2. **Authentication Stack**
   - Shibboleth for federated authentication (MIT Touchstone/Okta integration)
   - PAM authentication modules (`mod_authnz_pam`)
   - SSSD for user identity management

3. **Job Scheduler Integration**
   - Munge for authentication with the Slurm cluster
   - Slurm client utilities (configured via `slurmd` role but service disabled)

4. **System Services**
   - Apache HTTPD web server
   - OpenSSH server
   - Rsyslog for logging

#### Build Roles Applied

The build playbook applies these Ansible roles in sequence:

```yaml
roles:
  - common/bootstrap        # Container initialization
  - create_repos            # Repository configuration
  - create_users            # User/group setup
  - munge                   # Munge authentication
  - sssd                    # Identity management
  - slurmd                  # Slurm client (service disabled)
  - ood                     # OOD installation
  - setup_container_env     # Container-specific configs
  - common/final_cleanup    # Image optimization
  - common/build_info       # Metadata injection
```

The container image is then pushed to GitHub Container Registry (GHCR) at:
```
oras://ghcr.io/mit-orcd/ood-4.0:main_latest
```

## Container Deployment Phase

### Deployment Architecture

The OOD container is deployed on a physical host using Apptainer in **instance mode** with systemd service management. This approach provides:

- Persistent running container with defined lifecycle
- Network isolation via macvlan networking
- Filesystem bind-mounts for cluster integration
- Systemd integration for automatic restart and management

### Deployment Process

The deployment is orchestrated by the `orcd-quantori-warewulf-work` repository's `ood.yml` playbook, which executes two main roles:

#### 1. Master Role - Container Infrastructure Setup

The `master` role (executed on the physical host) prepares the infrastructure:

**Directory Structure Created:**
```
/shared/services/ood/<servicename>/
├── bin/
│   ├── start.sh          # Container startup script
│   └── stop.sh           # Container shutdown script
├── containers/
│   └── container.sif     # Pulled OOD image
├── overlay/
│   └── overlay.img       # Writable overlay filesystem
└── root/
    └── .ssh/
        ├── authorized_keys
        └── cluster        # SSH keys for cluster access
```

**Key Configuration Steps:**

1. **Install Apptainer with SUID support** for privileged operations
2. **Network Configuration** - Creates macvlan interface config at `/etc/apptainer/network/macvlan_<servicename>.conflist`
3. **Overlay Creation** - Creates persistent writable layer (default 1GB, configurable)
4. **Image Pull** - Downloads the OOD container from GHCR
5. **Systemd Service Creation** - Generates service unit file

**Apptainer Instance Configuration:**

The container is launched with the following options (from `start.sh.j2`):

```bash
apptainer instance start \
  --boot \                           # Boot container as a system
  --network macvlan_<servicename> \  # Dedicated macvlan network
  -B /etc/slurm \                    # Slurm configuration
  -B /shared \                       # Shared storage
  -B /home \                         # User home directories
  -B /orcd \                         # Additional cluster paths
  -B $DIRNAME/root:/root \           # Container root home
  --overlay overlay.img \            # Writable layer
  containers/container.sif <servicename>
```

**Systemd Service:**

The container is managed as a systemd service (`/etc/systemd/system/<servicename>.service`):

```ini
[Unit]
Description=Container Service <servicename>
Wants=network-online.target
After=network-online.target
After=time-sync.target

[Service]
Type=forking
ExecStart=/shared/services/ood/<servicename>/bin/start.sh
ExecStop=/shared/services/ood/<servicename>/bin/stop.sh

[Install]
WantedBy=multi-user.target
```

#### 2. OOD Role - Application Configuration

The `ood` role (executed inside the running container) configures the OOD application:

**Configuration Files Deployed:**

1. **Portal Configuration** (`/etc/ood/config/ood_portal.yml`)
   - Server name and SSL certificates
   - Shibboleth authentication integration
   - Proxy configuration for compute nodes
   - User mapping handler script

2. **Cluster Definition** (`/etc/ood/config/clusters.d/engaging.yml`)
   - Slurm adapter configuration
   - Job submission defaults
   - Batch Connect environment setup

3. **Dashboard Environment** (`/etc/ood/config/apps/dashboard/env`)
   - Environment variables for user sessions
   - Module system integration

4. **Application Definitions** - Synchronized to `/var/www/ood/apps/sys/`:
   - `jupyter/` - Jupyter Lab application
   - `RStudio/` - RStudio Server application
   - `bc_desktop/` - Virtual desktop application

5. **Shibboleth Configuration** (`/etc/shibboleth/`)
   - Service provider metadata (`shibboleth2.xml`)
   - Signing and encryption keys
   - Okta IdP metadata

6. **Authentication Secrets:**
   - SSL certificates and keys (`/etc/pki/tls/`)
   - Munge key (`/etc/munge/munge.key`)
   - Shibboleth keys (`/etc/shibboleth/sp-*.pem`)

**Service Startup:**

After configuration, three services are started inside the container:

```yaml
- shibd    # Shibboleth daemon
- munge    # Munge authentication daemon  
- httpd    # Apache web server
```

### Network Configuration

The container uses **macvlan networking** to obtain its own IP address on the physical network:

- Container appears as a separate host on the network
- Direct access from clients without port mapping
- DNS name mapping (e.g., `dev-ood001.inband` or `ood001.inband`)
- SSL/TLS termination at the container's HTTPD

The macvlan configuration is generated per-instance and includes:
- Bridge interface on the host
- VLAN tagging (if required)
- IP address allocation
- Gateway configuration

### Host Variables

Each OOD instance is configured via host-specific variables in the inventory:

```yaml
# Example: inventory/prod/host_vars/ood001.yml
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

ood_servername: orcd-ood.mit.edu
ood_login_host: orcd-login.inband
```

## Key Features of This Deployment Model

### Advantages

1. **Immutable Infrastructure** - Container image is read-only; changes require rebuild
2. **Version Control** - Container images are versioned and stored in GHCR
3. **Rapid Deployment** - New instances can be deployed in minutes
4. **Isolation** - Each OOD instance runs in its own network namespace
5. **Persistence** - Overlay filesystem preserves container-local changes
6. **Integration** - Direct access to cluster filesystems and Slurm via bind mounts
7. **Systemd Management** - Standard service management tools apply

### Design Considerations

1. **Stateful Service** - The container runs persistently, not ephemeral per-user
2. **Single Entry Point** - All users access the same container instance
3. **User Sessions** - OOD spawns per-user sessions on compute nodes, not in the container
4. **Authentication** - Federated authentication via Shibboleth/Okta
5. **Authorization** - User mapping handled by `ood_authed_user_handler.sh`

## Application Integration

OOD provides "Batch Connect" applications that users launch from the web interface. These applications:

1. Submit Slurm jobs from within the container using the user's credentials
2. Jobs run on compute nodes with appropriate resources
3. Applications run in nested Apptainer containers (Jupyter, RStudio)
4. Web traffic is proxied back through OOD to the user's browser

The OOD container serves as a **gateway and proxy**, not as a compute environment.

## Security Model

### Authentication Flow

1. User accesses OOD portal (e.g., `https://orcd-ood.mit.edu`)
2. HTTPD redirects to Shibboleth authentication
3. User authenticates via MIT Touchstone (Okta)
4. Shibboleth passes authenticated identity to OOD
5. `ood_authed_user_handler.sh` maps federated ID to local username
6. OOD impersonates user for Slurm submissions (via Munge)

### Security Boundaries

- Container runs as root but isolated via Apptainer
- User jobs run on compute nodes, not in OOD container
- Munge provides authentication between OOD and Slurm
- SSL/TLS encrypts all web traffic
- Shibboleth prevents unauthorized access

## Maintenance and Updates

### Updating the OOD Container

1. Make changes in `orcd-quantori-platform-work` repository
2. Run the `ood-4.0.yml` build playbook
3. Push new image to GHCR
4. Update `container_url` in inventory (or use `:main_latest` tag)
5. Run `ood.yml` playbook with `deploy_forced` variable set
6. Service restarts with new container

### Configuration Changes

For configuration-only changes (no package updates):

1. Modify files in `orcd-quantori-warewulf-work/roles/ood/`
2. Run `ood.yml` playbook
3. Configuration is synchronized into running container
4. Restart services inside container as needed

### Monitoring and Logs

- Systemd status: `systemctl status <servicename>`
- Container logs: `apptainer instance list` and logs inside container
- OOD application logs: Inside container at `/var/log/httpd/` and `/var/log/ondemand-nginx/`
- Slurm integration logs: Standard Slurm accounting and logs

## Summary

The OOD deployment uses a sophisticated containerized architecture that:

- **Builds** OOD as a complete system container with all dependencies
- **Deploys** the container as a persistent systemd service with dedicated networking
- **Configures** OOD applications and cluster integration inside the running container
- **Integrates** with Slurm for job submission and user authentication
- **Provides** a web gateway for users to access cluster resources

This approach balances the benefits of containerization (reproducibility, isolation) with the needs of a production web service (persistence, networking, integration).

