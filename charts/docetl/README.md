# DocETL Helm Chart

A production-ready Helm chart for deploying [DocETL](https://docetl.org), a powerful document processing pipeline with LLM operations, on Kubernetes.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Helm](https://img.shields.io/badge/Helm-3.2.0%2B-blue)](https://helm.sh)

## Features

- 🎯 **Multi-Container Pod Architecture**: Unified deployment with backend (FastAPI) and frontend (Next.js) in a single pod
- 🔐 **Enterprise Security**: Multiple API key management options with best practices
- 💾 **Persistent Storage**: Integrated PVC management for document processing data
- 🌐 **Ingress Support**: Optional TLS-enabled external access
- ⚖️ **High Availability**: Horizontal scaling with shared storage support
- 📊 **Production Ready**: Resource limits, health checks, and anti-affinity rules
- 🔄 **Automated Updates**: Optional workflow for tracking upstream releases
- 🚀 **Simplified Networking**: Localhost communication between containers eliminates CORS complexity

## Quick Start

### Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- PersistentVolume provisioner (if persistence enabled)
- Ingress controller (if Ingress enabled)

### Installation

```bash
# Add Helm repository
helm repo add extreme_structure https://x-tructure.github.io/helm-charts
helm repo update

# Install with default values
helm install docetl extreme_structure/docetl

# Install with custom values
helm install docetl extreme_structure/docetl -f my-values.yaml
```

**Important**: Update these required fields in your values file:
1. `image.repository` - Your Docker registry path (e.g., `ghcr.io/your-org/docetl`)
2. `backend.openaiApiKey` - Your OpenAI API key (see [Security Guide](docs/security.md))

For detailed installation instructions, see the [Installation Guide](docs/installation.md).

## Architecture

DocETL v2.0.0 uses a **multi-container pod architecture** where frontend and backend run as separate containers within the same pod:

```
┌─────────────────────────────────────────┐
│           DocETL Pod                    │
│  ┌──────────────┐   ┌──────────────┐   │
│  │  Frontend    │───│   Backend    │   │
│  │  (Next.js)   │   │  (FastAPI)   │   │
│  │  Port 3000   │   │  Port 8000   │   │
│  └──────┬───────┘   └──────┬───────┘   │
│         │     localhost    │           │
│         │ communication    │           │
│         └──────────────────┘           │
│                  │                     │
└──────────────────┼─────────────────────┘
                   │
         ┌─────────▼─────────┐
         │  Persistent       │
         │  Storage          │
         │  /docetl-data     │
         └───────────────────┘
```

**Key Characteristics:**

- **Single Deployment**: Both containers in one pod, sharing network namespace
- **Localhost Communication**: Frontend → Backend via `localhost`, eliminating DNS overhead and CORS
- **Shared Volume**: Both containers access the same persistent storage
- **Unified Scaling**: Frontend and backend scale together as a single unit
- **Single Service**: One service with two ports (3000 for frontend, 8000 for backend)

Both containers use the same Docker image with different startup commands.

### Why Multi-Container Pod?

This architecture provides:
- ✅ Simplified networking (no service discovery)
- ✅ Reduced operational complexity (one deployment vs two)
- ✅ Better resource density
- ✅ Natural fit for ReadWriteOnce storage
- ✅ Aligned lifecycles (components always co-located)

## Configuration

### Basic Configuration

Example `my-values.yaml`:

```yaml
# Docker image (REQUIRED - update to your registry)
image:
  repository: your-registry/docetl
  tag: latest

# OpenAI API Key (REQUIRED - see Security Guide)
secrets:
  openaiApiKey: "your-api-key-here"

# Replica count (both frontend and backend scale together)
replicaCount: 1

# Persistence
persistence:
  enabled: true
  size: 10Gi

# Ingress (optional)
ingress:
  enabled: false
```

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Docker image repository | `docetl` |
| `secrets.openaiApiKey` | OpenAI API key | `""` |
| `replicaCount` | Number of pod replicas (frontend & backend scale together) | `1` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `persistence.size` | PVC size | `10Gi` |
| `ingress.enabled` | Enable Ingress | `true` |
| `service.frontendPort` | Frontend service port | `3000` |
| `service.backendPort` | Backend service port | `8000` |

For complete parameter reference, see the [Configuration Guide](docs/configuration.md).

## Example Configurations

The `examples/` directory contains ready-to-use configurations:

- **[values-dev.yaml](examples/values-dev.yaml)**: Development environment with minimal resources
- **[values-prod.yaml](examples/values-prod.yaml)**: Production-ready with HA, TLS, and security hardening

### Quick Examples

**Development deployment:**

```bash
helm install docetl ./charts/docetl \
  -f examples/values-dev.yaml \
  --set image.repository=myregistry/docetl \
  --set secrets.openaiApiKey=sk-dev-key
```

**Production deployment with TLS:**

```bash
helm install docetl ./charts/docetl \
  -f examples/values-prod.yaml \
  --set image.repository=myregistry/docetl \
  --set secrets.openaiApiKey=sk-prod-key \
  --set ingress.hosts[0].host=docetl.example.com
```

## Security

DocETL chart provides multiple secure methods for API key management:

1. **Command-line override** (Development only)
2. **Separate values file** (Not committed to Git)
3. **External secret management** (Production - Sealed Secrets, Vault, AWS Secrets Manager)
4. **Manual Kubernetes secret** (Simple production)

**⚠️ Never commit API keys to version control!**

For detailed security setup including TLS, RBAC, and network policies, see the [Security Best Practices](docs/security.md).

## Documentation

Comprehensive guides are available in the `docs/` directory:

- **[Installation Guide](docs/installation.md)** - Step-by-step installation and deployment
- **[Configuration Reference](docs/configuration.md)** - Complete parameter documentation
- **[Security Best Practices](docs/security.md)** - API key management, TLS, and hardening
- **[Troubleshooting Guide](docs/troubleshooting.md)** - Common issues and solutions
- **[Upgrade Guide](docs/upgrade.md)** - Version upgrades and rollback procedures
- **[Upgrade to v2.0.0](docs/upgrade-2.0.md)** - Migration guide from v1.x to v2.0.0 (multi-container architecture)
- **[Auto-Update Guide](docs/auto-update.md)** - Automated release tracking setup

## Accessing the Application

### Via Ingress (Recommended for Production)

```
https://docetl.example.com
```

### Via Port Forwarding (Development)

**Frontend:**
```bash
kubectl port-forward svc/docetl 3000:3000
# Access at http://localhost:3000
```

**Backend:**
```bash
kubectl port-forward svc/docetl 8000:8000
# Access at http://localhost:8000/health
```

### Via LoadBalancer

```bash
kubectl get svc docetl -o wide
# Use EXTERNAL-IP with port 3000 for frontend, 8000 for backend
```

## Upgrading

### Standard Upgrades

To upgrade an existing installation:

```bash
# Standard upgrade
helm upgrade docetl extreme_structure/docetl -f my-values.yaml

# Upgrade to specific version
helm upgrade docetl extreme_structure/docetl --version 2.0.0 -f my-values.yaml

# Upgrade with automatic rollback on failure
helm upgrade docetl extreme_structure/docetl -f my-values.yaml --atomic
```

### ⚠️ Upgrading from v1.x to v2.0.0

Version 2.0.0 introduces a **breaking architectural change** from separate deployments to multi-container pods. This is a major version bump requiring manual migration.

**Key Changes:**
- Single deployment with two containers (was two separate deployments)
- Single service with two ports (was two separate services)
- Unified scaling (`replicaCount` instead of separate `backend.replicaCount` and `frontend.replicaCount`)
- Localhost communication between containers (was Kubernetes Service DNS)

**Migration Required:** See the [v2.0.0 Upgrade Guide](docs/upgrade-2.0.md) for complete migration instructions, including:
- Values file transformation
- Data backup procedures
- Step-by-step migration process
- Rollback instructions

For other version-specific migration steps, see the [Upgrade Guide](docs/upgrade.md).

## Uninstalling

```bash
helm uninstall docetl

# Delete persistent data (optional)
kubectl delete pvc -l app.kubernetes.io/instance=docetl
```

## Troubleshooting

Common issues and solutions:

| Issue | Quick Fix |
|-------|-----------|
| Pods not starting | Check resources: `kubectl describe pod <pod-name>` |
| API key errors | Verify secret: `kubectl get secret docetl-secret` |
| Backend connection fails | Check service: `kubectl get svc docetl` |
| PVC not binding | Check StorageClass: `kubectl get pvc` |
| Frontend can't reach backend | Verify localhost config in ConfigMap: `kubectl get cm docetl-config -o yaml` |

For detailed troubleshooting, see the [Troubleshooting Guide](docs/troubleshooting.md).

## Resource Requirements

**Note:** In v2.0.0, each pod runs both frontend and backend containers. Total pod resources are the sum of both containers.

### Minimum (Development)

Per pod (2 containers):
- **Backend**: 250m CPU, 512Mi memory
- **Frontend**: 100m CPU, 256Mi memory
- **Total per pod**: 350m CPU, 768Mi memory
- **Storage**: 5Gi

### Recommended (Production)

Per pod (2 containers):
- **Backend**: 1000m CPU, 2Gi memory
- **Frontend**: 500m CPU, 1Gi memory
- **Total per pod**: 1500m CPU, 3Gi memory
- **Storage**: 50Gi

### Scaling Example

With `replicaCount: 3`:
- Total cluster resources: 4.5 CPU, 9Gi memory (3 pods × resources per pod)
- Shared storage: 50Gi (ReadWriteMany required)

## High Availability

For production deployments with multiple replicas:

```yaml
# Both frontend and backend scale together
replicaCount: 3

# Pod anti-affinity to spread across nodes
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - docetl
          topologyKey: kubernetes.io/hostname

# ReadWriteMany storage for multi-replica access
persistence:
  accessMode: ReadWriteMany
  storageClassName: "nfs-client"  # Must support RWX
  size: 50Gi
```

**Important:** Multi-replica deployments require ReadWriteMany (RWX) storage. Compatible storage classes include:
- NFS-based: `nfs-client`, `nfs-subdir-external-provisioner`
- CephFS: `cephfs`
- AWS EFS: `efs-sc`
- Azure Files: `azurefile`
- Google Filestore: `filestore`

## Building Docker Image

Before installing the chart, build and push the DocETL Docker image:

```bash
# Clone DocETL repository
git clone https://github.com/ucbepic/docetl.git
cd docetl

# Build image
docker build -t your-registry/docetl:latest .

# Push to registry
docker push your-registry/docetl:latest
```

Then update your `values.yaml`:

```yaml
image:
  repository: your-registry/docetl
  tag: latest
```

## Testing

Run Helm tests to verify deployment:

```bash
helm test docetl

# View test logs
kubectl logs docetl-test-connection
```

## Development

Lint and validate the chart:

```bash
# Lint
helm lint ./charts/docetl

# Template with dry-run
helm template test ./charts/docetl -f examples/values-dev.yaml

# Test installation
helm install test ./charts/docetl --dry-run --debug
```

## Support and Links

- **DocETL Documentation**: https://ucbepic.github.io/docetl
- **DocETL Website**: https://docetl.org
- **DocETL Repository**: https://github.com/ucbepic/docetl
- **Chart Issues**: https://github.com/X-tructure/helm-charts/issues
- **Helm Chart Repository**: https://x-tructure.github.io/helm-charts

## Contributing

Contributions are welcome! Please:

1. Read the [Contributing Guide](../../docs/contributing.md)
2. Check existing [issues](https://github.com/X-tructure/helm-charts/issues)
3. Submit pull requests with clear descriptions

## License

Apache License 2.0 - See [LICENSE](../../LICENSE) for details.

DocETL application is licensed separately by its maintainers at [ucbepic/docetl](https://github.com/ucbepic/docetl).

---

**Need help?** Check the [documentation](docs/) or open an [issue](https://github.com/X-tructure/helm-charts/issues)!
