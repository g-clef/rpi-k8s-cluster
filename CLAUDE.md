# Raspberry Pi Kubernetes Cluster - Project Guide

## Overview
This repository contains Ansible playbooks for deploying a highly available Kubernetes cluster on 20 Raspberry Pi devices using network boot architecture. The setup eliminates the need for SD cards by implementing diskless operation with all storage provided by a Synology NAS.

## Architecture Components

### Hardware Infrastructure
- **20 Raspberry Pi nodes** - All configured for network boot (diskless)
- **3 control plane nodes** (pi01-pi03) - K3s server nodes
- **14 worker nodes** (pi04-pi17) - K3s agent nodes  
- **1 GPU node** - K3s agent node with dedicated taint for GPU workloads
- **Synology NAS** - Provides NFS storage and hosts the control VM
- **Control VM** (nas-vm at 192.168.1.50) - DHCP/TFTP server and Ansible controller

### Network Configuration
- **Network Range**: 192.168.1.0/24
- **DHCP Pool**: 192.168.1.51-150
- **Pi Node IPs**: 192.168.1.101-117
- **NAS IP**: 192.168.1.197
- **DHCP Server IP**: 192.168.1.50

## Ansible Roles Structure

### Infrastructure Setup Roles
1. **pxe_dhcp_tftp** - Network boot infrastructure
   - Configures dnsmasq for DHCP/TFTP services
   - Downloads Raspberry Pi firmware
   - Prepares boot files for network booting

2. **nfs_roots** - Diskless filesystem preparation
   - Downloads and processes Raspberry Pi OS images
   - Creates individual NFS root filesystems per node
   - Configures overlay filesystems for diskless operation

### Kubernetes Deployment Roles
3. **k3s_pi_servers** - Control plane setup
   - Configures first node for cluster initialization
   - Sets up additional servers to join cluster
   - Uses K3s version v1.29.7+k3s1

4. **k3s_pi_agents** - Worker node configuration
   - Installs K3s agent services
   - Connects worker nodes to control plane

5. **node_labels** - Post-deployment configuration
   - Applies ARM64 architecture labels
   - Configures GPU node taints and specialized workload scheduling

6. **argocd** - GitOps platform deployment
   - Installs Argo CD for continuous deployment workflows

7. **kuberay** - Ray operator installation
   - Installs KubeRay operator for Ray cluster management
   - Ray clusters are deployed via GitOps from dedicated repository

## Storage Architecture
- **NFS Export Path**: `/volume2/nfsroot` on Synology NAS
- **Local Mount**: `/mnt/nfsroot` on control VM
- **Per-node Roots**: Individual filesystem trees for each Pi
- **Overlay FS**: Enables read-write operations on read-only NFS roots

## Deployment Workflow

The main playbook (`cluster-netboot-ha/site.yaml`) executes in two phases:
1. **Initial deployment** - Sets up all infrastructure and K3s components
2. **Post-deployment** - Applies node labels and taints (re-runnable)

### Execution Order
1. DHCP/TFTP service configuration
2. NFS root filesystem creation
3. K3s server deployment (control plane)
4. K3s agent deployment (worker nodes)
5. MinIO S3 service deployment
6. Argo CD installation and application configuration
7. KubeRay operator installation
8. Node labeling and tainting

Note: Ray clusters are deployed via GitOps - ArgoCD pulls Ray cluster manifests from a dedicated GitHub repository after the operator is installed.

## Key Configuration Files
- `inventory/hosts.yaml` - Node definitions with MAC addresses and IP assignments
- `group_vars/all.yaml` - Cluster-wide variables including network settings, K3s version, and storage paths
- Individual role templates for systemd services and configuration files

## Special Features
- **High Availability**: 3-node control plane for fault tolerance
- **GPU Node Support**: Dedicated node with taints for GPU workloads
- **ARM64 Optimization**: Configured specifically for ARM64 architecture
- **GitOps Ready**: Pre-configured with Argo CD for continuous deployment
- **Diskless Operation**: Complete elimination of SD card dependencies

## Usage Notes
- All operations are executed from the control VM (nas-vm)
- The playbook is designed to be idempotent and re-runnable
- Individual Pi nodes are identified by MAC address for consistent DHCP assignment
- Cluster uses overlay networking with custom CIDR ranges for pods and services

This architecture provides a robust, scalable Kubernetes platform ideal for ARM64 development, edge computing, and learning environments.
