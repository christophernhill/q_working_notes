# OOD Fakeroot Container Launch with Slurm

## Overview

This document explains how Open OnDemand (OOD) launches isolated, fakeroot-enabled Apptainer containers for interactive applications (Jupyter Lab and RStudio Server) on Slurm compute nodes. This approach provides users with root-like privileges within an isolated container environment while maintaining cluster security.

## Background Concepts

### Apptainer Fakeroot

**Fakeroot** is an Apptainer feature that allows unprivileged users to:
- Appear as root (UID 0) inside a container
- Install system packages with package managers (yum, apt, pip)
- Modify system directories within the container
- Run applications that require root privileges

**Key Properties:**
- Changes are ephemeral (stored in memory or temporary overlay)
- No actual root privileges on the host system
- Uses user namespaces for UID/GID mapping
- Requires `apptainer-suid` for implementation

**Implementation:**
```bash
apptainer instance start --fakeroot --writable-tmpfs container.sif myinstance
```

### OOD Batch Connect Applications

OOD provides "Batch Connect" apps that:
1. Present a web form to users (resource requirements, options)
2. Generate and submit a Slurm batch job
3. Monitor job status and provide connection details
4. Proxy the application's web interface to the user's browser

The two primary applications in this deployment are:
- **Jupyter Lab** - Interactive Python/data science environment
- **RStudio Server** - R development and analysis environment

## User Workflow

### Step 1: User Selects Application

The user navigates the OOD web interface and selects an application (e.g., Jupyter Lab). They are presented with a form (`form.yml`) requesting:

**Common Parameters:**
- **Time**: Maximum runtime in hours (1-12)
- **CPU**: Number of CPU cores (2-8)
- **Memory**: RAM in gigabytes (2-62)
- **GPU**: Optional GPU allocation checkbox
- **Fakeroot**: Enable fakeroot mode (experimental feature)

**Application-Specific Parameters:**
- **Jupyter**: Extra jupyter arguments
- **RStudio**: R version selection (3.5.3 - 4.5.1)

Example form definition (`jupyter/form.yml`):
```yaml
cluster: "engaging"

attributes:
  time:
    widget: "number_field"
    value: "4"
    min: 1
    max: 12
  
  cpu:
    value: 2
    min: 2
    max: 8
    
  fakeroot:
    widget: "check_box"
    label: "Fakeroot feature (experimental)"
    help: "Enable fakeroot feature. You may install packages. Changes are ephemeral."
    value: "0"
```

### Step 2: Job Submission to Slurm

When the user submits the form, OOD generates a Slurm batch script based on `submit.yml.erb`:

```yaml
batch_connect:
  template: "basic"
  conn_params:
    - csrf_token
    - fakeroot              # Fakeroot state passed as connection parameter
  set_host: "host=$(hostname -A | awk '{print $1}')"

script:
  native:
     - "-N"                          # Number of nodes
     - "1"
     - "-n <%= cpu %>"               # CPUs
     - "--mem=<%= memory %>G"        # Memory
     <% if gpu == "1" %>
     - "--gres=gpu:1"                # GPU resources
     - "--partition=mit_normal_gpu"
     <% else %>
     - "--partition=mit_normal"
     <% end %>
     - "--time=<%= time %>:00:00"    # Walltime
```

The OOD framework combines this with the cluster definition and generates a complete batch script that includes:
- Resource allocations
- Environment setup (from cluster config)
- `before.sh` script (connection setup)
- `script.sh` script (application launch)

### Step 3: Batch Script Execution

When Slurm allocates the job on a compute node, the generated batch script runs in this sequence:

#### 3a. Environment Setup

From the cluster definition (`engaging.yml`):
```bash
source /etc/profile.d/modules.sh
module purge
```

#### 3b. Before Script Execution

The `before.sh.erb` script runs first and performs connection setup:

