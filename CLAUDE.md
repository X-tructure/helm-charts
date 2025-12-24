# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Helm chart repository for deploying Obsidian (note-taking application) on Kubernetes using the LinuxServer.io Docker image with Selkies web-based GUI. The chart enables running Obsidian as a web application accessible via browser.

## Common Commands

### Development and Testing

```bash
# Lint the chart
helm lint charts/obsidian/

# Template with default values
helm template test charts/obsidian/

# Template with example configurations
helm template test charts/obsidian/ -f examples/values-production.yaml
helm template test charts/obsidian/ -f examples/values-with-ingress.yaml

# Dry-run installation
helm install test charts/obsidian/ --dry-run --debug

# Install locally for testing
helm install my-obsidian ./charts/obsidian

# Install with custom values
helm install my-obsidian ./charts/obsidian -f examples/values-production.yaml
```

### Chart Testing (CI/CD)

```bash
# Run chart-testing lint
ct lint --config .github/ct.yaml --all

# Run chart-testing install (requires kind cluster)
ct install --config .github/ct.yaml --all
```

### Release Process

1. Update version in `charts/obsidian/Chart.yaml` (follows SemVer)
2. Update `appVersion` if upgrading Obsidian version
3. Commit changes following Conventional Commits:
   - `feat:` for new features
   - `fix:` for bug fixes
   - `docs:` for documentation
   - `refactor:`, `test:`, `chore:` as appropriate
4. Create and push tag: `git tag v0.x.x && git push origin v0.x.x`
5. GitHub Actions automatically packages and publishes the chart

## Architecture

### Chart Structure

- **Chart Location**: `charts/obsidian/`
- **Templates**: Standard Kubernetes resources (Deployment, Service, Ingress, PVC, ServiceAccount)
- **Helper Functions**: Defined in `_helpers.tpl` for consistent naming and labels

### Key Architectural Decisions

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

5. **Environment Variable Organization**:
   - Core variables: PUID, PGID, TZ, TITLE
   - Auth variables: CUSTOM_USER, PASSWORD
   - Selkies variables: Prefixed with `SELKIES_*`
   - Extra variables: `extraEnv` for flexibility

### Template Helpers (_helpers.tpl)

Key helper functions that maintain consistency:
- `obsidian.name`: Chart name with override support
- `obsidian.fullname`: Fully qualified app name (63 char limit)
- `obsidian.labels`: Standard Kubernetes labels
- `obsidian.selectorLabels`: Pod selector labels
- `obsidian.serviceAccountName`: Service account name resolution
- `obsidian.pvcName`: PVC name with existingClaim support
- `obsidian.image`: Image reference construction

### Values File Organization

The `values.yaml` follows this structure:
1. Basic deployment settings (replicas, image)
2. Service account configuration
3. Security contexts (pod and container level)
4. Service and Ingress configuration
5. Resource limits and requests
6. Persistence configuration
7. Probe configurations (liveness, readiness, startup)
8. Environment variables (env, auth, selkies, extraEnv)

## Important Considerations

### Security Context
Never add `runAsUser` or `runAsGroup` to the container security context. LinuxServer.io containers require root startup and handle user switching internally via PUID/PGID.

### Persistence
- Default storage class is used when `storageClass` is empty string
- Use `storageClass: "-"` to disable dynamic provisioning
- `existingClaim` allows reusing existing PVCs for data migration

### Health Probes
- All probes use HTTPS endpoint (port 3001) with `scheme: HTTPS`
- Startup probe allows 200 seconds for initial container startup
- Important for slow first boot when downloading/initializing Obsidian

### Example Files
The `examples/` directory contains complete value files for common scenarios:
- `values-basic.yaml`: Minimal setup
- `values-production.yaml`: Production-ready with auth and resources
- `values-with-ingress.yaml`: External access with TLS
- `values-gpu.yaml`: Hardware acceleration via VAAPI/NVIDIA
- `values-1080p.yaml`: Fixed 1920x1080 resolution

### CI/CD
GitHub Actions workflows (`.github/workflows/`):
- `lint-test.yaml`: Runs on PR/push, performs linting and chart-testing
- `release.yaml`: Automatically packages and publishes chart on tag push
- Chart testing uses kind cluster for integration tests
- Configuration in `.github/ct.yaml` (maintainer validation disabled)

## Chart Version Requirements

- Kubernetes: 1.19+
- Helm: 3.2.0+
- Minimum resources: 500m CPU, 512Mi RAM
- PV provisioner required for persistence
