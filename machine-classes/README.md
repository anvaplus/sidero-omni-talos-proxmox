# Machine Classes

This directory contains Omni machine class definitions that specify the hardware resources for VMs provisioned on Proxmox via the Sidero Omni infrastructure provider.

## Overview

Machine classes define the VM specifications (CPU, memory, disk, network) that will be used when Omni provisions machines on Proxmox. These classes are referenced by cluster templates to determine what type of resources each node should receive.

## Available Machine Classes

### `control-plane.yaml`
**Machine class ID:** `proxmox-control-plane`

Control plane node specifications:
- **CPU**: 2 cores, 1 socket
- **Memory**: 4096 MB (4 GB)
- **Disk**: 40 GB
- **Network**: vmbr0 bridge
- **Storage**: local-lvm (via selector `name == "local-lvm"`)
- **Provider**: Proxmox

**Usage:** Control plane nodes run the Kubernetes control plane components (API server, etcd, scheduler, controller manager). These resources are suitable for dev/test clusters. For production HA clusters with 3 control planes, consider increasing memory to 8 GB.

### `worker.yaml`
**Machine class ID:** `proxmox-worker`

Worker node specifications:
- **CPU**: 4 cores, 1 socket
- **Memory**: 8192 MB (8 GB)
- **Disk**: 60 GB
- **Network**: vmbr0 bridge
- **Storage**: local-lvm (via selector `name == "local-lvm"`)
- **Provider**: Proxmox

**Usage:** Worker nodes run application workloads. These specs provide a good starting point for general-purpose workloads. Adjust based on your application requirements.

## Configuration Fields

### Provider Data
The `providerdata` section contains Proxmox-specific configuration:

- **`cores`**: Number of CPU cores per socket
- **`sockets`**: Number of CPU sockets
- **`memory`**: RAM in MB
- **`disk_size`**: Root disk size in GB
- **`network_bridge`**: Proxmox network bridge to attach VMs to (usually `vmbr0`)
- **`storage_selector`**: CEL expression to select Proxmox storage (e.g., `name == "local-lvm"`)
- **`grpctunnel`**: Enable/disable gRPC tunnel (0 = disabled, 1 = enabled)

### Storage Selector
The `storage_selector` uses Common Expression Language (CEL) to match Proxmox storage pools:
- `name == "local-lvm"` - Match by exact name
- `type == "lvm"` - Match by storage type
- `name matches "^local.*"` - Match by regex pattern

## Usage

### Apply Machine Classes to Omni

Before deploying clusters, register the machine classes with Omni:

```bash
cd <local_path>/sidero-omni-talos-proxmox/machine-classes

# Apply control plane class
omnictl apply -f control-plane.yaml

# Apply worker class
omnictl apply -f worker.yaml

# Verify classes are registered
omnictl get machineclasses
```

### List Machine Classes

```bash
# List all machine classes
omnictl get machineclasses

# Get details for a specific class
omnictl get machineclass proxmox-control-plane -o yaml
omnictl get machineclass proxmox-worker -o yaml
```

### Update Machine Classes

To modify resource allocations:

1. Edit the YAML file (`control-plane.yaml` or `worker.yaml`)
2. Update the values in the `providerdata` section
3. Reapply to Omni:
   ```bash
   omnictl apply -f control-plane.yaml
   omnictl apply -f worker.yaml
   ```

**Note:** Changes to machine classes affect newly provisioned machines only. Existing machines keep their original specs.

### Delete Machine Classes

```bash
# Delete specific class
omnictl delete machineclass proxmox-control-plane
omnictl delete machineclass proxmox-worker

# Warning: Cannot delete classes that are referenced by active clusters
```

## Prerequisites

1. **Proxmox infrastructure provider running:**
   ```bash
   cd ../proxmox-provider
   docker compose ps
   ```

2. **Proxmox storage pool exists:**
   - The storage specified in `storage_selector` must exist in your Proxmox environment
   - Check Proxmox UI: Datacenter → Storage

3. **Network bridge exists:**
   - The `network_bridge` (e.g., `vmbr0`) must be configured in Proxmox
   - Check Proxmox UI: Node → Network

## Troubleshooting

**Machines fail to provision:**
- Verify storage selector matches an available Proxmox storage pool
- Check Proxmox has sufficient resources (CPU, RAM, disk space)
- Ensure network bridge `vmbr0` exists and is active
- Check Proxmox provider logs: `docker logs omni-infra-provider-proxmox`

**Storage not found:**
- List available storage in Proxmox: `pvesm status`
- Update `storage_selector` to match existing storage name
- Common storage names: `local`, `local-lvm`, `local-zfs`

**Network issues:**
- Verify `network_bridge` matches a Proxmox bridge
- List bridges in Proxmox: check `/etc/network/interfaces` or Proxmox UI
- Default bridge is usually `vmbr0`

## Related Documentation

- [Cluster Templates](../cluster-template/README.md)
- [Proxmox Provider](../proxmox-provider/README.md)
- [Sidero Omni Machine Classes Documentation](https://omni.siderolabs.com/docs/)
- [Proxmox Virtual Environment Documentation](https://pve.proxmox.com/pve-docs/)
