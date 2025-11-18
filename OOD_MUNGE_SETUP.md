# OOD Munge Key Setup and Configuration

## Overview

This document explains how the `/etc/munge/munge.key` file is configured in OOD (Open OnDemand) containers, addressing the seemingly mysterious observation that:

1. The container image (`ood001.sif`) has an **empty** `/etc/munge/` directory
2. The running container has `/etc/munge/munge.key` correctly populated
3. There is **no bind mount** providing this file

**Answer:** The munge.key is deployed **after** the container starts, via Ansible SSH connection to the running container.

## Why Munge is Needed

Munge (MUNGE Uid 'N' Gid Emporium) provides authentication for communication between OOD and the Slurm scheduler:

- **Purpose**: Cryptographic authentication for inter-process communication
- **OOD Use Case**: When OOD submits jobs to Slurm on behalf of users
- **Requirement**: All systems communicating with Slurm must share the **same** munge.key
- **Security**: The munge.key must be kept secret and have strict permissions (0600, owned by munge:munge)

## Deployment Architecture

The munge.key deployment follows a two-phase process:

```
Phase 1: Container Infrastructure (Host)
    ↓
Phase 2: Application Configuration (Inside Container via SSH)
```

## Detailed Deployment Flow

### Phase 1: Host Infrastructure Setup (Master Role)

**Executed on:** Physical host (e.g., `core015`)  
**Role:** `master`  
**Playbook:** `ood.yml`

The master role running on the host system:

1. Creates directory structure at `/shared/services/ood/ood001/`
2. Pulls the container image from GHCR
3. Creates the overlay filesystem
4. Generates start/stop scripts
5. Creates systemd service
6. **Starts the container** with systemd service

**Container Start Command:**

```bash
# From /shared/services/ood/ood001/bin/start.sh
apptainer instance start \
  --boot \                              # Runs systemd as PID 1
  --network macvlan_ood001 \            # Dedicated network IP
  -B /home/systems/slurm/etc/slurm:/etc/slurm \
  -B /shared \
  -B /home \
  -B /orcd \
  -B $DIRNAME/root:/root \
  --overlay overlay.img \
  containers/container.sif ood001
```

**Important:** Notice that `/etc/munge` is **NOT** in the bind mount list.

At this point:
- Container is running with systemd as PID 1
- SSH daemon (sshd) starts inside the container (part of systemd boot)
- Container has its own IP address via macvlan (e.g., `ood001.inband`)
- `/etc/munge/` directory exists but is empty (from container image)

### Phase 2: Application Configuration (OOD Role)

**Executed on:** Inside the running container (via SSH)  
**Role:** `ood`  
**Playbook:** `ood.yml`

After the container starts, Ansible connects to it via SSH and configures the application.

#### Step 1: Wait for SSH

The playbook waits for the container's SSH service to be available:

```yaml
# From ood.yml
- name: Wait for container SSH to be available
  delegate_to: localhost
  ansible.builtin.wait_for:
    host: "{{ hostvars[service].ansible_host }}"  # ood001.inband
    port: "{{ ansible_port | default(22) }}"
    delay: 5 
    timeout: 60
```

This ensures the container is fully booted and SSH is accepting connections.

#### Step 2: SSH Connection Details

Ansible connects to the container using parameters from the inventory:

```yaml
# From inventory/prod/host_vars/ood001.yml
ansible_host: ood001.inband
ansible_ssh_private_key_file: "{{ lookup('env','CLUSTER_KEY') }}"
ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
```

