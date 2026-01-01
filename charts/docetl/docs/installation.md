# DocETL Installation Guide

This guide provides step-by-step instructions for installing the DocETL Helm chart.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- PersistentVolume provisioner support in the underlying infrastructure (if persistence is enabled)
- Ingress controller (if Ingress is enabled)

## Architecture

This chart deploys DocETL with the following architecture:
- **Backend**: Python FastAPI application (port 8000)
- **Frontend**: Next.js application (port 3000)
- **Persistent Storage**: PersistentVolumeClaim for /docetl-data
- **Ingress**: Optional Ingress for external access

Both frontend and backend use the same Docker image with different startup commands.

## Installation Methods

### From Helm Repository (Recommended)

```bash
# Add the repository
helm repo add extreme_structure https://x-tructure.github.io/helm-charts
helm repo update

# Install with default values
helm install docetl extreme_structure/docetl

# Install with custom values
helm install docetl extreme_structure/docetl -f my-values.yaml

# Install in a specific namespace
helm install docetl extreme_structure/docetl \
  -f my-values.yaml \
  --namespace docetl \
  --create-namespace
```

### From Source

First, build the DocETL Docker image and push it to your registry:

```bash
# Clone the DocETL repository
git clone https://github.com/ucbepic/docetl.git
cd docetl

# Build the image
docker build -t your-registry/docetl:latest .

# Push to your registry
docker push your-registry/docetl:latest
```

Then install the chart:

```bash
# Clone the Helm chart repository
git clone https://github.com/X-tructure/helm-charts.git
cd helm-charts

# Install with default values
helm install docetl ./charts/docetl

# Install with custom values
helm install docetl ./charts/docetl -f my-values.yaml
```

## Configuration

### Quick Start Configuration

Create a custom values file or use the provided examples:

```bash
# For basic setup
cp charts/docetl/examples/values-basic.yaml my-values.yaml

# For development
cp charts/docetl/examples/values-dev.yaml my-values.yaml

# For production
cp charts/docetl/examples/values-prod.yaml my-values.yaml
```

Edit `my-values.yaml` and update the following required fields:

1. **Image Repository** (REQUIRED):
   ```yaml
   image:
     repository: your-registry/docetl  # e.g., ghcr.io/your-org/docetl
     tag: latest
   ```

2. **OpenAI API Key** (REQUIRED):
   ```yaml
   backend:
     openaiApiKey: "your-api-key-here"
   ```

   **WARNING**: Never commit actual API keys to version control! See the [Security Guide](security.md) for best practices.

3. **Ingress Configuration** (Optional):
   ```yaml
   ingress:
     enabled: true
     hosts:
       - host: docetl.example.com
         paths:
           - path: /
             pathType: Prefix
   ```

For detailed configuration options, see the [Configuration Reference](configuration.md).

## Installation Examples

### Basic Development Setup

```bash
helm install docetl extreme_structure/docetl \
  --set image.repository=myregistry/docetl \
  --set image.tag=dev \
  --set backend.openaiApiKey=sk-your-dev-key \
  --set ingress.hosts[0].host=docetl-dev.example.com
```

### Production Setup with TLS

```bash
helm install docetl extreme_structure/docetl \
  -f examples/values-prod.yaml \
  --set image.repository=myregistry/docetl \
  --set image.tag=v1.0.0 \
  --set backend.openaiApiKey=sk-your-prod-key \
  --set ingress.hosts[0].host=docetl.example.com \
  --set ingress.tls.enabled=true
```

### Using External Secret Management

For production deployments, use external secret management instead of setting secrets in values files:

```bash
# Create a Kubernetes secret manually
kubectl create secret generic docetl-secret \
  --from-literal=openai-api-key=your-actual-key \
  --namespace docetl

# Install without setting the secret in values
helm install docetl extreme_structure/docetl \
  -f my-values.yaml \
  --set backend.openaiApiKey=placeholder
```

See the [Security Guide](security.md) for more secret management options.

## Upgrading

To upgrade an existing installation:

```bash
# Upgrade with custom values
helm upgrade docetl extreme_structure/docetl -f my-values.yaml

# Upgrade and force recreation of pods
helm upgrade docetl extreme_structure/docetl -f my-values.yaml --force
```

For version-specific upgrade instructions, see the [Upgrade Guide](upgrade.md).

## Uninstalling

To remove the DocETL installation:

```bash
helm uninstall docetl

# If installed in a specific namespace
helm uninstall docetl --namespace docetl
```

**Note**: PersistentVolumeClaims are not deleted automatically. To delete them:

```bash
kubectl delete pvc -l app.kubernetes.io/instance=docetl
```

## Accessing the Application

### Via Ingress

If Ingress is enabled, access the application at:
```
http://docetl.example.com  # or https:// if TLS is enabled
```

### Via Port Forwarding

If Ingress is not enabled:

```bash
# Forward frontend port
kubectl port-forward svc/docetl-frontend 3000:3000

# Forward backend port (in another terminal)
kubectl port-forward svc/docetl-backend 8000:8000

# Access at:
# Frontend: http://localhost:3000
# Backend: http://localhost:8000
```

### Via LoadBalancer

If you set service type to LoadBalancer:

```bash
# Get external IP
kubectl get svc docetl-frontend
kubectl get svc docetl-backend
```

## Verification

After installation, verify that all pods are running:

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/instance=docetl

# View backend logs
kubectl logs -l app.kubernetes.io/component=backend -f

# View frontend logs
kubectl logs -l app.kubernetes.io/component=frontend -f

# Run Helm tests
helm test docetl
```

## Next Steps

- Review [Configuration Options](configuration.md) for customization
- Set up [Security Best Practices](security.md) for production
- Check [Troubleshooting Guide](troubleshooting.md) if you encounter issues
- Learn about [Automated Updates](auto-update.md) for tracking new releases

## Support

- Documentation: https://ucbepic.github.io/docetl
- Website: https://docetl.org
- Issues: https://github.com/ucbepic/docetl/issues
