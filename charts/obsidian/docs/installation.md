# Installation Guide

This guide provides detailed instructions for installing the Obsidian Helm chart in various scenarios.

## Prerequisites

Before installing, ensure you have:

- A Kubernetes cluster (1.19+) with kubectl access
- Helm 3.2.0 or later installed
- Sufficient cluster resources (minimum 500m CPU, 512Mi RAM)
- A storage provisioner for PersistentVolumeClaims (if using persistence)

## Installation Methods

### Method 1: Using Helm Repository (Recommended)

```bash
# Add the Helm repository
helm repo add extreme_structure https://x-tructure.github.io/helm-charts

# Update your local Helm chart repository cache
helm repo update

# Install the chart with default values
helm install my-obsidian extreme_structure/obsidian

# Install with custom values
helm install my-obsidian extreme_structure/obsidian \
  --set persistence.size=20Gi \
  --set auth.enabled=true \
  --set auth.username=admin \
  --set auth.password=secure-password
```

### Method 2: Using Git Repository

```bash
# Clone the repository
git clone https://github.com/X-tructure/helm-charts.git
cd helm-charts

# Install from local chart
helm install my-obsidian ./charts/obsidian

# Or use an example configuration
helm install my-obsidian ./charts/obsidian -f ./charts/obsidian/examples/values-production.yaml
```

### Method 3: Using a Custom Values File

```bash
# Create your custom values file
cat > my-values.yaml << 'YAML'
persistence:
  enabled: true
  size: 30Gi
  storageClass: "fast-ssd"

auth:
  enabled: true
  username: "myuser"
  password: "mypassword"

resources:
  requests:
    cpu: 1000m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 2Gi
YAML

# Install with your custom values
helm install my-obsidian obsidian/obsidian -f my-values.yaml
```

## Installation in Different Namespaces

```bash
# Create a dedicated namespace
kubectl create namespace obsidian

# Install in the namespace
helm install my-obsidian obsidian/obsidian -n obsidian

# Install and create namespace if it doesn't exist
helm install my-obsidian obsidian/obsidian -n obsidian --create-namespace
```

## Post-Installation

### Accessing the Application

After installation, get access instructions:

```bash
# Get the application URL
helm status my-obsidian -n obsidian

# Follow the instructions in the output
```

#### Port Forwarding (for testing)

```bash
# Forward HTTPS port (recommended)
kubectl port-forward svc/my-obsidian-obsidian 3001:3001 -n obsidian

# Access at https://localhost:3001
```

#### Using NodePort

```bash
# Change service type to NodePort
helm upgrade my-obsidian obsidian/obsidian \
  --set service.type=NodePort \
  -n obsidian

# Get the NodePort
kubectl get svc my-obsidian-obsidian -n obsidian
```

#### Using LoadBalancer (cloud environments)

```bash
# Change service type to LoadBalancer
helm upgrade my-obsidian obsidian/obsidian \
  --set service.type=LoadBalancer \
  -n obsidian

# Get the external IP (may take a few minutes)
kubectl get svc my-obsidian-obsidian -n obsidian -w
```

### Verifying the Installation

```bash
# Check pod status
kubectl get pods -n obsidian -l app.kubernetes.io/name=obsidian

# Check pod logs
kubectl logs -n obsidian -l app.kubernetes.io/name=obsidian

# Check PVC status
kubectl get pvc -n obsidian

# Run Helm tests
helm test my-obsidian -n obsidian
```

## Common Installation Scenarios

### Scenario 1: Basic Development Setup

```bash
helm install dev-obsidian obsidian/obsidian \
  --set persistence.size=5Gi \
  --set resources.requests.cpu=250m \
  --set resources.requests.memory=256Mi
```

### Scenario 2: Production with Ingress

```bash
helm install prod-obsidian obsidian/obsidian \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set ingress.hosts[0].host=obsidian.company.com \
  --set ingress.annotations."cert-manager\.io/cluster-issuer"=letsencrypt-prod \
  --set ingress.tls[0].secretName=obsidian-tls \
  --set ingress.tls[0].hosts[0]=obsidian.company.com \
  --set auth.enabled=true \
  --set auth.username=admin \
  --set persistence.size=50Gi \
  --set persistence.storageClass=fast-ssd
```

### Scenario 3: Using Existing PVC

```bash
# If you already have a PVC with data
helm install my-obsidian obsidian/obsidian \
  --set persistence.existingClaim=my-existing-pvc
```

### Scenario 4: GPU Acceleration

```bash
helm install my-obsidian obsidian/obsidian \
  --set extraEnv[0].name=DRINODE \
  --set extraEnv[0].value=/dev/dri/renderD128 \
  --set nodeSelector.gpu=nvidia
```

## Troubleshooting Installation

### Pod Not Starting

```bash
# Check pod status and events
kubectl describe pod -n obsidian -l app.kubernetes.io/name=obsidian

# Check logs
kubectl logs -n obsidian -l app.kubernetes.io/name=obsidian
```

### PVC Pending

```bash
# Check PVC status
kubectl describe pvc -n obsidian

# Ensure you have a storage provisioner
kubectl get storageclass
```

### Insufficient Resources

```bash
# Check node resources
kubectl top nodes

# Reduce resource requests if needed
helm upgrade my-obsidian obsidian/obsidian \
  --set resources.requests.cpu=250m \
  --set resources.requests.memory=256Mi \
  -n obsidian
```

## Next Steps

- Configure [authentication](configuration.md#authentication)
- Setup [Ingress](configuration.md#ingress-configuration)
- Review [security best practices](configuration.md#security)
- See [troubleshooting guide](troubleshooting.md) for common issues
