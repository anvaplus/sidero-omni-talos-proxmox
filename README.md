# Sidero Omni + Talos + Proxmox Integration

A complete infrastructure-as-code setup for deploying Talos Kubernetes clusters on Proxmox Virtual Environment using Sidero Omni's infrastructure provider.

## Overview

This repository provides a production-ready configuration for:
- **Sidero Omni**: Kubernetes cluster management platform
- **Talos Linux**: Secure, immutable, and minimal Kubernetes OS
- **Proxmox VE**: Open-source virtualization platform

The integration enables automated provisioning of Kubernetes clusters with:
- Automated VM creation and lifecycle management on Proxmox
- Static IP or DHCP network configuration
- High-availability control plane support (3 nodes)
- Machine classes for different node types (control plane, workers)
- Infrastructure-as-code cluster templates

## Repository Structure

```
.
├── proxmox-provider/       # Omni infrastructure provider for Proxmox
│   ├── docker-compose.yaml # Provider container orchestration
│   ├── .env.example        # Example environment variables
│   ├── config.yaml.example # Example Proxmox configuration
│   └── README.md           # Detailed provider setup guide
│
├── machine-classes/        # VM resource specifications
│   ├── control-plane.yaml  # Control plane node specs (2 CPU, 4GB RAM)
│   ├── worker.yaml         # Worker node specs (4 CPU, 8GB RAM)
│   └── README.md           # Machine class documentation
│
├── patches/                # Patches for Talos machine configuration
│   ├── cni.yaml            # Disables default CNI (enables Cilium installation)
│   ├── disable-kube-proxy.yaml # Disables kube-proxy for Cilium eBPF mode
│   ├── longhorn.yaml       # Configures Longhorn distributed storage support
│   └── delete-HostnameConfig.yaml # Removes conflicting HostnameConfig for Talos 1.12+
│
└── cluster-template/       # Kubernetes cluster definitions
    ├── k8s-dev-static-ip.yaml   # Dev cluster with static IPs (1 CP + 3 workers)
    ├── k8s-dev-dhcp.yaml        # Dev cluster with DHCP (1 CP + 3 workers)
    ├── k8s-prod.yaml            # Production HA cluster (3 CP + 3 workers)
    └── README.md                # Cluster deployment guide
```

## Features

### Network Configuration
- **Static IP support**: Pre-configured IPs for all nodes with custom hostnames
- **DHCP support**: Dynamic IP assignment for simpler setups
- **Talos 1.12+ Compatibility**: Includes patches to handle `HostnameConfig` resource changes (see `patches/delete-HostnameConfig.yaml`) to avoid conflicts with stable hostnames.
- **Custom DNS and NTP**: Configurable nameservers and time servers
- **Gateway configuration**: Full network routing control

### High Availability
- **Multi-master control plane**: 3 control plane nodes for production
- **Load-balanced API server**: Distributed Kubernetes API
- **etcd clustering**: Distributed key-value store across control planes

### Infrastructure Management
- **Machine classes**: Define VM specs (CPU, RAM, disk) for different node types
- **Cluster templates**: Declarative cluster definitions with version control
- **Automated provisioning**: VMs created automatically on Proxmox
- **System extensions**: iSCSI, NFS, QEMU guest agent support

### Patches
- **`cni.yaml`**: Disables the default Flannel CNI installation in Talos, allowing alternative CNI providers (like Cilium) to be installed separately after cluster creation. This is required when using CNI solutions that provide their own networking implementation.
- **`disable-kube-proxy.yaml`**: Disables `kube-proxy` on all cluster nodes. This is essential when running Cilium in eBPF mode, as Cilium replaces kube-proxy functionality with its own eBPF-based implementation for superior performance and security.
- **`delete-HostnameConfig.yaml`**: Removes the default `HostnameConfig` resource (which defaults to `auto: stable`) to allow static hostnames to be set without conflict in Talos 1.12+. This addresses a known issue where applying a static hostname patch fails if the stable hostname feature is already active.
- **`longhorn.yaml`**: Configures Longhorn distributed storage support by mounting and binding `/var/mnt/longhorn` with proper kubelet extra mounts. This enables nodes to support Longhorn as a persistent storage backend for applications requiring block storage.

## Quick Start

### Prerequisites

- **Proxmox VE 9.x.x**: Running virtualization cluster
- **Sidero Omni**: Accessible Omni instance (cloud or self-hosted)
- **Docker & Docker Compose**: For running the infrastructure provider
- **omnictl CLI**: Omni command-line tool installed

### 1. Set Up the Proxmox Provider

All commands should be run from the root of the repository.

```bash
# Copy configuration examples
cp proxmox-provider/.env.example proxmox-provider/.env
cp proxmox-provider/config.yaml.example proxmox-provider/config.yaml

# Edit configuration files inside the proxmox-provider/ directory:
# - Set OMNI_API_ENDPOINT and OMNI_INFRA_PROVIDER_KEY in .env
# - Set Proxmox URL, tokenID, and tokenSecret in config.yaml

# Secure files
chmod 600 proxmox-provider/.env proxmox-provider/config.yaml

# Start the provider
docker compose -f proxmox-provider/docker-compose.yaml up -d

# Verify it's running
docker compose -f proxmox-provider/docker-compose.yaml logs -f
```

See [proxmox-provider/README.md](proxmox-provider/README.md) for detailed setup instructions.

### 2. Register Machine Classes

```bash
# Apply control plane and worker machine classes from the repository root
omnictl apply -f machine-classes/control-plane.yaml
omnictl apply -f machine-classes/worker.yaml

# Verify registration
omnictl get machineclasses
```