**For Jupyter:**
```bash
# Find an available port
port=$(find_port)

# Generate a random password
SALT="$(create_passwd 16)"
password="$(create_passwd 16)"
PASSWORD_SHA1="$(echo -n "${password}${SALT}" | openssl dgst -sha1 | awk '{print $NF}')"

# Create Jupyter configuration
export CONFIG_FILE="${PWD}/config.py"
cat > "${CONFIG_FILE}" << EOL
c.NotebookApp.ip = '0.0.0.0'
c.NotebookApp.port = ${port}
c.NotebookApp.password = u'sha1:${SALT}:${PASSWORD_SHA1}'
c.NotebookApp.base_url = '/node/${host}/${port}/'
c.NotebookApp.allow_origin = '*'
EOL
```

This establishes:
- `$host` - The compute node hostname
- `$port` - Randomly selected available port
- `$password` - Authentication password for the session
- `$CONFIG_FILE` - Path to application configuration

These variables are used by OOD to:
1. Display connection information to the user
2. Configure the reverse proxy
3. Provide authentication credentials

#### 3c. Container Launch Script

The `script.sh.erb` template contains the main application logic with **conditional fakeroot execution**:

## Jupyter Lab Launch Process

### Standard Mode (No Fakeroot)

```bash
# Start Apptainer instance without fakeroot
/home/systems/engaging-ood-ww/apptainer/bin/apptainer instance start \
  --writable-tmpfs \                    # Writable overlay in tmpfs
  --nv \                                # NVIDIA GPU support
  -B/etc/slurm \                        # Bind mount Slurm config
  -B/home/systems \                     # Bind mount system files
  -B/orcd \                             # Bind mount cluster paths
  -B/var/run/munge \                    # Bind mount Munge socket
  -B/var/lib/sss \                      # Bind mount SSSD cache
  /home/systems/engaging-ood-ww/jupyter/container/container.sif \
  jupyter-$SLURM_JOB_ID                 # Instance name

# Execute Jupyter inside the instance
/home/systems/engaging-ood-ww/apptainer/bin/apptainer exec \
  --env PS1='\[\e[1;32m\]\u@\h:\w$ \[\e[0m\]' \
  instance://jupyter-$SLURM_JOB_ID \
  bash -c "
    source /jupyter/bin/activate
    jupyter lab --debug --config=\"${CONFIG_FILE}\"
  "
```

### Fakeroot Mode (When Enabled)

```bash
# Start Apptainer instance WITH fakeroot
/home/systems/engaging-ood-ww/apptainer/bin/apptainer instance start \
  --nv \
  --writable-tmpfs \
  --fakeroot \                          # Enable fakeroot mode
  -B/etc/slurm \
  -B/home/systems \
  -B/orcd \
  -B/var/run/munge \
  -B/var/lib/sss \
  -B/home/$USER:/root \                 # Map user's home to /root
  -B/home/$USER:/home/$USER \           # Also keep at original location
  /home/systems/engaging-ood-ww/jupyter/container/container.sif \
  jupyter-$SLURM_JOB_ID

# Execute Jupyter as root inside the instance
/home/systems/engaging-ood-ww/apptainer/bin/apptainer exec \
  --env PS1='\[\e[1;31m\]fake\u@\h:\w# \[\e[0m\]' \
  instance://jupyter-$SLURM_JOB_ID \
  bash -c "
    source /jupyter/bin/activate
    jupyter lab --allow-root --debug --config=\"${CONFIG_FILE}\"
  "
```

**Key Differences in Fakeroot Mode:**
1. `--fakeroot` flag enables user namespace remapping
2. `-B/home/$USER:/root` maps the user's home directory to `/root`
3. `--allow-root` permits Jupyter to run as UID 0
4. Red prompt (`\e[1;31m`) indicates fakeroot environment

## RStudio Server Launch Process

RStudio has a more complex launch process due to its architecture:

### Wrapper Script Generation

First, an `rsession.sh` wrapper is created to set the R version:

