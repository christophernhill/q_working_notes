# Plan for Creating PREREQS.md

This document outlines the strategy to analyze the `orcd-quantori` codebases and deduce the full hierarchy of prerequisites required to deploy and operate the system. The output will be a document `PREREQS.md` in `q_working_notes`.

## Objective

Create a hierarchical map of dependencies, distinguishing between:
1.  **DevOps/Build Prerequisites**: Services needed to build and deploy (CI/CD, secrets, registries).
2.  **Operational/Runtime Prerequisites**: Core infrastructure services required for the cluster to function (Network, Identity, Time, Storage).

## Task Breakdown

### Phase 1: Repository Analysis

#### 1. Analyze DevOps & Security Dependencies
*   **Target**: `.github/workflows` (all repos), `ansible/vars`, `group_vars`.
*   **What to look for**:
    *   GitHub Secrets usage (e.g., SSH keys, API tokens).
    *   Container Registry requirements (GHCR authentication).
    *   Git repository access (SSH keys for submodules or dependent repos).
    *   Ansible Vault usage or external secret stores.

#### 2. Analyze Identity and Access Management (IAM)
*   **Target**: `orcd-quantori-platform-work/roles` (specifically `sssd`, `ldap`, `duo`), `orcd-quantori-ood` configs.
*   **What to look for**:
    *   LDAP/AD server requirements (URIs, bind DNs).
    *   Kerberos/KDC requirements (`krb5.conf`).
    *   Duo/MFA integration keys.
    *   Certificate Authorities (CAs) for SSL/TLS.

#### 3. Analyze Core Network Services
*   **Target**: `orcd-quantori-warewulf-work/inventory`, `orcd-quantori-platform-work/roles` (`chrony`, `network`).
*   **What to look for**:
    *   **DNS**: Upstream resolvers, required internal zones/records.
    *   **NTP**: Upstream time servers (`chrony.conf`).
    *   **DHCP**: Subnets managed by Warewulf vs. upstream DHCP relays.
    *   **Network Topology**: VLANs, gateways, IPMI networks defined in `hosts.yml`.

#### 4. Analyze Storage & Compute Infrastructure
*   **Target**: `mkdisk` configs, Ansible mount tasks, `fstab` templates.
*   **What to look for**:
    *   NFS servers (shared `/home`, `/software`).
    *   Parallel filesystems (Lustre/GPFS client requirements).
    *   Hardware requirements (Disk types for `mkdisk`, GPU types for drivers).

### Phase 2: Synthesis and Hierarchy Construction

Organize findings into a dependency tree.
*   **Level 0: Physical/Virtual Infrastructure** (Network fabric, Power, IPMI).
*   **Level 1: Core Network Services** (DNS, NTP).
*   **Level 2: Identity & Security** (LDAP, Kerberos, Secrets).
*   **Level 3: DevOps Infrastructure** (Git, Registry, CI Runners).
*   **Level 4: Deployment Control Plane** (Warewulf Master).
*   **Level 5: Compute/Service Nodes** (The actual cluster).

### Phase 3: Drafting PREREQS.md

Structure the document to be actionable for a site reliability engineer setting up a new environment.

## Execution Steps

1.  **Scan `platform-work` for Identity/Auth**: Grep for `sssd`, `ldap`, `krb5` configurations.
2.  **Scan `warewulf-work` for Network/Infra**: Examine `inventory` files and `warewulf.conf` templates.
3.  **Scan `github` workflows**: Identify all `secrets.*` references.
4.  **Scan for Storage Mounts**: Check for NFS exports/mounts in playbooks.
5.  **Compile Findings**: Create the hierarchy.
6.  **Write `PREREQS.md`**.