**Connection Details:**
- **Target**: `ood001.inband` (container's macvlan IP)
- **Port**: 22 (SSH inside container)
- **Authentication**: SSH key from `CLUSTER_KEY` environment variable
- **User**: root (default for container SSH)

#### Step 3: Execute OOD Role

Once connected via SSH, Ansible runs the `ood` role inside the container:

```yaml
# From ood.yml
- name: Execute ood role
  include_role:
    name: ood
  tags: always, live_updates
```

#### Step 4: Deploy Munge Key

The OOD role includes a task that writes the munge.key:

```yaml
# From roles/ood/tasks/main.yml (lines 119-127)
- name: write decoded munge key to /etc/munge/munge.key with 0600 permissions
  ansible.builtin.copy:
    dest: /etc/munge/munge.key
    content: "{{ MUNGE_KEY | b64decode }}"
    mode: '0600'
    owner: munge
    group: munge
  register: munge_key
  tags: secret
```

**What This Does:**
1. Takes the `MUNGE_KEY` variable (base64-encoded secret)
2. Decodes it from base64
3. Writes the decoded content to `/etc/munge/munge.key`
4. Sets ownership to `munge:munge`
5. Sets permissions to `0600` (read/write for owner only)

#### Step 5: Start Munge Service

After the key is deployed, the munge service is started:

```yaml
# From roles/ood/tasks/main.yml (lines 135-139)
- name: Starting munge
  ansible.builtin.service:
    name: munge
    enabled: true
    state: started
```

This starts the munge daemon (`munged`) inside the container, which reads the newly-deployed key.

## Secret Management

### Where MUNGE_KEY Comes From

The `MUNGE_KEY` variable is sourced from GitHub repository secrets/environment variables:

**GitHub Secret Name:** `ANSIBLE_MUNGE_KEY`

**Usage in Deployment:**

When running the Ansible playbook (typically in CI/CD or from a workstation with environment variables set):

```bash
export MUNGE_KEY="$ANSIBLE_MUNGE_KEY"  # Set from GitHub secret
ansible-playbook -i inventory/prod/hosts.yml ood.yml
```

**Secret Format:**
- The actual munge.key file is binary data
- It's base64-encoded for storage as a GitHub secret
- Ansible decodes it when deploying: `{{ MUNGE_KEY | b64decode }}`

### Secret Source Locations

The secret is referenced in multiple places:

1. **GitHub Secrets**: `ANSIBLE_MUNGE_KEY` stored in repository settings
2. **Environment Variable**: Exported in CI/CD or deployment environment
3. **Ansible Variable**: `MUNGE_KEY` used in playbooks
4. **Target File**: `/etc/munge/munge.key` inside the container

### Security Considerations

**Why Not Include in Container Image?**

The munge.key is **intentionally NOT** baked into the container image because:

1. **Security**: Container images may be distributed or cached where they could be accessed
2. **Flexibility**: Different environments (dev, test, prod) use different keys
3. **Secret Rotation**: Keys can be rotated without rebuilding container images
4. **Least Privilege**: Build systems don't need access to production secrets

**Why Not Use Bind Mount?**

The munge.key is **not bind-mounted** because:

1. **Overlay Persistence**: Once written, it persists in the container's overlay filesystem
2. **Single Deployment**: Only needs to be deployed once (or when updated)
3. **File Ownership**: Easier to set proper ownership (munge:munge) when written directly
4. **Simplicity**: No host filesystem dependencies

## Complete Deployment Sequence Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ Ansible Control Node (Workstation or CI/CD)                    │
│ Environment: MUNGE_KEY = <base64-encoded secret>               │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ ansible-playbook ood.yml
                     ↓
┌─────────────────────────────────────────────────────────────────┐
│ Physical Host: core015                                          │
│                                                                  │
│  [Phase 1: Master Role - executed via delegate_to]              │
│    1. Create /shared/services/ood/ood001/                       │
│    2. Pull container image                                      │
│    3. Create overlay.img                                        │
│    4. Generate start.sh and stop.sh                             │
│    5. Create systemd service                                    │
│    6. systemctl start ood001.service                            │
│         ↓                                                        │
│    ┌──────────────────────────────────────────┐                │
│    │ Apptainer Container: ood001              │                │
│    │ IP: ood001.inband (via macvlan)          │                │
│    │                                           │                │
│    │ System Boot:                              │                │
│    │   1. systemd starts as PID 1             │                │
│    │   2. sshd starts (port 22)               │                │
│    │   3. /etc/munge/ exists but empty        │                │
│    │                                           │                │
│    │ Waiting for SSH...                       │                │
│    └──────────────────────────────────────────┘                │
│                                                                  │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ Wait for port 22 on ood001.inband
                     ↓
┌─────────────────────────────────────────────────────────────────┐
│ SSH Connection: ood001.inband:22                                │
│ Key: CLUSTER_KEY                                                │
│                                                                  │
│  [Phase 2: OOD Role - executed via SSH into container]          │
│    Inside Container (ood001):                                   │
│                                                                  │
│    Task 1: Generate OOD configs                                 │
│    Task 2: Deploy SSL certificates                              │
│    Task 3: Deploy Shibboleth configs                            │
│    Task 4: Write munge.key                                      │
│         ↓                                                        │
│       ansible.builtin.copy:                                     │
│         dest: /etc/munge/munge.key                              │
│         content: "{{ MUNGE_KEY | b64decode }}"                  │
│         mode: '0600'                                            │
│         owner: munge                                            │
│         group: munge                                            │
│         ↓                                                        │
│       /etc/munge/munge.key created! ✓                           │
│                                                                  │
│    Task 5: Start shibd                                          │
│    Task 6: Start munge                                          │
│         ↓                                                        │
│       systemctl start munge                                     │
│       munged reads /etc/munge/munge.key ✓                       │
│                                                                  │
│    Task 7: Start httpd                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Result: OOD container running with munge.key properly configured
```

## Why This Approach Works

### Overlay Filesystem Persistence

Once the munge.key is written via Ansible:

1. **Written to overlay**: The file goes into `/shared/services/ood/ood001/overlay/overlay.img`
2. **Persists across restarts**: When the container restarts, the overlay is remounted
3. **No re-deployment needed**: The key remains in place unless the overlay is deleted

### Verification After Deployment

**Check from host:**

```bash
# On core015
apptainer exec instance://ood001 ls -la /etc/munge/munge.key

# Expected output:
-rw------- 1 munge munge 1024 Nov 15 10:30 /etc/munge/munge.key
```

**Check from inside container:**

```bash
# On core015
apptainer shell instance://ood001

# Inside container:
ls -la /etc/munge/munge.key
systemctl status munge
```

**Check via SSH (same as Ansible):**

```bash
# From workstation
ssh -i $CLUSTER_KEY root@ood001.inband "ls -la /etc/munge/munge.key"
```

## Other Secrets Deployed the Same Way

The same SSH-based deployment method is used for other secrets in OOD containers:

### SSL/TLS Certificates

```yaml
- name: "Write SSL cert"
  ansible.builtin.copy:
    dest: /etc/pki/tls/certs/localhost.crt
    content: "{{ ood_ssl_cert | b64decode }}"
    mode: '0644'

- name: "Write SSL key"
  ansible.builtin.copy:
    dest: /etc/pki/tls/private/localhost.key
    content: "{{ ood_ssl_key | b64decode }}"
    mode: '0600'
```

### Shibboleth Keys

```yaml
- name: write decoded key
  ansible.builtin.copy:
    dest: /etc/shibboleth/sp-signing-key.pem
    content: "{{ shib_signing_key }}"
    mode: '0600'
    owner: shibd
    group: shibd

- name: write decoded cert
  ansible.builtin.copy:
    dest: /etc/shibboleth/sp-signing-cert.pem
    content: "{{ shib_signing_cert }}"
    mode: '0600'
    owner: shibd
    group: shibd
```

### Shibboleth IdP Metadata

```yaml
- name: write decoded cert
  ansible.builtin.copy:
    dest: /etc/shibboleth/okta.xml
    content: "{{ shib_okta_xml }}"
    mode: '0600'
    owner: shibd
    group: shibd
```

All of these secrets are:
- Stored as GitHub repository secrets
- Passed as environment variables
- Base64-decoded when deployed
- Written via Ansible SSH connection to the running container
- Persisted in the overlay filesystem

## Login Containers Use the Same Pattern

Login containers also need munge.key for Slurm integration:

```yaml
# From roles/login/tasks/main.yml (lines 24-31)
- name: write decoded munge key to /etc/munge/munge.key with 0600 permissions
  ansible.builtin.copy:
    dest: /etc/munge/munge.key
    content: "{{ MUNGE_KEY | b64decode }}"
    mode: '0600'
    owner: munge
    group: munge
  register: munge_key
```

The deployment flow is identical:
1. Master role starts container on host
2. Wait for SSH
3. Connect via SSH
4. Login role deploys configs and secrets
5. Services start with proper authentication

## Troubleshooting

### Scenario 1: Munge Key Not Present After Deployment

**Symptoms:**
```bash
apptainer exec instance://ood001 ls /etc/munge/
# Empty or munge.key missing
```

**Possible Causes:**

1. **SSH connection failed**: Check if Ansible could connect
   ```bash
   ssh -i $CLUSTER_KEY root@ood001.inband
   ```

2. **MUNGE_KEY variable not set**: Verify environment variable
   ```bash
   echo $MUNGE_KEY  # Should show base64 string
   ```

3. **OOD role didn't execute**: Check Ansible output for errors
   ```
   TASK [write decoded munge key to /etc/munge/munge.key] *****
   ```

4. **Permissions issue**: Check if munge user exists
   ```bash
   apptainer exec instance://ood001 id munge
   ```

### Scenario 2: Munge Service Fails to Start

**Symptoms:**
```bash
apptainer exec instance://ood001 systemctl status munge
# Failed or inactive
```

**Debugging:**

1. Check key permissions:
   ```bash
   apptainer exec instance://ood001 ls -la /etc/munge/munge.key
   # Must be -rw------- munge munge
   ```

2. Check key validity:
   ```bash
   apptainer exec instance://ood001 file /etc/munge/munge.key
   # Should show: data or non-empty file
   ```

3. Check munge logs:
   ```bash
   apptainer exec instance://ood001 journalctl -u munge -n 50
   ```

4. Test key manually:
   ```bash
   apptainer exec instance://ood001 /usr/sbin/munged --foreground
   ```

### Scenario 3: Wrong Munge Key

**Symptoms:**
- OOD can't submit jobs to Slurm
- "Invalid credential" errors in logs

**Diagnosis:**
```bash
# Compare with Slurm controller's key (on slurm head node)
md5sum /etc/munge/munge.key

# Compare with OOD container's key
apptainer exec instance://ood001 md5sum /etc/munge/munge.key

# They must match!
```

**Resolution:**
Update the `ANSIBLE_MUNGE_KEY` secret in GitHub with the correct key, then redeploy.

### Scenario 4: Updating the Munge Key

**Process to rotate munge.key across cluster:**

1. **Generate new key** (on a secure system):
   ```bash
   dd if=/dev/urandom bs=1 count=1024 > new_munge.key
   base64 -w0 new_munge.key > new_munge.key.b64
   ```

2. **Update GitHub secret**:
   - Go to repository settings → Secrets
   - Update `ANSIBLE_MUNGE_KEY` with contents of `new_munge.key.b64`

3. **Deploy to all systems** (must be coordinated):
   ```bash
   # OOD containers
   ansible-playbook -i inventory/prod/hosts.yml ood.yml --tags secret
   
   # Login containers
   ansible-playbook -i inventory/prod/hosts.yml login.yml --tags secret
   
   # Slurm controller and compute nodes
   # (process depends on your Slurm deployment method)
   ```

4. **Restart munge everywhere**:
   ```bash
   # OOD containers
   apptainer exec instance://ood001 systemctl restart munge
   
   # Login containers
   apptainer exec instance://login005 systemctl restart munge
   ```

## Live Updates

The munge.key can be updated without full container redeployment:

```bash
# Update only secrets (tagged with 'secret')
ansible-playbook -i inventory/prod/hosts.yml ood.yml --tags secret -l ood001

# Inside container, munge service will be restarted automatically
```

This is useful for:
- Key rotation
- Fixing corrupted keys
- Updating other secrets (SSL certs, Shibboleth keys)

## Comparison: Build-Time vs Runtime Deployment

### Build-Time Inclusion (NOT USED for munge.key)

```
Container Build → Include munge.key → Push to GHCR → Deploy container
                  └──────────────────────────────┐
                                    ❌ Problems:  │
                                    - Key in image/layers
                                    - No per-environment keys
                                    - Requires rebuild for rotation
```

### Runtime Deployment (USED for munge.key)

```
Container Build → Empty /etc/munge → Push to GHCR → Deploy container
                                                           ↓
                                            Start container with systemd
                                                           ↓
                                              Wait for SSH to be ready
                                                           ↓
                                       Ansible SSH: Deploy munge.key
                                                           ↓
                                              Start munge service
                                              ✓ Secure
                                              ✓ Flexible
                                              ✓ No rebuild needed
```

## Summary

The `/etc/munge/munge.key` in OOD containers is deployed through a **two-phase process**:

1. **Phase 1 (Host)**: Container infrastructure is set up and container is started with systemd
2. **Phase 2 (Container)**: Ansible connects via SSH and deploys secrets including munge.key

**Key Points:**
- ✓ Key is **NOT** in the container image (security)
- ✓ Key is **NOT** bind-mounted (simplicity)
- ✓ Key is deployed **via Ansible SSH** after container boots
- ✓ Key persists in the **overlay filesystem**
- ✓ Same pattern used for **all secrets** (SSL, Shibboleth, etc.)
- ✓ Same pattern used for **login containers** too
- ✓ Can be updated with `--tags secret` without full redeployment

This approach balances security (secrets not in images), flexibility (per-environment keys), and operational simplicity (standard Ansible deployment patterns).

---

**Related Documentation:**
- `APPTAINER_LAUNCHERS.md` - Container launch mechanisms
- `OOD_CONTAINER_ORGANIZATION_AND_DEPLOYMENT.md` - Complete OOD architecture
- `orcd-quantori-warewulf-work/working_notes/secrets.md` - List of all secrets

**Document Version**: 1.0  
**Last Updated**: 2025-11-18  
**Maintainer**: Cluster Operations Team