```bash
export RSESSION_WRAPPER_FILE="${PWD}/rsession.sh"
cat > "${RSESSION_WRAPPER_FILE}" << EOL
#!/usr/bin/env bash
export RSESSION_LOG_FILE="${PWD}/rsession.log"
exec &>>"\${RSESSION_LOG_FILE}"

# Set PATH to selected R version
export PATH=/opt/R/<%= context.rversion %>/bin/:$PATH

exec /usr/lib/rstudio-server/bin/rsession --r-libs-user "${R_LIBS_USER}" "\${@}"
EOL
chmod 700 "${RSESSION_WRAPPER_FILE}"
```

### Environment Setup

```bash
export TMPDIR="$(mktemp -d)"
mkdir -p "$TMPDIR/rstudio-server"
python3 -c 'from uuid import uuid4; print(uuid4())' > "$TMPDIR/rstudio-server/secure-cookie-key"
chmod 0600 "$TMPDIR/rstudio-server/secure-cookie-key"

# Configure RStudio environment
export RSTUDIO_AUTH=$CURRDIR/bin/auth
export RS_PROXY_PATH=/
export RS_WHICH_R=/opt/R/<%= context.rversion %>/bin/R
export RS_PORT=${port}
export RSTUDIO_SERVER_IMAGE=/home/systems/engaging-ood-ww/rstudio/container.sif
export RSESSION_PATH=$RSESSION_WRAPPER_FILE
export RSTUDIO_COOKIE=$TMPDIR/rstudio-server/secure-cookie-key
```

### Standard Mode Launch

```bash
export RS_USER=$USER
export RS_EXTRA_OPTS=""

/home/systems/engaging-ood-ww/apptainer/bin/apptainer instance start \
  -B /home/$USER \
  -B /var/run/munge \
  -B /var/lib/sss \
  -B /home \
  -B $TMPDIR:/tmp \
  -B /net \
  -B /nfs \
  -B /pool001 \
  -B /orcd \
  --writable-tmpfs \
  --nv \
  "$RSTUDIO_SERVER_IMAGE" \
  rstudio-server

# Run the RStudio server
/home/systems/engaging-ood-ww/apptainer/bin/apptainer run \
  instance://rstudio-server
```

The container's `%runscript` (defined in `rstudio.def`) then launches:
```bash
/usr/lib/rstudio-server/bin/rserver \
  --rsession-which-r $RS_WHICH_R \
  --www-address 0.0.0.0 \
  --server-daemonize 0 \
  --server-user $RS_USER \
  --www-port=$RS_PORT \
  --auth-pam-helper-path "${RSTUDIO_AUTH}" \
  --rsession-path ${RSESSION_PATH} \
  --secure-cookie-key-file ${RSTUDIO_COOKIE}
```

### Fakeroot Mode Launch

```bash
export RS_EXTRA_OPTS="--auth-minimum-user-id 0"    # Allow UID 0
export RS_USER=root                                # Run as root

/home/systems/engaging-ood-ww/apptainer/bin/apptainer instance start \
  -B /home/$USER \
  -B /var/run/munge \
  -B /var/lib/sss \
  -B /home \
  -B $TMPDIR:/tmp \
  -B /net \
  -B /nfs \
  -B /pool001 \
  -B /orcd \
  --fakeroot \                                     # Enable fakeroot
  --writable-tmpfs \
  --nv \
  "$RSTUDIO_SERVER_IMAGE" \
  rstudio-server

/home/systems/engaging-ood-ww/apptainer/bin/apptainer run \
  instance://rstudio-server
```

RStudio server then runs with:
- `--server-user root` - Server process runs as root
- `--auth-minimum-user-id 0` - Allows authentication as UID 0

## Container Definitions

### Jupyter Container

The Jupyter container (`jupyter.def`) is simple:

```singularity
bootstrap: oras
from: ghcr.io/mit-orcd/rocky-8.10-login:main_ab64

%post
    python3 -m venv /jupyter
    source /jupyter/bin/activate
    pip install jupyterlab
```

It builds on the standard login node container and adds:
- Python virtual environment at `/jupyter`
- JupyterLab via pip

### RStudio Container

The RStudio container (`rstudio.def`) is more complex:

