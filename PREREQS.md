# System Prerequisites

This document outlines the hierarchy of services and infrastructure required to deploy and operate the `orcd-quantori` HPC cluster. It distinguishes between external dependencies (services the cluster consumes) and internal components (services the cluster provides).

## 1. External Infrastructure Services
These services must exist and be reachable before the cluster can be deployed.

### 1.1. Network Services
*   **DNS**:
    *   Upstream resolvers are required for the Master node and Compute nodes (e.g., MIT campus DNS).
    *   Reverse DNS for the cluster subnet is recommended.
*   **NTP**:
    *   A reliable time source is critical for Kerberos and logging.
    *   Current Config: `time.mit.edu`.
*   **Package Mirrors**:
    *   Local or upstream HTTP/HTTPS mirrors for OS packages (Rocky Linux 8/9, Ubuntu 22/24).
    *   Current Config: `http://core014.inband/` (internal mirror).

### 1.2. Identity & Access Management (IAM)
The cluster relies on external sources for user identity and authentication.

*   **Kerberos (Authentication)**:
    *   Service: MIT Kerberos (`kerberos.mit.edu`).
    *   Realm: `ATHENA.MIT.EDU`.
    *   Requirement: Keytabs for services (OOD, SSH) if not using user-only auth.
*   **LDAP (Authorization & Directory)**:
    *   Service: Internal LDAP cluster (`ldap00?.inband`).
    *   Schema: RFC2307 (posixAccount, posixGroup) and Automount maps.
    *   Base DN: `dc=cm,dc=cluster`.
    *   Requirement: Bind credentials (if not anonymous) and TLS trust chain.
*   **Duo Security (MFA)**:
    *   Service: Duo Cloud.
    *   Requirement: Integration keys (IKEY, SKEY, HOST) for SSH and Web authentication.

### 1.3. Storage Infrastructure
*   **Cluster Filesystem**:
    *   An NFS or parallel filesystem server must export the primary data directories.
    *   Mount Point: `/orcd` (or similar high-performance storage).
    *   Usage: User Home directories (`/orcd/home`), Project data.
    *   *Note: Automount maps in LDAP direct nodes to these exports.*

---

## 2. DevOps & Build Prerequisites
These secrets and services are required to build the software stack using the "Cluster as Code" pipelines.

### 2.1. Secrets Management (GitHub Actions)
The following secrets must be present in the GitHub repository settings:

| Secret Name | Description | Usage |
| :--- | :--- | :--- |
| `GHCR_SECRET` | Personal Access Token (PAT) with `read:packages` scope. | Pulling base images and pushing build artifacts to GHCR. |
| `CONTAINER_REPO_KEY` | SSH Private Key (Ed25519/RSA). | Accessing the private `orcd-quantori-platform-work` repo during deployment. |
| `SSH_HOSTKEYS` | Base64 encoded tarball of SSH host keys. | Ensuring persistent host identity for nodes. |
| `WW_MASTER_KEY` | SSH Private Key. | Ansible access to the Warewulf Master host. |
| `WAREWULF_CLUSTER_KEY` | SSH Private Key. | Ansible access to the Warewulf container and Compute nodes. |

### 2.2. Container Registry
*   **Registry**: GitHub Container Registry (`ghcr.io`).
*   **Namespace**: `mit-orcd` (or target organization).
*   **Requirement**: Authentication token for push/pull operations.

---

## 3. Operational Prerequisites (The "Bootstrap" Layer)
These are the first things that must be deployed to bring the cluster up.

### 3.1. Master Node (Physical/Virtual Host)
*   **OS**: Rocky Linux 8/9 or compatible EL distribution.
*   **Network**:
    *   One interface on the management/external network.
    *   One interface on the private cluster network (for PXE/DHCP).
*   **Software**:
    *   `apptainer` (with suid support).
    *   `git`.
    *   `ansible` (for running the deployment playbooks).

### 3.2. Warewulf Service (Containerized)
*   **Role**: Provides DHCP, TFTP, and HTTP for node booting.
*   **Dependencies**:
    *   Must run in `--privileged` mode (or with specific capabilities) to manage network services.
    *   Requires the `GHCR_SECRET` to pull the Warewulf container image.
    *   Requires write access to `/var/lib/warewulf` (persisted on Master host).

---

## 4. Compute Node Runtime Environment
Once booted, compute nodes depend on:

*   **Runtime Overlays**: Provided by Warewulf (contains `hosts`, `network`, `udev` rules).
*   **MkDisk**: A valid disk profile configuration to format local scratch/log storage.
*   **Network Access**:
    *   To Master Node (HTTP/TFTP).
    *   To LDAP/Kerberos/NTP servers.
    *   To NFS Storage servers.

