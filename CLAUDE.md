# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a multi-chart Helm repository for deploying various applications on Kubernetes. The repository is structured to support multiple independent Helm charts, each with its own versioning, documentation, and examples.

**Current Charts:**
- **Obsidian**: Note-taking application using LinuxServer.io Docker image with Selkies web-based GUI
- **DocETL**: Document processing pipeline with LLM operations (backend + frontend)
- **Docling-Serve**: AI-powered document conversion API service (FastAPI wrapper for Docling)

The repository uses automated CI/CD pipelines for linting, testing, and releasing charts.

## Repository Structure

```
helm-charts/
├── charts/                     # All Helm charts
│   ├── obsidian/              # Obsidian chart
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── README.md
│   │   ├── templates/
│   │   ├── examples/          # Chart-specific examples
│   │   └── docs/              # Chart-specific docs
│   └── [other-charts]/        # Future charts
├── docs/                       # Shared documentation
│   ├── contributing.md
│   ├── ci-cd.md
│   └── repository-structure.md
├── scripts/                    # Utility scripts
└── .github/workflows/          # CI/CD pipelines
    ├── lint-test.yaml         # Generic linting/testing
    ├── release.yaml           # Generic releasing
    └── auto-update-*.yaml     # Chart-specific auto-updates
```

## Common Commands

### Working with Charts

#### Chart Development

```bash
# Lint a specific chart
helm lint charts/<chart-name>/

# Template with default values
helm template test charts/<chart-name>/

# Template with example configurations
helm template test charts/<chart-name>/ -f charts/<chart-name>/examples/values-production.yaml

# Dry-run installation
helm install test charts/<chart-name>/ --dry-run --debug
```

#### Obsidian Chart Specific

```bash
# Lint the Obsidian chart
helm lint charts/obsidian/

# Template with examples
helm template test charts/obsidian/ -f charts/obsidian/examples/values-production.yaml
helm template test charts/obsidian/ -f charts/obsidian/examples/values-with-ingress.yaml

# Install locally for testing
helm install my-obsidian ./charts/obsidian

# Install with custom values
helm install my-obsidian ./charts/obsidian -f charts/obsidian/examples/values-production.yaml
```

#### DocETL Chart Specific

```bash
# Lint the DocETL chart
helm lint charts/docetl/

# Template with examples
helm template test charts/docetl/ -f charts/docetl/examples/values-dev.yaml
helm template test charts/docetl/ -f charts/docetl/examples/values-prod.yaml

# Install locally for testing
helm install my-docetl ./charts/docetl

# Install with production values
helm install my-docetl ./charts/docetl -f charts/docetl/examples/values-prod.yaml
```

#### Docling-Serve Chart Specific

```bash
# Lint the Docling-Serve chart
helm lint charts/docling-serve/

# Template with examples
helm template test charts/docling-serve/ -f charts/docling-serve/examples/values-basic.yaml
helm template test charts/docling-serve/ -f charts/docling-serve/examples/values-production.yaml
helm template test charts/docling-serve/ -f charts/docling-serve/examples/values-gpu.yaml

# Install locally for testing
helm install my-docling ./charts/docling-serve

# Install with production values
helm install my-docling ./charts/docling-serve -f charts/docling-serve/examples/values-production.yaml

# Install with GPU support
helm install my-docling ./charts/docling-serve -f charts/docling-serve/examples/values-gpu.yaml
```

### Chart Testing (CI/CD)

```bash
# Run chart-testing lint (scans all charts)
ct lint --config .github/ct.yaml --all

# Run chart-testing install (requires kind cluster)
ct install --config .github/ct.yaml --all

# Test all charts with their examples
for chart in charts/*/; do
  if [ -f "$chart/Chart.yaml" ]; then
    chart_name=$(basename "$chart")
    echo "Testing $chart_name..."

    # Lint
    helm lint "$chart"

    # Template with default values
    helm template test "$chart"

    # Template with examples if they exist
    if [ -d "$chart/examples" ]; then
      for example in "$chart/examples"/*.yaml; do
        [ -f "$example" ] && helm template test "$chart" -f "$example"
      done
    fi
  fi
done
```