```singularity
bootstrap: oras
from: ghcr.io/mit-orcd/rocky-8.10-login:main_ab64

%post
    # Install RStudio Server
    wget https://download2.rstudio.org/server/rhel8/x86_64/rstudio-server-rhel-2025.05.1-513-x86_64.rpm
    dnf install -y rstudio-server-rhel-2025.05.1-513-x86_64.rpm
    
    # Install multiple R versions (3.5.3 through 4.5.1)
    export R_VERSION=4.5.1
    curl -O https://cdn.posit.co/r/centos-8/pkgs/R-${R_VERSION}-1-1.x86_64.rpm
    dnf install -y R-${R_VERSION}-1-1.x86_64.rpm
    # ... (repeated for each R version)

%runscript
    # Dynamic port selection
    if [ -z $RS_PORT ]; then
        PORT=$(python3 -c 'import socket; s=socket.socket(); s.bind(("",0)); print(s.getsockname()[1]); s.close()')
        RS_PORT=$PORT
    fi
    
    # Launch RStudio Server
    /usr/lib/rstudio-server/bin/rserver \
        --rsession-which-r $RS_WHICH_R \
        --www-port=$RS_PORT \
        --server-user $RS_USER \
        ...
```

## Data Flow and Network Proxying

### Connection Establishment

1. **Job Starts** on compute node (e.g., `node1234`)
2. **Application Listens** on `node1234:${port}`
3. **OOD Detects** job is running and extracts connection info
4. **User Clicks "Connect"** in OOD interface

### Proxy Configuration

OOD's HTTPD is configured to reverse proxy to compute nodes:

From `ood_portal.yml`:
```yaml
host_regex: '[\w.-]+\.inband'
node_uri: '/node'
rnode_uri: '/rnode'
```

This creates proxy rules:
- `https://orcd-ood.mit.edu/node/node1234.inband/38472/` → `http://node1234.inband:38472/`

The Jupyter/RStudio application is configured with a base URL matching this path:
```python
c.NotebookApp.base_url = '/node/node1234.inband/38472/'
```

### User Experience

From the user's perspective:
1. Access OOD at `https://orcd-ood.mit.edu`
2. Submit Jupyter/RStudio job
3. Wait for job to start (shown in OOD interface)
4. Click "Connect to Jupyter" button
5. Browser opens `https://orcd-ood.mit.edu/node/node1234.inband/38472/lab`
6. Enter password (displayed in OOD)
7. Use Jupyter/RStudio normally

All traffic flows through OOD's reverse proxy, so:
- No direct access to compute nodes required
- SSL/TLS encryption end-to-end
- Authentication handled by OOD

## Fakeroot Use Cases

### What Fakeroot Enables

With fakeroot mode enabled, users can:

**In Jupyter:**
```bash
# Install system packages
yum install -y package-name

# Install Python packages system-wide
pip install --upgrade numpy scipy

# Modify system directories
echo "something" > /etc/myconfig

# Run applications requiring root
chown root:root file
```

**In RStudio:**
```r
# Install R packages to system library
install.packages("dplyr", lib="/usr/lib64/R/library")

# System-level operations within container
system("yum install -y gsl-devel")
```

### Limitations and Ephemeral Nature

**Important Constraints:**

1. **Ephemeral Storage** - `--writable-tmpfs` means:
   - Changes stored in memory-backed filesystem
   - All changes lost when container stops
   - Job completion = container cleanup

2. **No Persistent Root Access** - Users cannot:
   - Modify the actual host system
   - Affect other users' sessions
   - Persist changes across job submissions

3. **Resource Limits** - Writable tmpfs is bounded by:
   - Available memory
   - Kernel tmpfs limits
   - Large installations may fail

4. **Security Isolation** - Fakeroot provides:
   - UID namespace isolation
   - No actual privilege escalation
   - User still bound by Slurm cgroups

### Best Practices

**Recommended Workflow:**

1. **Test installations** in fakeroot mode
2. **Document requirements** (packages installed)
3. **Build custom container** with needed packages
4. **Deploy custom container** for production use
5. **Use fakeroot only for** ad-hoc testing

