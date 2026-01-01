# Obsidian Helm Chart

Helm chart for deploying [LinuxServer.io Obsidian](https://github.com/linuxserver/docker-obsidian) on Kubernetes with web-based GUI using Selkies.

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/obsidian)](https://artifacthub.io/packages/search?repo=obsidian)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## Features

- 🚀 Easy deployment with sensible defaults
- 🔒 Optional authentication and security settings
- 💾 Persistent storage with PVC support
- 🌐 Ingress support with TLS
- 🎛️ Comprehensive configuration options
- 📊 Resource limits and health checks
- 🖥️ GPU acceleration support (VAAPI, NVIDIA)
- 🔄 Automated deployment strategy for stateful applications
- 🤖 Automated version tracking and updates

## Quick Start

### Using Helm Repository (Recommended)

```bash
# Add the Helm repository
helm repo add extreme_structure https://x-tructure.github.io/helm-charts
helm repo update

# Install the chart
helm install my-obsidian extreme_structure/obsidian

# Access the application
kubectl port-forward svc/my-obsidian-obsidian 3001:3001
# Visit https://localhost:3001
```

### Using Git Repository

```bash
# Clone the repository
git clone https://github.com/X-tructure/helm-charts.git
cd helm-charts

# Install the chart
helm install my-obsidian ./charts/obsidian

# Or with custom values
helm install my-obsidian ./charts/obsidian -f charts/obsidian/examples/values-production.yaml
```

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- PV provisioner support in the underlying infrastructure (for persistence)
- Sufficient cluster resources (500m CPU, 512Mi RAM minimum)

## Configuration

See [Configuration Reference](docs/configuration.md) for detailed configuration options.

### Common Configurations

#### Enable Authentication

```bash
helm install my-obsidian extreme_structure/obsidian \
  --set auth.enabled=true \
  --set auth.username=admin \
  --set auth.password=your-secure-password
```

#### With Ingress and TLS

```bash
helm install my-obsidian extreme_structure/obsidian \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=obsidian.example.com \
  --set ingress.className=nginx \
  --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod
```

#### Custom Resources

```bash
helm install my-obsidian extreme_structure/obsidian \
  --set resources.limits.cpu=4000m \
  --set resources.limits.memory=4Gi \
  --set persistence.size=50Gi \
  --set persistence.storageClass=fast-ssd
```

#### GPU Acceleration

```bash
helm install my-obsidian extreme_structure/obsidian \
  --set extraEnv[0].name=DRINODE \
  --set extraEnv[0].value=/dev/dri/renderD128
```

## Examples

Check the [examples/](examples/) directory for complete configuration examples:

- [Basic Setup](examples/values-basic.yaml) - Minimal configuration with defaults
- [Production Setup](examples/values-production.yaml) - Production-ready with auth and resources
- [Ingress with TLS](examples/values-with-ingress.yaml) - External access with SSL
- [GPU Acceleration](examples/values-gpu.yaml) - Hardware acceleration for better performance
- [Fixed Resolution](examples/values-1080p.yaml) - 1920x1080 display resolution

## Documentation

- [Installation Guide](docs/installation.md) - Step-by-step installation instructions
- [Configuration Reference](docs/configuration.md) - Complete parameter reference
- [Troubleshooting](docs/troubleshooting.md) - Common issues and solutions
- [Upgrade Guide](docs/upgrade.md) - Version upgrade instructions
- [Auto-Update Setup](docs/auto-update.md) - Automated version tracking and releases

## Key Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas (should be 1 for stateful apps) | `1` |
| `image.repository` | Container image repository | `lscr.io/linuxserver/obsidian` |
| `image.tag` | Container image tag | `latest` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | Storage size | `10Gi` |
| `auth.enabled` | Enable HTTP basic authentication | `false` |
| `ingress.enabled` | Enable Ingress resource | `false` |
| `resources.requests.cpu` | CPU resource requests | `500m` |
| `resources.requests.memory` | Memory resource requests | `512Mi` |

For a complete list of parameters, see [values.yaml](values.yaml) or the [Configuration Reference](docs/configuration.md).

## Architecture

### LinuxServer.io Container Requirements

**IMPORTANT**: This chart uses the LinuxServer.io Obsidian container which has specific requirements:

- Container MUST start as root (do not set `runAsUser`/`runAsGroup`)
- Uses PUID/PGID environment variables for user mapping
- Container automatically switches to specified user after initialization
- Breaking this pattern causes container initialization failures

### Selkies Web Desktop

- Provides browser-based access to Obsidian desktop application
- Exposes two ports: HTTP (3000) and HTTPS (3001)
- HTTPS is required for full functionality (WebCodecs support)
- Health checks use HTTPS endpoint

### Deployment Strategy

- Uses `Recreate` strategy (not RollingUpdate)
- Required because PVC uses `ReadWriteOnce` access mode
- Only one pod can mount the volume at a time

### Shared Memory Volume

- Required for Electron/Chromium rendering engine
- Mounted at `/dev/shm` with configurable size (default: 1Gi)
- Prevents application crashes and rendering issues

## Upgrading

```bash
# Update Helm repository
helm repo update

# Upgrade to latest version
helm upgrade my-obsidian extreme_structure/obsidian

# Upgrade with custom values
helm upgrade my-obsidian extreme_structure/obsidian -f my-values.yaml
```

See [Upgrade Guide](docs/upgrade.md) for version-specific upgrade instructions.

## Uninstalling

```bash
# Uninstall the release
helm uninstall my-obsidian

# Optionally, delete the PVC (this will delete all your data!)
kubectl delete pvc my-obsidian-obsidian-config
```

## Support

If you encounter any issues or have questions:

- Check the [Troubleshooting Guide](docs/troubleshooting.md)
- Open an issue on [GitHub](https://github.com/X-tructure/helm-charts/issues)
- Refer to the [LinuxServer.io Obsidian documentation](https://docs.linuxserver.io/images/docker-obsidian)

## Contributing

Contributions are welcome! Please see the [Contributing Guide](../../docs/contributing.md) in the repository root.

## License

Apache License 2.0 - see [LICENSE](../../LICENSE) for details.

## Acknowledgments

- Based on [LinuxServer.io Obsidian Docker container](https://github.com/linuxserver/docker-obsidian)
- Uses [Selkies](https://github.com/selkies-project/selkies-gstreamer) for web desktop streaming
- Built on [Kubernetes](https://kubernetes.io/) and [Helm](https://helm.sh/)

## Security

> [!WARNING]
> This container provides privileged access to a desktop environment. Do not expose it to the Internet without proper authentication and security measures.

- HTTPS is **required** for full functionality (WebCodecs support)
- The default self-signed certificate is only suitable for local development
- For production, use a reverse proxy with valid TLS certificates
- Enable authentication (`auth.enabled=true`) or use external authentication via Ingress
- The web interface includes a terminal with sudo access

## Chart Information

- **Chart Version**: See [Chart.yaml](Chart.yaml)
- **Source Code**: https://github.com/X-tructure/helm-charts
- **Upstream Application**: https://obsidian.md
- **Container Image**: https://github.com/linuxserver/docker-obsidian