### Release Process

**For all charts:**
1. Update chart version in `charts/<name>/Chart.yaml` (follows SemVer)
2. Update `appVersion` if upgrading application version
3. Commit changes following Conventional Commits:
   - `feat(<chart>):` for new features
   - `fix(<chart>):` for bug fixes
   - `docs(<chart>):` for documentation
   - `refactor:`, `test:`, `chore:` as appropriate
4. Create and push tag: `git tag <chart>-v0.x.x && git push origin <chart>-v0.x.x`
5. GitHub Actions automatically packages and publishes the chart

**Tag naming conventions:**
- Single chart update: `v0.x.x` or `<chart-name>-v0.x.x`
- Multiple charts: Use multiple tags or `<chart-name>-v0.x.x` format (recommended)

### Auto-Update Workflows

The repository includes automated tracking of upstream application releases:

**Obsidian Auto-Update:**
```bash
# Setup auto-update (requires gh CLI)
./scripts/setup-auto-update.sh

# Manually trigger update check
gh workflow run auto-update-obsidian.yaml

# Check workflow status
gh run list --workflow=auto-update-obsidian.yaml
```

**Workflow behavior:**
- Runs daily at 00:00 UTC (configurable via cron)
- Checks for new LinuxServer.io Docker image releases
- Compares with current `appVersion` in charts/obsidian/Chart.yaml
- If update found:
  - Sends notification (GitHub Issue or email)
  - Auto-increments chart patch version
  - Either creates PR for review OR auto-commits and tags (based on `AUTO_MERGE` setting)

**Configuration:** See `charts/obsidian/docs/auto-update.md` for detailed setup instructions.

**Adding auto-update for new charts:**
1. Copy `.github/workflows/auto-update-obsidian.yaml`
2. Rename to `auto-update-<chart-name>.yaml`
3. Update chart paths and upstream source
4. Configure workflow variables in GitHub repository settings

## Architecture

### Multi-Chart Repository Design

**Key Principles:**
1. **Independence**: Each chart is self-contained with own docs and examples
2. **Consistency**: Shared CI/CD pipelines detect and test all charts automatically
3. **Flexibility**: Charts can have chart-specific workflows (like auto-update)
4. **Scalability**: Easy to add new charts without modifying existing infrastructure

**When to add a new chart:**
- Create `charts/<name>/` directory
- Follow the structure documented in `docs/repository-structure.md`
- CI/CD automatically detects and processes it
- No workflow changes needed!

### Obsidian Chart Architecture

The Obsidian chart has specific architectural decisions documented in detail:

#### Key Architectural Patterns

1. **Stateful Application Pattern**:
   - Uses `Recreate` deployment strategy (not RollingUpdate)
   - Required because PVC uses `ReadWriteOnce` access mode
   - Only one pod can mount the volume at a time

2. **LinuxServer.io Container Requirements**:
   - Container MUST start as root (do not set `runAsUser`/`runAsGroup`)
   - Uses PUID/PGID environment variables for user mapping
   - Container automatically switches to specified user after initialization
   - Breaking this pattern causes container init failures

3. **Selkies Web Desktop**:
   - Provides browser-based access to Obsidian desktop application
   - Exposes two ports: HTTP (3000) and HTTPS (3001)
   - HTTPS required for full functionality (WebCodecs support)
   - Health checks use HTTPS endpoint

4. **Shared Memory Volume**:
   - Required for Electron/Chromium rendering engine
   - Mounted at `/dev/shm` with configurable size (default: 1Gi)
   - Prevents application crashes and rendering issues

### Docling-Serve Chart Architecture

The Docling-Serve chart is designed for simplicity and production readiness:

#### Key Architectural Patterns

