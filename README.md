# Helm Charts Repository

A collection of production-ready Helm charts for deploying applications on Kubernetes.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## Available Charts

| Chart | Description | Version | App Version |
|-------|-------------|---------|-------------|
| [obsidian](charts/obsidian/) | Obsidian note-taking app with web GUI | ![Version](https://img.shields.io/badge/dynamic/yaml?url=https://raw.githubusercontent.com/X-tructure/helm-charts/main/charts/obsidian/Chart.yaml&label=version&query=$.version&color=blue) | ![AppVersion](https://img.shields.io/badge/dynamic/yaml?url=https://raw.githubusercontent.com/X-tructure/helm-charts/main/charts/obsidian/Chart.yaml&label=app&query=$.appVersion&color=green) |
| [docetl](charts/docetl/) | Document processing pipeline with LLM operations | ![Version](https://img.shields.io/badge/dynamic/yaml?url=https://raw.githubusercontent.com/X-tructure/helm-charts/main/charts/docetl/Chart.yaml&label=version&query=$.version&color=blue) | ![AppVersion](https://img.shields.io/badge/dynamic/yaml?url=https://raw.githubusercontent.com/X-tructure/helm-charts/main/charts/docetl/Chart.yaml&label=app&query=$.appVersion&color=green) |
| [docling-serve](charts/docling-serve/) | AI-powered document conversion API service | ![Version](https://img.shields.io/badge/dynamic/yaml?url=https://raw.githubusercontent.com/X-tructure/helm-charts/main/charts/docling-serve/Chart.yaml&label=version&query=$.version&color=blue) | ![AppVersion](https://img.shields.io/badge/dynamic/yaml?url=https://raw.githubusercontent.com/X-tructure/helm-charts/main/charts/docling-serve/Chart.yaml&label=app&query=$.appVersion&color=green) |

## Quick Start

### Add the Helm Repository

```bash
helm repo add extreme_structure https://x-tructure.github.io/helm-charts
helm repo update
```

### Install a Chart

```bash
# Install Obsidian chart
helm install my-obsidian extreme_structure/obsidian

# Install with custom values
helm install my-obsidian extreme_structure/obsidian -f my-values.yaml
```

### Browse Charts

Each chart has its own documentation:

- **[Obsidian](charts/obsidian/)**: Web-based Obsidian note-taking application
  ```bash
  helm install my-obsidian extreme_structure/obsidian
  ```

- **[DocETL](charts/docetl/)**: Document processing pipeline with LLM operations
  ```bash
  helm install my-docetl extreme_structure/docetl
  ```

- **[Docling-Serve](charts/docling-serve/)**: AI-powered document conversion API service
  ```bash
  helm install my-docling extreme_structure/docling-serve
  ```

## Repository Structure

```
helm-charts/
├── charts/
│   ├── obsidian/           # Obsidian note-taking app chart
│   │   ├── Chart.yaml      # Chart metadata
│   │   ├── values.yaml     # Default configuration
│   │   ├── README.md       # Chart documentation
│   │   ├── templates/      # Kubernetes manifests
│   │   ├── examples/       # Example configurations
│   │   └── docs/           # Detailed documentation
│   ├── docetl/             # DocETL document processing chart
│   │   ├── Chart.yaml      # Chart metadata
│   │   ├── values.yaml     # Default configuration
│   │   ├── README.md       # Chart documentation
│   │   ├── templates/      # Kubernetes manifests
│   │   ├── examples/       # Example configurations
│   │   └── docs/           # Detailed documentation
│   ├── docling-serve/      # Docling-Serve AI document conversion chart
│   │   ├── Chart.yaml      # Chart metadata
│   │   ├── values.yaml     # Default configuration
│   │   ├── README.md       # Chart documentation
│   │   ├── templates/      # Kubernetes manifests
│   │   ├── examples/       # Example configurations
│   │   └── docs/           # Detailed documentation
│   └── [future-charts]/    # Additional charts
├── docs/                   # Shared documentation
└── .github/workflows/      # CI/CD pipelines
```

For detailed repository structure, see [Repository Structure Guide](docs/repository-structure.md).

## Features

- 🚀 **Production-Ready**: Battle-tested charts with best practices
- 📦 **Easy Installation**: One-command deployment via Helm
- 🔧 **Highly Configurable**: Extensive customization options
- 📖 **Well Documented**: Comprehensive guides and examples
- 🔄 **Automated Updates**: CI/CD pipelines for testing and releasing
- ✅ **Tested**: Automated linting and integration testing
- 🛡️ **Secure**: Security best practices and hardening options

## Documentation

### For Users

- **Installation Guides**: Each chart has its own installation guide
- **Configuration**: See individual chart documentation
- **Examples**: Check `charts/*/examples/` directories
- **Troubleshooting**: Each chart includes a troubleshooting guide

### For Contributors

- [Contributing Guide](docs/contributing.md) - How to contribute new charts
- [CI/CD Documentation](docs/ci-cd.md) - Understanding the pipelines
- [Repository Structure](docs/repository-structure.md) - Layout and organization

## Prerequisites

- **Kubernetes**: 1.19+ (varies by chart)
- **Helm**: 3.2.0+
- **kubectl**: Configured to access your cluster

## Using the Charts

### From Helm Repository (Recommended)

```bash
# Add repository
helm repo add extreme_structure https://x-tructure.github.io/helm-charts

# Search for charts
helm search repo X-tructure

# Install a chart
helm install my-release extreme_structure/<chart-name>

# Upgrade a release
helm upgrade my-release extreme_structure/<chart-name>

# Uninstall a release
helm uninstall my-release
```

### From Source

```bash
# Clone the repository
git clone https://github.com/X-tructure/helm-charts.git
cd helm-charts

# Install a chart from local directory
helm install my-release ./charts/<chart-name>

# With custom values
helm install my-release ./charts/<chart-name> -f my-values.yaml
```

## Chart Highlights

### Obsidian

Deploy Obsidian note-taking application with web-based access.

**Key Features:**
- Web-based GUI via Selkies desktop streaming
- Persistent storage for notes
- Optional authentication
- Ingress support
- GPU acceleration
- Automated updates

**Quick Install:**
```bash
helm install obsidian extreme_structure/obsidian \
  --set auth.enabled=true \
  --set auth.username=admin \
  --set auth.password=secure-password
```

📖 [Full Documentation](charts/obsidian/) | 💡 [Examples](charts/obsidian/examples/)

### DocETL

Deploy document processing pipeline with LLM operations.

**Key Features:**
- Multi-service architecture (backend + frontend)
- Secure API key management (4 best-practice options)
- Persistent document storage
- Production-ready with TLS
- Horizontal scaling support
- RBAC and network policies

**Quick Install:**
```bash
helm install docetl extreme_structure/docetl \
  --set backend.openaiApiKey=your-api-key \
  --set persistence.enabled=true
```

📖 [Full Documentation](charts/docetl/) | 💡 [Examples](charts/docetl/examples/)

### Docling-Serve

Deploy AI-powered document conversion API service.

**Key Features:**
- CPU-optimized defaults with GPU support
- FastAPI with OpenAPI documentation
- Convert PDF/DOCX/PPTX to Markdown/JSON/HTML
- Optional API key authentication
- Ephemeral or persistent storage
- Production-ready configurations

**Quick Install:**
```bash
helm install docling extreme_structure/docling-serve \
  --set apiKey.enabled=true \
  --set scratch.persistent=true
```

📖 [Full Documentation](charts/docling-serve/) | 💡 [Examples](charts/docling-serve/examples/)

## Contributing

We welcome contributions! Whether you want to:

- 🐛 Report bugs
- 💡 Suggest new features
- 📝 Improve documentation
- 🎁 Add new charts
- 🔧 Fix issues

Please read our [Contributing Guide](docs/contributing.md) to get started.

### Adding a New Chart

1. Create chart in `charts/your-chart/`
2. Follow the [chart structure](docs/repository-structure.md)
3. Add documentation and examples
4. Test thoroughly
5. Submit a pull request

The CI/CD pipelines will automatically detect and test your chart!

## Development

### Testing Charts Locally

```bash
# Lint a chart
helm lint charts/<chart-name>/

# Template a chart (dry-run)
helm template test-release charts/<chart-name>/

# Test with example values
helm template test-release charts/<chart-name>/ \
  -f charts/<chart-name>/examples/values-production.yaml

# Install in a test cluster
helm install test-release charts/<chart-name>/ --dry-run --debug
```

### Chart Testing

We use [chart-testing](https://github.com/helm/chart-testing) for validation:

```bash
# Install ct
brew install chart-testing

# Lint all charts
ct lint --config .github/ct.yaml --all

# Test installation (requires kind)
kind create cluster
ct install --config .github/ct.yaml --all
```

## CI/CD

All charts are automatically:

- ✅ **Linted**: Syntax and best practices validation
- 🧪 **Tested**: Integration testing in Kubernetes
- 📦 **Packaged**: Automated chart packaging
- 🚀 **Released**: Published to GitHub Pages

See [CI/CD Documentation](docs/ci-cd.md) for details.

## Versioning

- Charts follow [Semantic Versioning](https://semver.org/)
- Each chart has independent versioning
- `version`: Chart version (e.g., 0.1.0)
- `appVersion`: Application version (e.g., v1.10.6)

## Repository Maintenance

This repository is actively maintained. Updates include:

- 🔄 Application version updates
- 🐛 Bug fixes and improvements
- 📝 Documentation enhancements
- ✨ New features
- 🔒 Security patches

Some charts support automated version tracking via GitHub Actions workflows.

## Support

### Getting Help

- 📖 **Documentation**: Check chart-specific docs
- 🐛 **Issues**: [GitHub Issues](https://github.com/X-tructure/helm-charts/issues)
- 💬 **Discussions**: [GitHub Discussions](https://github.com/X-tructure/helm-charts/discussions)

### Reporting Issues

When reporting issues, please include:

1. Chart name and version
2. Kubernetes version
3. Helm version
4. Error messages and logs
5. Steps to reproduce

## License

Apache License 2.0 - see [LICENSE](LICENSE) for details.

All charts are provided as-is. Individual applications may have their own licenses.

## Acknowledgments

- Built with ❤️ using [Helm](https://helm.sh/)
- Deployed on [Kubernetes](https://kubernetes.io/)
- Charts published via [GitHub Pages](https://pages.github.com/)
- CI/CD powered by [GitHub Actions](https://github.com/features/actions)
- Chart testing via [chart-testing](https://github.com/helm/chart-testing)
- Releases managed by [chart-releaser](https://github.com/helm/chart-releaser)

## Project Status

🟢 **Active**: This repository is actively maintained and accepting contributions.

---

**Need a custom chart?** Check our [Contributing Guide](docs/contributing.md) or open a [feature request](https://github.com/X-tructure/helm-charts/issues/new)!
