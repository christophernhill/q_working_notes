# ORCD Quantori Repositories Overview

This document provides a comprehensive overview of the repositories within the `orcd-quantori` project. Collectively, these repositories implement a "Cluster as Code" architecture for High Performance Computing (HPC) environments, primarily leveraging Warewulf 4 for node provisioning, Ansible for configuration management, and Apptainer for containerized services.

## System Architecture High-Level View

The system is designed around the principle of immutable infrastructure where possible. Compute nodes and service nodes (like Open OnDemand) are provisioned using version-controlled container images.

1.  **Source of Truth**: The `orcd-quantori-warewulf-work` repository acts as the central definition of the cluster state (inventory, network config, node profiles).
2.  **Image Factory**: The `orcd-quantori-platform-work` repository builds the operating system images (containers) that nodes boot into.
3.  **Dependencies**: Repositories like `reposync` and `slurm-build` provide the raw materials (RPMs, DEBs) for the image builds.
4.  **Runtime Config**: Tools like `mkdisk` run on the nodes at boot time to configure local hardware.

---

## Repository Details

### 1. orcd-quantori-warewulf-work
**Role:** Central Deployment Controller ("The Control Plane")

This is the most critical repository for cluster operations. It contains the Ansible playbooks and inventory that define a specific cluster instance (e.g., "prod", "dev", "test").

*   **Key Responsibilities:**
    *   **Inventory Management**: Defines all hosts (`hosts.yml`), including compute nodes, login nodes, and the Warewulf master itself.
    *   **Warewulf Deployment**: The `warewulf.yml` playbook configures the Warewulf server, imports container images, and sets up node profiles.
    *   **Service Deployment**: Deploys containerized services like Open OnDemand (OOD) and Login nodes via playbooks like `ood.yml` and `login.yml`.
    *   **Runtime Overlays**: Manages file overlays that are injected into nodes at boot time (e.g., `hosts` files, network configs).
    *   **Live Updates**: The `live_update.yml` playbook allows for updating running nodes without a reboot for safe changes.

### 2. orcd-quantori-platform-work
**Role:** Container Build System ("The Factory")

This repository is responsible for building the operating system images (containers) that run on the cluster nodes. It uses Ansible to construct these images from base OS Docker images (Rocky Linux, Ubuntu).

*   **Key Responsibilities:**
    *   **OS Images**: Builds `rocky-8.10-compute`, `ubuntu-22.04-compute`, `warewulf-server`, etc.
    *   **Role Management**: Contains modular Ansible roles (e.g., `slurmd`, `munge`, `sssd`, `nvidia`) that are applied to the images during the build process.
    *   **CI/CD**: GitHub Actions workflows in this repo automatically build and push these images to the container registry (GHCR) when changes are committed.
    *   **OOD Build**: Builds the Open OnDemand container image (`ood-4.0.yml`).

### 3. orcd-quantori-mkdisk
**Role:** Node Disk Configuration Utility

This repository contains the source code for the `mkdisk` tool, which is installed into the compute node images.

*   **Key Responsibilities:**
    *   **Disk Partitioning**: Runs at boot time on compute nodes.
    *   **Storage Setup**: Detects available disks and configures them according to a profile (e.g., RAID 0 for scratch, partitions for local logs).
    *   **State Management**: Ensures disks are formatted and mounted correctly for the node's operation (providing `/scratch`, `/var/log`, etc.).

### 4. orcd-quantori-slurm-build
**Role:** Slurm Packaging

Since Slurm is not always available in standard repositories or specific versions are needed, this repository handles the building of Slurm packages.

*   **Key Responsibilities:**
    *   **RPM/DEB Building**: Contains scripts and spec files to build Slurm packages from source.
    *   **Version Control**: Ensures the exact version of Slurm needed for the cluster is available for the `platform-work` build process.

### 5. orcd-quantori-reposync
**Role:** Package Mirror Management

To ensure reproducible builds and network efficiency, this repository manages local mirrors of upstream Linux repositories.

*   **Key Responsibilities:**
    *   **Mirroring**: Scripts to sync repositories for Rocky Linux 8/9 and Ubuntu 22/04/24.04.
    *   **Dependency Assurance**: Ensures that the `platform-work` builds have access to all required packages even if upstream repos change or are temporarily unavailable.

### 6. orcd-quantori-checkmk-work
**Role:** Monitoring Infrastructure

This repository manages the deployment and configuration of the Checkmk monitoring system.

*   **Key Responsibilities:**
    *   **Server Config**: Configures the Checkmk site, admin users, and global settings.
    *   **Host Registration**: Automates the registration of cluster hosts into the monitoring system.
    *   **Rule Management**: Configures monitoring rules (e.g., TCP checks, service checks) via the Checkmk API.

### 7. orcd-quantori-platform-doc
**Role:** Documentation Hub

Contains operational documentation, architecture guides, and runbooks for the platform.

### 8. orcd-quantori-ood
**Role:** Deprecated / Placeholder

Currently contains only a LICENSE file.
*   **Note**: The actual Open OnDemand container build logic is in `platform-work`, and the deployment logic is in `warewulf-work`. This repository is likely a placeholder or deprecated reference.

---

## Data Flow & Relationships

The repositories work together in a supply chain:

1.  **`reposync`** fetches upstream packages.
2.  **`slurm-build`** compiles Slurm packages.
3.  **`platform-work`** uses those packages to build OS container images (e.g., `rocky-8-compute.img`).
4.  **`warewulf-work`** pulls those images and configures the Warewulf server to serve them to physical nodes.
5.  **`mkdisk`** (inside the image) configures the physical disks when the node boots.
6.  **`checkmk-work`** connects to the running nodes to monitor their health.

This separation of concerns allows for:
*   **Reproducibility**: Images are built once and deployed many times.
*   **Flexibility**: Different hardware profiles can be managed via `warewulf-work` without rebuilding images.
*   **Reliability**: Changes to the OS are tested in the build phase (`platform-work`) before reaching production.