See [machine-classes/README.md](machine-classes/README.md) for customization options.

### 3. Deploy a Cluster

```bash
# For development (1 control plane + 3 workers, static IPs)
omnictl cluster template sync -v -f cluster-template/k8s-dev-static-ip.yaml

# For production (3 control planes + 3 workers, HA setup)
omnictl cluster template sync -v -f cluster-template/k8s-prod.yaml

# Monitor cluster creation
omnictl get clusters
omnictl get machines
```

See [cluster-template/README.md](cluster-template/README.md) for all available templates.

### 4. Install Cilium CNI

The cluster templates use patches to disable the default CNI and `kube-proxy`. This means you must install a CNI to get a fully functional cluster. In this case, we'll install Cilium.

After applying the cluster template, your nodes will appear as "Not Ready." This is expected behavior because Kubernetes nodes are only marked as `Ready` once a CNI is running and managing pod networking.

Follow these steps to install Cilium.

**1. Get Kubeconfig**

```bash
# Get the kubeconfig for your new cluster (e.g., k8s-dev)
omnictl get kubeconfig k8s-dev > k8s-dev.yaml
export KUBECONFIG=$(pwd)/k8s-dev.yaml
```

**2. Install Cilium using Helm**

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install \
    cilium \
    cilium/cilium \
    --version 1.18.0 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445
```
After running the Helm command, it will take a few minutes for the Cilium pods to be deployed and become operational. Once they are running, your Kubernetes nodes will transition to a "Ready" state, and your cluster will be fully networked with Cilium.


## Configuration Details

### Network Settings

All clusters use the following network configuration:
- **Gateway**: 10.20.0.1
- **DNS**: 10.20.0.2
- **NTP**: pool.ntp.org
- **Interface**: ens18

### IP Address Allocation

**Development Cluster** (`k8s-dev`):
- Control Plane: 10.20.10.11/16
- Workers: 10.20.10.15-17/16

**Production Cluster** (`k8s-prod`):
- Control Planes: DHCP (recommend DHCP reservations for 10.20.20.11-13)
- Workers: 10.20.20.15-17/16 (static IPs)

### Machine Specifications

**Control Plane Nodes**:
- 2 CPU cores, 1 socket
- 4 GB RAM
- 40 GB disk
- Network: vmbr0
- Storage: local-lvm

**Worker Nodes**:
- 4 CPU cores, 1 socket
- 8 GB RAM
- 60 GB disk
- Network: vmbr0
- Storage: local-lvm

### System Extensions

All nodes include:
- **siderolabs/iscsi-tools**: iSCSI initiator support
- **siderolabs/nfsd**: NFS server support
- **siderolabs/qemu-guest-agent**: Proxmox VM integration
- **siderolabs/util-linux-tools**: System utilities

## Cluster Templates

### Development Templates

**k8s-dev-static-ip.yaml** (Recommended)
- 1 control plane with static IP
- 3 workers with static IPs
- Hostnames: k8s-dev-cp-1, k8s-dev-worker-1/2/3
- Full network configuration (gateway, DNS, NTP)

**k8s-dev-dhcp.yaml**
- 1 control plane
- 3 workers using DHCP
- Hostnames: k8s-dev-cp-1, k8s-dev-worker (base)
- DNS and NTP configured, gateway from DHCP

### Production Template

**k8s-prod.yaml**
- 3 control planes (HA) using DHCP
- 3 workers with static IPs
- Hostnames: k8s-worker-1/2/3
- Full network configuration (DNS, NTP)
- Production-ready setup
- Note: Control planes use DHCP. Configure DHCP reservations for static-like IPs.

## Management Commands

### Infrastructure Provider

```bash
# View provider logs
docker compose -f proxmox-provider/docker-compose.yaml logs -f

# Restart provider
docker compose -f proxmox-provider/docker-compose.yaml restart

# Stop provider
docker compose -f proxmox-provider/docker-compose.yaml down
```

### Machine Classes

```bash
# List machine classes
omnictl get machineclasses

# View specific class
omnictl get machineclass proxmox-control-plane -o yaml

# Update machine class
omnictl apply -f machine-classes/control-plane.yaml

# Delete machine class
omnictl delete machineclass proxmox-worker
```

### Clusters

```bash
# List clusters
omnictl get clusters

# View cluster details
omnictl get cluster k8s-dev -o yaml

# Delete cluster
omnictl cluster delete k8s-dev
# or
omnictl cluster delete -f cluster-template/k8s-dev-static-ip.yaml
```

### Machines

```bash
# List all machines
omnictl get machines

# Watch machine provisioning
watch omnictl get machines

# View machine details
omnictl get machine <machine-name> -o yaml
```

## Technology Stack

- **[Sidero Omni](https://www.siderolabs.com/platform/saas-for-kubernetes/)**: Kubernetes management platform
- **[Talos Linux](https://www.talos.dev/)**: Immutable Kubernetes OS
- **[Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment)**: Virtualization platform
- **[omni-infra-provider-proxmox](https://github.com/siderolabs/omni-infra-provider-proxmox)**: Infrastructure provider

## Documentation

- [Proxmox Provider Setup](proxmox-provider/README.md)
- [Machine Classes Guide](machine-classes/README.md)
- [Cluster Templates Guide](cluster-template/README.md)
- [Sidero Omni Documentation](https://omni.siderolabs.com/docs/)
- [Talos Linux Documentation](https://www.talos.dev/latest/)

## Contributing

This is a personal infrastructure repository. Feel free to fork and adapt for your own environment.

## License

This repository configuration is provided as-is for reference and personal use.