1. **CPU-First Design**:
   - Defaults to CPU-optimized image (`docling-serve-cpu`)
   - GPU support available via examples (`values-gpu.yaml`)
   - Container images: 4.4GB (CPU) to 11GB (GPU variants)

2. **Simple Local Engine**:
   - Uses built-in thread-based processing by default
   - No external dependencies (Redis/RQ) required
   - Configurable worker threads and processing parallelism

3. **FastAPI Service**:
   - RESTful API for document conversion
   - OpenAPI/Swagger documentation at `/docs`
   - Optional Gradio UI at `/ui` (disabled by default)
   - Port 5001 (HTTP)

4. **Security Model**:
   - Runs as non-root user (UID/GID 1001)
   - Drops all capabilities
   - Optional API key authentication with auto-generation
   - Cannot use readOnlyRootFilesystem (needs `/tmp/docling` writes)

5. **Storage Strategy**:
   - Ephemeral by default (emptyDir)
   - Models pre-baked into container image
   - Optional persistent volume for scratch space
   - Memory-backed emptyDir option for performance

6. **Health Probes**:
   - Startup probe: 200s window for model loading
   - Readiness probe: Fast check (5s period)
   - Liveness probe: Conservative (10s period)
   - All probes use `GET /` endpoint

7. **GPU Support** (Optional):
   - CUDA 12.6 (`docling-serve-cu126`) or 12.8 (`docling-serve-cu128`)
   - Requires NVIDIA GPU Operator
   - Node selector for GPU nodes
   - Increased memory allocation (4-16Gi)

### Template Helpers (_helpers.tpl)

Each chart should define helper functions for consistency. Example (Obsidian):
- `<chart>.name`: Chart name with override support
- `<chart>.fullname`: Fully qualified app name (63 char limit)
- `<chart>.labels`: Standard Kubernetes labels
- `<chart>.selectorLabels`: Pod selector labels
- `<chart>.serviceAccountName`: Service account name resolution

### Values File Organization

Standard structure (see individual charts for specifics):
1. Basic deployment settings (replicas, image)
2. Service account configuration
3. Security contexts (pod and container level)
4. Service and Ingress configuration
5. Resource limits and requests
6. Persistence configuration
7. Probe configurations (liveness, readiness, startup)
8. Environment variables

## Important Considerations

### Adding New Charts

1. **Create chart structure**:
   ```bash
   helm create charts/<name>
   mkdir -p charts/<name>/examples
   mkdir -p charts/<name>/docs
   ```

2. **Required files**:
   - `Chart.yaml`: Metadata
   - `values.yaml`: Default configuration
   - `README.md`: Chart documentation
   - `templates/`: Kubernetes manifests
   - `examples/values-production.yaml`: At least one example

3. **Documentation**:
   - README.md in chart root
   - Optional detailed docs in `docs/` subdirectory
   - Examples in `examples/` subdirectory

4. **Testing**:
   - CI/CD automatically detects and tests new charts
   - Ensure `helm lint` passes
   - All examples must template successfully

### Security Contexts

- Each chart may have different security requirements
- **Obsidian**: Never add `runAsUser` or `runAsGroup` to container security context (LinuxServer.io requirement)
- **Docling-Serve**: Runs as non-root user 1001, drops all capabilities, cannot use readOnlyRootFilesystem
- Document security requirements in chart-specific README

### Persistence

- Use consistent parameter naming across charts:
  - `persistence.enabled` (Obsidian, DocETL)
  - `scratch.persistent` (Docling-Serve - different pattern for ephemeral-first design)
  - `persistence.size` / `scratch.pvc.size`
  - `persistence.storageClass` / `scratch.pvc.storageClass`
  - `persistence.existingClaim` (Obsidian)
- Default storage class when `storageClass` is empty string
- Use `storageClass: "-"` to disable dynamic provisioning
- **Docling-Serve**: Also supports memory-backed emptyDir via `scratch.emptyDir.medium: "Memory"`

### Health Probes

