# Configuration Reference

Complete reference for all configuration parameters in the Docling-Serve Helm chart.

## Table of Contents

- [Image Configuration](#image-configuration)
- [Deployment Configuration](#deployment-configuration)
- [Service Account](#service-account)
- [Security Contexts](#security-contexts)
- [Service Configuration](#service-configuration)
- [Ingress Configuration](#ingress-configuration)
- [Resource Management](#resource-management)
- [Storage Configuration](#storage-configuration)
- [Health Probes](#health-probes)
- [Environment Variables](#environment-variables)
- [API Authentication](#api-authentication)
- [Node Scheduling](#node-scheduling)

## Image Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `image.repository` | string | `ghcr.io/docling-project/docling-serve-cpu` | Container image repository |
| `image.pullPolicy` | string | `IfNotPresent` | Image pull policy |
| `image.tag` | string | `""` | Image tag (overrides appVersion) |
| `imagePullSecrets` | list | `[]` | Image pull secrets for private registries |

**GPU Images:**
- CPU: `ghcr.io/docling-project/docling-serve-cpu` (4.4GB)
- CUDA 12.6: `ghcr.io/docling-project/docling-serve-cu126` (10GB)
- CUDA 12.8: `ghcr.io/docling-project/docling-serve-cu128` (11GB)

## Deployment Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `replicaCount` | int | `1` | Number of replicas |
| `strategy.type` | string | `RollingUpdate` | Deployment strategy |
| `strategy.rollingUpdate.maxUnavailable` | int | `0` | Max unavailable pods during update |
| `strategy.rollingUpdate.maxSurge` | int | `1` | Max surge pods during update |
| `nameOverride` | string | `""` | Override chart name |
| `fullnameOverride` | string | `""` | Override full name |

## Service Account

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `serviceAccount.create` | bool | `false` | Create service account |
| `serviceAccount.annotations` | object | `{}` | Service account annotations |
| `serviceAccount.name` | string | `""` | Service account name |

## Security Contexts

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `podSecurityContext.fsGroup` | int | `1001` | File system group ID |
| `podSecurityContext.runAsNonRoot` | bool | `true` | Run as non-root user |
| `podSecurityContext.runAsUser` | int | `1001` | User ID to run as |
| `podSecurityContext.runAsGroup` | int | `1001` | Group ID to run as |
| `securityContext.capabilities.drop` | list | `[ALL]` | Dropped capabilities |
| `securityContext.readOnlyRootFilesystem` | bool | `false` | Read-only root filesystem |
| `securityContext.allowPrivilegeEscalation` | bool | `false` | Allow privilege escalation |
| `podAnnotations` | object | `{}` | Pod annotations |

## Service Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `service.type` | string | `ClusterIP` | Service type (ClusterIP, NodePort, LoadBalancer) |
| `service.port` | int | `5001` | Service port |
| `service.name` | string | `http` | Port name |
| `service.annotations` | object | `{}` | Service annotations |

## Ingress Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ingress.enabled` | bool | `false` | Enable ingress |
| `ingress.className` | string | `""` | Ingress class name |
| `ingress.annotations` | object | `{}` | Ingress annotations |
| `ingress.hosts` | list | See values.yaml | Ingress hosts configuration |
| `ingress.tls` | list | `[]` | TLS configuration |

**Common Annotations:**
```yaml
ingress:
  annotations:
    # Allow large file uploads
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    # Increase timeouts for long conversions
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    # TLS configuration
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

## Resource Management

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `resources.requests.cpu` | string | `250m` | CPU request |
| `resources.requests.memory` | string | `1Gi` | Memory request |
| `resources.limits.cpu` | string | `1` | CPU limit |
| `resources.limits.memory` | string | `4Gi` | Memory limit |

**GPU Resources:**
```yaml
resources:
  requests:
    nvidia.com/gpu: 1
  limits:
    nvidia.com/gpu: 1
```

**Recommended Values:**
- **Basic/Testing:** 100m CPU, 512Mi RAM
- **Production (CPU):** 500m CPU, 2Gi RAM
- **Production (GPU):** 500m CPU, 4Gi RAM, 1 GPU

## Storage Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `scratch.persistent` | bool | `false` | Enable persistent volume |
| `scratch.emptyDir.medium` | string | `""` | emptyDir medium (Memory or "") |
| `scratch.emptyDir.sizeLimit` | string | `10Gi` | emptyDir size limit |
| `scratch.pvc.storageClass` | string | `""` | Storage class for PVC |
| `scratch.pvc.accessMode` | string | `ReadWriteOnce` | PVC access mode |
| `scratch.pvc.size` | string | `10Gi` | PVC size |
| `scratch.pvc.annotations` | object | `{}` | PVC annotations |

**Storage Options:**
- **Ephemeral (default):** `persistent: false` - uses emptyDir
- **Memory-backed:** `emptyDir.medium: "Memory"` - faster, counts against memory
- **Persistent:** `persistent: true` - creates PVC

## Model Persistence Configuration

Model persistence allows downloading and storing AI models on a PersistentVolumeClaim instead of using pre-baked container images. An **init container** downloads models before the main application starts.

### Benefits

- **Smaller container images**: Reduces image pull time and storage requirements
- **Selective model downloads**: Only download the models you need
- **Model sharing**: Multiple pods share the same model storage
- **Easier updates**: Update models without rebuilding container images
- **Guaranteed readiness**: Init container ensures models are available before app starts
- **Idempotent downloads**: Models only downloaded once (skipped on subsequent restarts)

### How It Works

1. PVC is created for model storage
2. Pod starts with init container
3. Init container checks if models exist (marker file)
4. If models missing or `forceRedownload=true`, downloads models to PVC
5. Init container completes
6. Main container starts and loads models from PVC

### Configuration Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `models.enabled` | bool | `false` | Enable model persistence |
| `models.download` | list | `[layout, ...]` | List of models to download |
| `models.artifactsPath` | string | `/modelcache` | Path where models are stored |
| `models.forceRedownload` | bool | `false` | Force re-download on pod startup |
| `models.pvc.storageClass` | string | `""` | Storage class for PVC |
| `models.pvc.accessMode` | string | `ReadWriteOnce` | PVC access mode |
| `models.pvc.size` | string | `15Gi` | PVC storage size |
| `models.pvc.existingClaim` | string | `""` | Use existing PVC |
| `models.initContainer.resources` | object | See values.yaml | Init container resources |
| `models.initContainer.image.repository` | string | `""` | Override init container image |
| `models.initContainer.image.tag` | string | `""` | Override init container tag |
| `models.initContainer.image.pullPolicy` | string | `""` | Override pull policy |

### Available Models

- **layout**: Document layout analysis (~2-3Gi)
- **tableformer**: Table structure recognition (~1-2Gi)
- **code_formula**: Code and formula detection (~1Gi)
- **picture_classifier**: Image classification (~1Gi)
- **smolvlm**: Small vision-language model (~5-7Gi)
- **granite_vision**: IBM Granite vision model (~5-7Gi)
- **easyocr**: OCR engine (~500Mi)

### Startup Time Expectations

| Scenario | Init Container Time | App Startup Time | Total |
|----------|---------------------|------------------|-------|
| Models present (cached) | 5-10s | 30-60s | ~1 min |
| First startup (basic 4 models) | 5-10 min | 30-60s | ~6-10 min |
| First startup (all 7 models) | 10-20 min | 30-60s | ~11-21 min |
| Force redownload | 5-20 min | 30-60s | ~6-21 min |

### Monitoring Init Container

```bash
# Watch init container logs during download
kubectl logs -f <pod-name> -c model-downloader

# Check if init container completed
kubectl get pod <pod-name> -o jsonpath='{.status.initContainerStatuses[*].state}'

# Debug init container failure
kubectl describe pod <pod-name>
```

### Force Re-Download Models

To update models or recover from corrupted downloads:

```bash
# Set forceRedownload flag
helm upgrade my-docling charts/docling-serve/ \
  --set models.enabled=true \
  --set models.forceRedownload=true \
  --reuse-values

# After successful update, disable the flag
helm upgrade my-docling charts/docling-serve/ \
  --set models.forceRedownload=false \
  --reuse-values
```

Or delete and recreate the pod:

```bash
kubectl delete pod <pod-name>
# Kubernetes recreates pod, init container re-runs
```

### Example Configurations

See example files for complete configurations:
- `examples/values-with-models.yaml` - Basic model persistence
- `examples/values-with-all-models.yaml` - All models
- `examples/values-gpu-with-models.yaml` - GPU with models


## Health Probes

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `startupProbe.httpGet.path` | string | `/health` | Startup probe path |
| `startupProbe.httpGet.port` | string | `http` | Startup probe port |
| `startupProbe.initialDelaySeconds` | int | `5` | Initial delay |
| `startupProbe.periodSeconds` | int | `10` | Period between checks |
| `startupProbe.failureThreshold` | int | `20` | Failure threshold (200s total) |
| `readinessProbe.httpGet.path` | string | `/health` | Readiness probe path |
| `readinessProbe.initialDelaySeconds` | int | `10` | Initial delay |
| `readinessProbe.periodSeconds` | int | `5` | Period between checks |
| `livenessProbe.httpGet.path` | string | `/health` | Liveness probe path |
| `livenessProbe.initialDelaySeconds` | int | `30` | Initial delay |
| `livenessProbe.periodSeconds` | int | `10` | Period between checks |

**Note:** The default probe configurations are optimized for model-loading scenarios
(`loadModelsAtBoot: "true"`). If you disable model loading or use constrained resources,
consider using scenario-specific probe configurations from the example files:
- `values-basic.yaml`: Optimized for fast startup without model loading (65s window)
- `values-gpu.yaml`: Extended window for GPU initialization overhead (315s window)

See [troubleshooting.md](troubleshooting.md#understanding-probe-configuration-for-different-scenarios) for guidance on choosing the right probe configuration.

## Environment Variables

### Server Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `env.uvicornHost` | string | `0.0.0.0` | Uvicorn bind address |
| `env.uvicornPort` | string | `5001` | Uvicorn port |
| `env.uvicornWorkers` | string | `1` | Number of Uvicorn workers |

### Application Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `env.enableUI` | string | `false` | Enable Gradio UI at /ui |
| `env.loadModelsAtBoot` | string | `true` | Pre-load models at startup |
| `env.engineKind` | string | `local` | Engine type (local/rq) |
| `env.engineLocalNumWorkers` | string | `2` | Local engine worker threads |
| `env.scratchPath` | string | `/tmp/docling` | Scratch directory path |
| `env.singleUseResults` | string | `true` | Clean up results after retrieval |

### Performance Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `env.numThreads` | string | `4` | Processing threads per request |
| `env.device` | string | `cpu` | Device type (cpu/cuda) |

### Extra Environment Variables

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `extraEnv` | list | `[]` | Additional environment variables |

**Example:**
```yaml
extraEnv:
  - name: DOCLING_SERVE_MAX_NUM_PAGES
    value: "1000"
  - name: DOCLING_SERVE_MAX_FILE_SIZE
    value: "104857600"  # 100MB
```

## API Authentication

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `apiKey.enabled` | bool | `false` | Enable API key authentication |
| `apiKey.value` | string | `""` | API key value (empty = auto-generate) |
| `apiKey.existingSecret` | string | `""` | Use existing secret |
| `apiKey.existingSecretKey` | string | `api-key` | Secret key name |

**Usage:**
```bash
# Auto-generate key
helm install my-docling --set apiKey.enabled=true

# Provide specific key
helm install my-docling --set apiKey.enabled=true --set apiKey.value=mysecretkey

# Use existing secret
kubectl create secret generic my-api-key --from-literal=api-key=mysecretkey
helm install my-docling --set apiKey.enabled=true --set apiKey.existingSecret=my-api-key
```

## Node Scheduling

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `nodeSelector` | object | `{}` | Node selector labels |
| `tolerations` | list | `[]` | Pod tolerations |
| `affinity` | object | `{}` | Pod affinity rules |

**Example - GPU Nodes:**
```yaml
nodeSelector:
  nvidia.com/gpu.present: "true"

tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"
```

## Complete Example

```yaml
# Full configuration example
image:
  repository: ghcr.io/docling-project/docling-serve-cpu
  tag: "v1.9.0"

replicaCount: 2

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
  numThreads: "8"
  loadModelsAtBoot: "true"
  enableUI: "false"

apiKey:
  enabled: true
  value: "my-secure-api-key"

scratch:
  persistent: true
  pvc:
    size: 50Gi
    storageClass: "fast-ssd"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
  hosts:
    - host: docling.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: docling-tls
      hosts:
        - docling.example.com

podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "5001"
```

## Environment Variable Reference

For complete list of Docling-Serve environment variables, see:
https://github.com/docling-project/docling-serve#configuration

## Support

For configuration questions:
- [Installation Guide](installation.md)
- [Troubleshooting](troubleshooting.md)
- [GitHub Issues](https://github.com/X-tructure/helm-charts/issues)
