# Installation Guide

This guide provides step-by-step instructions for installing the Docling-Serve Helm chart.

## Prerequisites

Before installing the chart, ensure you have:

1. **Kubernetes Cluster** (version 1.19+)
   ```bash
   kubectl version --short
   ```

2. **Helm** (version 3.2.0+)
   ```bash
   helm version --short
   ```

3. **Sufficient Resources**
   - Minimum: 250m CPU, 1Gi RAM per pod
   - Recommended for production: 500m CPU, 2Gi RAM

4. **For GPU Deployment** (optional)
   - NVIDIA GPU Operator installed
   - CUDA drivers ≥550.54.14
   - Available GPU nodes

## Installation Methods

### Method 1: From Helm Repository (Recommended)

```bash
# Add the Helm repository
helm repo add thinking-and-coding https://thinking-and-coding.github.io/helm-charts

# Update repository information
helm repo update

# Install the chart
helm install my-docling-serve thinking-and-coding/docling-serve

# Or with custom release name
helm install docling thinking-and-coding/docling-serve --namespace docling --create-namespace
```

### Method 2: From Git Repository

```bash
# Clone the repository
git clone https://github.com/thinking-and-coding/helm-charts.git
cd helm-charts

# Install the chart
helm install my-docling-serve ./charts/docling-serve

# Or specify a namespace
helm install my-docling-serve ./charts/docling-serve \
  --namespace docling --create-namespace
```

## Basic Installation Scenarios

### Scenario 1: Quick Testing (Minimal Resources)

For development and testing environments:

```bash
helm install my-docling-serve ./charts/docling-serve \
  -f charts/docling-serve/examples/values-basic.yaml
```

This configuration:
- Reduces resource requirements
- Disables model loading at boot (faster startup)
- Uses memory-backed scratch storage

### Scenario 2: Production Deployment

For production environments:

```bash
helm install my-docling-serve ./charts/docling-serve \
  -f charts/docling-serve/examples/values-production.yaml \
  --set apiKey.value=YOUR_SECURE_API_KEY
```

This configuration:
- Enables API key authentication
- Uses persistent storage
- Configures production resource limits
- Includes monitoring annotations

### Scenario 3: External Access with Ingress

For exposing the service externally:

```bash
# Update the ingress host in the example file first
# Then install:
helm install my-docling-serve ./charts/docling-serve \
  -f charts/docling-serve/examples/values-with-ingress.yaml \
  --set ingress.hosts[0].host=docling.yourdomain.com \
  --set apiKey.enabled=true
```

### Scenario 4: GPU Acceleration

For GPU-accelerated document processing:

```bash
helm install my-docling-serve ./charts/docling-serve \
  -f charts/docling-serve/examples/values-gpu.yaml
```

Requirements:
- GPU nodes with NVIDIA drivers
- NVIDIA GPU Operator installed
- CUDA 12.6 or 12.8 compatible GPUs

## Verification

### 1. Check Pod Status

```bash
# Wait for the pod to be ready
kubectl get pods -l app.kubernetes.io/name=docling-serve

# Watch pod startup (may take 1-2 minutes for model loading)
kubectl get pods -l app.kubernetes.io/name=docling-serve --watch
```

Expected output:
```
NAME                             READY   STATUS    RESTARTS   AGE
my-docling-serve-xxxxxxxxx-xxxxx   1/1     Running   0          2m
```

### 2. Check Service

```bash
kubectl get svc -l app.kubernetes.io/name=docling-serve
```

### 3. View Logs

```bash
# Get pod name
POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=docling-serve -o jsonpath='{.items[0].metadata.name}')

# View logs
kubectl logs $POD_NAME

# Follow logs
kubectl logs $POD_NAME -f
```

Look for:
- Model loading messages (if `loadModelsAtBoot: true`)
- Uvicorn startup message
- No error messages

### 4. Test API Access

```bash
# Port-forward to local machine
kubectl port-forward svc/my-docling-serve 5001:5001

# In another terminal, test the API
curl http://localhost:5001/version

# Or open API documentation in browser
open http://localhost:5001/docs
```

### 5. Run Helm Test

```bash
helm test my-docling-serve
```

## Post-Installation Configuration

### Retrieve Auto-Generated API Key

If you enabled API key authentication without providing a value:

```bash
kubectl get secret my-docling-serve-api-key \
  -o jsonpath='{.data.api-key}' | base64 -d
echo
```

### Test with API Key

```bash
API_KEY=$(kubectl get secret my-docling-serve-api-key -o jsonpath='{.data.api-key}' | base64 -d)

curl -H "X-API-Key: $API_KEY" http://localhost:5001/version
```

### Enable Persistent Storage

If you need to enable persistence after installation:

```bash
helm upgrade my-docling-serve ./charts/docling-serve \
  --set scratch.persistent=true \
  --set scratch.pvc.size=50Gi \
  --reuse-values
```

## Customization

### Create Custom Values File

```bash
# Create custom values
cat > my-values.yaml <<EOF
replicaCount: 1

resources:
  requests:
    cpu: 500m
    memory: 2Gi
  limits:
    cpu: 2
    memory: 8Gi

env:
  uvicornWorkers: "2"
  engineLocalNumWorkers: "4"
  loadModelsAtBoot: "true"

apiKey:
  enabled: true
  value: "my-secure-key-123"

scratch:
  persistent: true
  pvc:
    size: 20Gi
EOF

# Install with custom values
helm install my-docling-serve ./charts/docling-serve -f my-values.yaml
```

### Override Specific Values

```bash
helm install my-docling-serve ./charts/docling-serve \
  --set replicaCount=1 \
  --set resources.requests.memory=2Gi \
  --set env.enableUI=true \
  --set apiKey.enabled=true
```

## Troubleshooting Installation

### Pod Not Starting

```bash
# Describe the pod to see events
kubectl describe pod -l app.kubernetes.io/name=docling-serve

# Check pod logs
kubectl logs -l app.kubernetes.io/name=docling-serve
```

Common issues:
- Insufficient resources: Increase memory/CPU limits
- Image pull errors: Check imagePullSecrets
- Model loading timeout: Increase startupProbe failureThreshold

### Service Not Accessible

```bash
# Check service endpoints
kubectl get endpoints -l app.kubernetes.io/name=docling-serve

# Test from within cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://my-docling-serve:5001/version
```

See [Troubleshooting Guide](troubleshooting.md) for more details.

## Uninstallation

```bash
# Uninstall the release
helm uninstall my-docling-serve

# Optionally delete PVCs (if persistent storage was used)
kubectl delete pvc -l app.kubernetes.io/name=docling-serve

# Delete namespace if created
kubectl delete namespace docling
```

## Next Steps

- [Configuration Reference](configuration.md) - Explore all configuration options
- [Troubleshooting](troubleshooting.md) - Common issues and solutions
- [Upgrade Guide](upgrade.md) - Version upgrade instructions

## Support

For installation issues:
1. Check the [Troubleshooting Guide](troubleshooting.md)
2. Review pod logs and events
3. Report issues on [GitHub](https://github.com/thinking-and-coding/helm-charts/issues)
