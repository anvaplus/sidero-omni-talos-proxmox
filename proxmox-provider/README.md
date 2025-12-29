# Proxmox Infrastructure Provider

This folder contains the Omni Proxmox infrastructure provider configuration and container orchestration for running the provider locally or in an environment that supports Docker Compose.

## Purpose

The Proxmox infrastructure provider connects Sidero Omni to your Proxmox Virtual Environment cluster, enabling Omni to:
- Automatically provision VMs for Kubernetes nodes
- Manage VM lifecycle (create, delete, resize)
- Integrate with Proxmox storage and networking
- Support machine classes for different node types

## Files

- **`docker-compose.yaml`**: Compose stack for running the provider container
- **`.env.example`**: Example environment variables (copy to `.env`)
- **`config.yaml.example`**: Example Proxmox connection configuration (copy to `config.yaml`)
- **`config.yaml`**: Local provider configuration (not committed to git)
- **`.env`**: Local environment variables (not committed to git)

## Quickstart

### 1. Copy Example Files

```bash
cd <local_path>/sidero-omni-talos-proxmox/proxmox-provider
cp .env.example .env
cp config.yaml.example config.yaml
```

### 2. Configure Environment Variables

Edit `.env` and set:

```bash
# Your Omni instance URL (include trailing slash if needed)
OMNI_API_ENDPOINT=https://omni.yourdomain.com/

# Infrastructure Provider Key from Omni UI
OMNI_INFRA_PROVIDER_KEY=your-infra-provider-key-here
```

### 3. Configure Proxmox Connection

Edit `config.yaml` and set:

```yaml
proxmox:
  # Proxmox API endpoint
  url: "https://192.168.1.100:8006/api2/json"
  
  # Skip SSL verification for self-signed certs
  insecureSkipVerify: true
  
  # Proxmox API token credentials
  tokenID: "user@realm!tokenid"
  tokenSecret: "your-token-secret-here"
```

### 4. Secure Configuration Files

Restrict permissions to protect credentials:

```bash
chmod 600 .env config.yaml
```

### 5. Start the Provider

```bash
docker compose up -d
```

### 6. Verify Provider is Running

```bash
# Check container status
docker compose ps

# View logs
docker compose logs -f

# Check provider is registered in Omni
omnictl get infraproviders
```

## Obtaining Credentials

### Omni Infrastructure Provider Key

1. Open the Omni web UI
2. Navigate to: **Settings → Infrastructure Providers**
3. Click **Create Provider**
4. Give it a name (e.g., "Proxmox Provider")
5. Copy the generated **Infrastructure Provider Key**
6. Paste it into `.env` as `OMNI_INFRA_PROVIDER_KEY`

**Important:** This is an Infrastructure Provider Key, NOT a service account key.

### Proxmox API Token

1. Log into the Proxmox web UI
2. Navigate to: **Datacenter → Permissions → API Tokens**
3. Click **Add**
4. Configure:
   - **User**: Select or create a user (e.g., `root@pam`)
   - **Token ID**: Give it a name (e.g., `omni`)
   - **Privilege Separation**: Uncheck (or ensure user has required permissions)
5. Click **Add** and copy both:
   - **Token ID**: Format is `user@realm!tokenid` (e.g., `root@pam!omni`)
   - **Token Secret**: Long random string
6. Update `config.yaml`:
   ```yaml
   tokenID: "root@pam!omni"
   tokenSecret: "the-copied-secret"
   ```

### Required Proxmox Permissions

The API token user needs these permissions:
- **VM.Allocate**: Create VMs
- **VM.Config.***: Configure VMs
- **VM.Console**: Access console
- **Datastore.AllocateSpace**: Use storage
- **SDN.Use**: Use network bridges

For simplicity, you can use the root user or create a dedicated user with these permissions.

## Configuration Options

### Environment Variables (`.env`)

| Variable | Description | Example |
|----------|-------------|---------|
| `OMNI_API_ENDPOINT` | Omni API base URL (HTTPS required) | `https://omni.example.com/` |
| `OMNI_INFRA_PROVIDER_KEY` | Infrastructure provider key from Omni | `<generated-key>` |

### Proxmox Configuration (`config.yaml`)

| Field | Description | Example |
|-------|-------------|---------|
| `url` | Proxmox API endpoint | `https://192.168.1.100:8006/api2/json` |
| `insecureSkipVerify` | Skip TLS certificate verification | `true` (dev), `false` (prod) |
| `tokenID` | Proxmox API token ID | `root@pam!omni` |
| `tokenSecret` | Proxmox API token secret | `<secret-value>` |

## Management Commands

### View Logs

```bash
# Follow logs in real-time
docker compose logs -f

# View last 100 lines
docker compose logs --tail 100
```

### Restart Provider

```bash
docker compose restart
```

### Stop Provider

```bash
docker compose down
```

### Update Provider Image

```bash
# Pull latest image
docker compose pull

# Restart with new image
docker compose up -d
```

### Check Container Status

```bash
docker compose ps
docker inspect omni-infra-provider-proxmox
```

## Next Steps

After the provider is running:

1. **Create Machine Classes**:
   ```bash
   cd ../machine-classes
   omnictl apply -f control-plane.yaml
   omnictl apply -f worker.yaml
   ```

2. **Deploy a Cluster**:
   ```bash
   cd ../cluster-template
   omnictl cluster template sync -f k8s-dev-static-ip.yaml
   ```

3. **Monitor Provisioning**:
   ```bash
   omnictl get machines
   omnictl get clusters
   ```

## Related Documentation

- [Machine Classes](../machine-classes/README.md) - Define VM resource specifications
- [Cluster Templates](../cluster-template/README.md) - Deploy Kubernetes clusters
- [Sidero Omni Documentation](https://omni.siderolabs.com/docs/)
- [Proxmox API Documentation](https://pve.proxmox.com/pve-docs/)
- [GitHub: omni-infra-provider-proxmox](https://github.com/siderolabs/omni-infra-provider-proxmox)
