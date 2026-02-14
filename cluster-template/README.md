# Cluster Templates

This directory contains Omni cluster template definitions for deploying Talos Kubernetes clusters on Proxmox via Sidero Omni.

## Available Templates

### Development Clusters

**`k8s-dev-static-ip.yaml`** (Recommended for dev)
- Cluster name: `k8s-dev`
- 1 control plane node with static IP (10.20.10.11/16)
- 3 worker nodes with static IPs (10.20.10.15-17/16)
- Custom hostnames: `k8s-dev-cp-1`, `k8s-dev-worker-1/2/3`
- Network: Gateway 10.20.0.1, DNS 10.20.0.2, NTP pool.ntp.org
- Interface: ens18

**`k8s-dev-dhcp.yaml`**
- Cluster name: `k8s-dev-dhcp`
- 1 control plane node
- 3 worker nodes
- Uses DHCP for IP assignment (gateway provided by DHCP)
- Configured NTP (pool.ntp.org)
- Hostnames: DHCP/Default (static hostnames removed for Talos 1.12+ compatibility)
- Simpler configuration, good for testing

### Production Clusters

**`k8s-prod.yaml`** (HA, Mixed Networking)
- Cluster name: `k8s-prod`
- 3 control plane nodes (HA setup) using DHCP
- 3 worker nodes with static IPs (10.20.20.15-17/16)
- Custom hostnames: `k8s-worker-1/2/3`
- Network: Gateway 10.20.0.1, DNS 10.20.0.2, NTP pool.ntp.org
- Interface: ens18
- **Note**: Control planes use DHCP due to Omni limitation with `size: 3`. For static control plane IPs, configure DHCP reservations on your network.

## Network Configuration

All templates use:
- **Machine classes**: `proxmox-control-plane` and `proxmox-worker` (defined in `../machine-classes/`)
- **System extensions**:
  - siderolabs/iscsi-tools (iSCSI storage support)
  - siderolabs/nfsd (NFS support)
  - siderolabs/qemu-guest-agent (Proxmox integration)
  - siderolabs/util-linux-tools (System utilities)

### Static IP Configuration

The static IP template assigns individual IPs by creating separate Workers definitions for each node:
- Each worker gets `size: 1` with unique network config
- This is required because Omni doesn't support `machineSelector` with `machineIndex`

**To add more workers with static IPs**, duplicate a Workers section and update:
1. `name`: worker-4, worker-5, etc.
2. `hostname`: k8s-worker-4, k8s-worker-5, etc.
3. `addresses`: Next available IP in your range

## Usage

### Deploy a Cluster

From the root of the `sidero-omni-talos-proxmox` repository:

```bash
# Sync template to Omni (creates/updates cluster definition)
omnictl cluster template sync -v -f cluster-template/k8s-dev-static-ip.yaml

# Or for DHCP version
omnictl cluster template sync -v -f cluster-template/k8s-dev-dhcp.yaml

# Production cluster
omnictl cluster template sync -v -f cluster-template/k8s-prod.yaml
```

### Delete a Cluster

From the root of the `sidero-omni-talos-proxmox` repository:

```bash
# Delete cluster and all resources using the template file
omnictl cluster delete -v -f cluster-template/k8s-dev-static-ip.yaml

# Or delete by name
omnictl cluster delete k8s-dev
omnictl cluster delete k8s-prod
```

### Monitor Cluster Creation

```bash
# Watch cluster status
omnictl get clusters

# Watch machines being provisioned
omnictl get machines

# Get cluster details
omnictl get cluster k8s-dev -o yaml
```

## Prerequisites

Before deploying clusters, ensure the following commands are run from the root of the `sidero-omni-talos-proxmox` repository:

1. **Machine classes are created**:
   ```bash
   omnictl apply -f machine-classes/control-plane.yaml
   omnictl apply -f machine-classes/worker.yaml
   ```

2. **Proxmox provider is running**:
   ```bash
   docker compose -f proxmox-provider/docker-compose.yaml ps
   ```

3. **Network configuration matches your environment**:
   - Update IP addresses, gateway, DNS in templates if needed
   - Verify `ens18` is the correct network interface name
   - Ensure IPs don't conflict with DHCP ranges

## Template Structure

```yaml
---
kind: Cluster           # Cluster metadata
name: cluster-name
kubernetes:
  version: v1.34.2      # K8s version
talos:
  version: v1.11.5      # Talos OS version

---
kind: ControlPlane      # Control plane definition
machineClass:
  name: machine-class-name
  size: 1               # Number of control plane nodes
systemExtensions: [...]
patches: [...]          # Network config, extensions, etc.

---
kind: Workers           # Worker definition (can have multiple)
name: worker-name
machineClass:
  name: machine-class-name
  size: 1
systemExtensions: [...]
patches: [...]
```


## Notes

- **Static IP limitation**: Each worker with a unique IP requires a separate Workers definition due to Omni's template limitations.
- **Scaling**: To scale workers with static IPs, manually add Workers sections. For dynamic scaling, use DHCP templates.
- **DHCP reservations**: For production with static-like IPs but easier scaling, configure DHCP reservations on your network and use DHCP templates.
- **Hostname persistence**: Hostnames set via patches persist across reboots and updates.

## Related Documentation

- [Sidero Omni Documentation](https://omni.siderolabs.com/docs/)
- [Talos Linux Documentation](https://www.talos.dev/latest/)
- [Machine Classes](../machine-classes/README.md)
- [Proxmox Provider](../proxmox-provider/README.md)