**Example Workflow:**
```bash
# In fakeroot Jupyter session
yum install -y special-library
pip install special-package
# Test that application works

# Then: Update jupyter.def
%post
    dnf install -y special-library
    source /jupyter/bin/activate
    pip install special-package

# Rebuild and deploy container
```

## Container Instance Management

### Instance Lifecycle

Each user session creates a uniquely-named instance:

**Jupyter:**
```bash
instance_name="jupyter-$SLURM_JOB_ID"
```

**RStudio:**
```bash
instance_name="rstudio-server"
```

The instance lifecycle is tied to the Slurm job:
1. Job starts → `apptainer instance start`
2. Job runs → Container executes application
3. Job ends → Slurm cleanup kills processes
4. Container cleanup → Instance automatically removed

### Manual Management

Users can inspect instances on compute nodes:

```bash
# List running instances
apptainer instance list

# Stop an instance manually
apptainer instance stop jupyter-12345

# Execute commands in instance
apptainer exec instance://jupyter-12345 command
```

## Security Considerations

### Fakeroot Security Model

**Safe Operations:**
- User isolated within user namespace
- UID 0 inside container = user's UID outside
- No access to host's root filesystem
- Bound by Slurm cgroup limits (CPU, memory, GPU)

**Attack Surface:**
- Kernel user namespace implementation (well-tested)
- Apptainer SUID helper (security-audited)
- No additional privileges outside container

**Limitations:**
- Cannot escape container
- Cannot affect other users
- Cannot bypass Slurm resource limits
- Cannot access host system resources

### Container Image Security

The application containers are:
- Built from verified base images (`ghcr.io/mit-orcd/rocky-8.10-login`)
- Stored in authenticated registry (GHCR)
- Pulled at job submission time (always fresh)
- Read-only except for writable-tmpfs overlay

### Cluster Integration Security

The containers have access to:
- **Munge socket** (`/var/run/munge`) - For Slurm authentication
- **SSSD cache** (`/var/lib/sss`) - For user identity resolution
- **Slurm config** (`/etc/slurm`) - For cluster connection
- **User filesystems** (`/home`, `/orcd`, etc.) - With user's permissions

These bind mounts ensure:
- Proper identity mapping
- Slurm integration for nested jobs
- Access to user's data
- No privilege escalation

## Troubleshooting

### Common Issues

**Issue: Fakeroot job fails to start**
```
Error: Failed to create user namespace
```
*Solution:* Check that `apptainer-suid` is installed on compute nodes

**Issue: Changes not persisting**
```
Installed packages gone after reconnect
```
*Solution:* This is expected behavior with `--writable-tmpfs`; changes are ephemeral

**Issue: Out of memory during package install**
```
Error: No space left on device
```
*Solution:* Writable tmpfs uses RAM; reduce installation size or increase job memory

**Issue: Container cannot access Slurm**
```
Error: Unable to contact slurm controller
```
*Solution:* Check that `/etc/slurm`, `/var/run/munge`, and `/var/lib/sss` are bound

### Debug Mode

Enable verbose output in job submission:

**Jupyter:**
```yaml
extra_jupyter_args: "--debug --log-level=DEBUG"
```

**RStudio:**
Set environment variables before launch:
```bash
export RS_LOG_LEVEL=debug
```

View logs:
- Jupyter: Output captured in Slurm job output file
- RStudio: `${PWD}/rsession.log`

## Summary

The OOD fakeroot container launch mechanism provides:

1. **User Choice** - Optional fakeroot mode via web form checkbox
2. **Isolation** - Each user job runs in its own container instance
3. **Flexibility** - Root-like privileges for package installation
4. **Security** - User namespace isolation prevents privilege escalation
5. **Ephemeral** - Changes are temporary and not persisted
6. **Integration** - Seamless access to cluster filesystems and Slurm
7. **Convenience** - Users access via browser through OOD proxy

This approach balances user flexibility (install packages on-demand) with system security (no persistent root access) and operational simplicity (no per-user container images). It enables rapid prototyping and testing while encouraging formalization of requirements into proper container definitions for production use.

