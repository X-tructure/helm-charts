# Docling-Serve Helm Chart

Helm chart for deploying [Docling-Serve](https://github.com/docling-project/docling-serve) - API service wrapper for AI-powered document conversion.

## Features

- Easy deployment with CPU-optimized defaults
- Optional GPU acceleration (CUDA 12.6/12.8)
- API-only focus (Gradio UI optional)
- Flexible storage (ephemeral or persistent)
- Optional API key authentication
- FastAPI with OpenAPI documentation
- Health checks and resource management
- Production-ready configurations

## What is Docling-Serve?

Docling-Serve provides a FastAPI wrapper around [Docling](https://github.com/docling-project/docling), an AI toolkit for document conversion. It converts:

- **Input Formats:** PDF, DOCX, PPTX, HTML, Images
- **Output Formats:** Markdown, JSON, HTML, Plain Text

Key capabilities:
- Advanced OCR with multiple engines (EasyOCR, Tesseract, RapidOCR)
- Table structure extraction
- Image extraction and description
- Synchronous and asynchronous processing
- WebSocket support for real-time progress

## Quick Start

### Using Helm Repository

```bash
# Add the Helm repository
helm repo add thinking-and-coding https://thinking-and-coding.github.io/helm-charts
helm repo update

# Install with CPU defaults
helm install my-docling-serve thinking-and-coding/docling-serve

# Access the API
kubectl port-forward svc/my-docling-serve 5001:5001
# Visit http://localhost:5001/docs for API documentation
```

### Using Git Repository

```bash
git clone https://github.com/thinking-and-coding/helm-charts.git
cd helm-charts

# Install with default CPU configuration
helm install my-docling-serve ./charts/docling-serve

# Or with production settings
helm install my-docling-serve ./charts/docling-serve \
  -f charts/docling-serve/examples/values-production.yaml
```

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- Minimum resources: 250m CPU, 1Gi RAM
- For GPU: NVIDIA GPU Operator and appropriate CUDA drivers (≥550.54.14)

## Configuration

See [Configuration Reference](docs/configuration.md) for detailed options.

### Common Configurations

#### Enable API Key Authentication

```bash
helm install my-docling-serve ./charts/docling-serve \
  --set apiKey.enabled=true \
  --set apiKey.value=your-secret-key
```

#### With Ingress and TLS

```bash
helm install my-docling-serve ./charts/docling-serve \
  -f charts/docling-serve/examples/values-with-ingress.yaml
```

#### GPU Acceleration

```bash
helm install my-docling-serve ./charts/docling-serve \
  -f charts/docling-serve/examples/values-gpu.yaml
```

#### Enable Gradio UI

```bash
helm install my-docling-serve ./charts/docling-serve \
  --set env.enableUI=true
```

#### Persistent Storage

```bash
helm install my-docling-serve ./charts/docling-serve \
  --set scratch.persistent=true \
  --set scratch.pvc.size=50Gi
```

## Architecture Decisions

This chart is designed with the following priorities:

- **CPU-First:** Optimized for CPU by default (GPU available via examples)
- **Simple:** Local engine only, no Redis/RQ complexity
- **API-Focused:** Gradio UI disabled by default
- **Ephemeral Storage:** Uses emptyDir by default (opt-in persistence)

These defaults work for most users. See examples/ for alternative configurations.

## API Usage

Once deployed, interact with the API:

```bash
# Get API documentation
curl http://localhost:5001/docs

# Check version
curl http://localhost:5001/version

# Convert a PDF (synchronous)
curl -X POST http://localhost:5001/v1/convert/file \
  -H "Content-Type: multipart/form-data" \
  -F "file=@document.pdf" \
  -F "output_format=markdown"

# With API key authentication
curl -X POST http://localhost:5001/v1/convert/file \
  -H "X-API-Key: your-api-key" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@document.pdf"
```

## Documentation

- [Installation Guide](docs/installation.md) - Step-by-step installation
- [Configuration Reference](docs/configuration.md) - Complete parameter list
- [Troubleshooting](docs/troubleshooting.md) - Common issues and solutions
- [Upgrade Guide](docs/upgrade.md) - Version upgrade instructions

## Examples

All example configurations are in `examples/`:

- **values-basic.yaml** - Minimal testing setup (reduced resources, fast startup)
- **values-production.yaml** - Production-ready (authentication, persistent storage)
- **values-with-ingress.yaml** - External access (TLS, Ingress)
- **values-gpu.yaml** - GPU acceleration (CUDA 12.6/12.8)

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `replicaCount` | int | `1` | Number of replicas |
| `image.repository` | string | `ghcr.io/docling-project/docling-serve-cpu` | Container image repository |
| `image.tag` | string | `""` | Image tag (defaults to appVersion) |
| `service.type` | string | `ClusterIP` | Kubernetes service type |
| `service.port` | int | `5001` | Service port |
| `resources.requests.cpu` | string | `250m` | CPU request |
| `resources.requests.memory` | string | `1Gi` | Memory request |
| `resources.limits.cpu` | string | `1` | CPU limit |
| `resources.limits.memory` | string | `4Gi` | Memory limit |
| `env.device` | string | `cpu` | Device type (cpu/cuda) |
| `env.enableUI` | string | `false` | Enable Gradio UI |
| `apiKey.enabled` | bool | `false` | Enable API key authentication |
| `scratch.persistent` | bool | `false` | Use persistent storage |

See [Configuration Reference](docs/configuration.md) for complete list.

## Upgrading

```bash
# Update repository
helm repo update

# Upgrade to latest version
helm upgrade my-docling-serve thinking-and-coding/docling-serve

# Or with custom values
helm upgrade my-docling-serve thinking-and-coding/docling-serve \
  -f my-values.yaml
```

## Uninstalling

```bash
helm uninstall my-docling-serve
```

## Contributing

Contributions are welcome! Please see the main repository [CONTRIBUTING.md](../../docs/contributing.md).

## License

This Helm chart is licensed under Apache 2.0.

Docling and Docling-Serve are licensed under MIT License.

## Links

- [Docling-Serve GitHub](https://github.com/docling-project/docling-serve)
- [Docling GitHub](https://github.com/docling-project/docling)
- [Chart Repository](https://github.com/thinking-and-coding/helm-charts)
- [Helm Documentation](https://helm.sh/docs/)

## Support

- Report issues: [GitHub Issues](https://github.com/thinking-and-coding/helm-charts/issues)
- Upstream project: [Docling-Serve Issues](https://github.com/docling-project/docling-serve/issues)