- Configure appropriate probes for each application
- Consider startup time for `startupProbe`
- Document probe requirements in chart README

### Example Files

- Keep examples in `charts/<name>/examples/` directory
- Provide examples for common scenarios:
  - `values-basic.yaml`: Minimal setup
  - `values-production.yaml`: Production-ready
  - `values-with-ingress.yaml`: External access
  - Application-specific examples as needed

### CI/CD

GitHub Actions workflows (`.github/workflows/`):
- `lint-test.yaml`: Runs on PR/push, performs linting and chart-testing for **all charts**
- `release.yaml`: Automatically packages and publishes charts on tag push
- `auto-update-<chart>.yaml`: Chart-specific automated version tracking
- Configuration in `.github/ct.yaml` (validates all charts)

**Important:**
- Generic workflows automatically detect all charts
- No need to modify workflows when adding new charts
- Chart-specific workflows (like auto-update) are optional

## Chart Version Requirements

### General (varies by chart)

- Kubernetes: 1.19+ (minimum, check individual charts)
- Helm: 3.2.0+
- PV provisioner (if persistence is used)

### Obsidian Specific

- Kubernetes: 1.19+
- Helm: 3.2.0+
- Minimum resources: 500m CPU, 512Mi RAM
- PV provisioner required for persistence

## Adding Documentation

### Chart-Specific Documentation

Place in `charts/<chart-name>/docs/`:
- `configuration.md`: Complete parameter reference
- `installation.md`: Step-by-step guide
- `troubleshooting.md`: Common issues
- `upgrade.md`: Version migration guide
- `auto-update.md`: Auto-update setup (if applicable)

### Shared Documentation

Place in `docs/` root:
- `contributing.md`: How to contribute new charts
- `ci-cd.md`: CI/CD pipeline documentation
- `repository-structure.md`: Repository layout guide

## Best Practices

### Chart Development

1. **Follow Helm best practices**: https://helm.sh/docs/chart_best_practices/
2. **Use meaningful names**: Clear, descriptive parameter names
3. **Provide defaults**: Sensible defaults that work out-of-box
4. **Document everything**: Comments in values.yaml and comprehensive README
5. **Test thoroughly**: Lint, template, and install testing
6. **Version properly**: Follow SemVer, bump on any change

### Multi-Chart Considerations

1. **Independence**: Don't create dependencies between charts
2. **Consistency**: Use similar structures and naming conventions
3. **Documentation**: Each chart should be self-documenting
4. **Testing**: Ensure chart works standalone
5. **Versioning**: Each chart has independent version

### Git Workflow

1. **Branches**: Create feature branches for changes
2. **Commits**: Use Conventional Commits format
3. **PRs**: One chart per PR when possible
4. **Tags**: Use `<chart>-v<version>` format for clarity
5. **Reviews**: Test changes locally before submitting

## Migration Notes

This repository was restructured from a single-chart to multi-chart layout:

**Changes:**
- `examples/` → `charts/obsidian/examples/`
- Chart-specific `docs/` → `charts/obsidian/docs/`
- Generic workflows now detect all charts automatically
- Auto-update workflow renamed to `auto-update-obsidian.yaml`

**Backwards Compatibility:**
- Chart name remains `obsidian`
- Chart version continues from 0.1.3
- No breaking changes for users
- Helm repo URL unchanged (GitHub Pages redirect)

## Resources

### Helm Documentation
- [Helm Charts Guide](https://helm.sh/docs/topics/charts/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Chart Template Guide](https://helm.sh/docs/chart_template_guide/)

### Tools
- [chart-testing](https://github.com/helm/chart-testing)
- [chart-releaser](https://github.com/helm/chart-releaser)
- [kind](https://kind.sigs.k8s.io/) - Kubernetes in Docker

### Repository Documentation
- [Contributing Guide](docs/contributing.md)
- [CI/CD Documentation](docs/ci-cd.md)
- [Repository Structure](docs/repository-structure.md)
