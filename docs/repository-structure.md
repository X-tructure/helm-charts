# Repository Structure Guide

This document explains the layout and organization of this multi-chart Helm repository.

## Directory Tree

```
helm-charts/
├── README.md                          # Repository overview
├── LICENSE                            # Apache 2.0 license
├── CLAUDE.md                          # Claude Code assistant instructions
│
├── charts/                            # Helm charts directory
│   ├── obsidian/                      # Obsidian note-taking app chart
│   │   ├── Chart.yaml                 # Chart metadata
│   │   ├── values.yaml                # Default configuration
│   │   ├── README.md                  # Chart documentation
│   │   │
│   │   ├── templates/                 # Kubernetes manifests
│   │   │   ├── _helpers.tpl           # Template helpers
│   │   │   ├── deployment.yaml        # Deployment resource
│   │   │   ├── service.yaml           # Service resource
│   │   │   ├── ingress.yaml           # Ingress resource (optional)
│   │   │   ├── pvc.yaml               # PersistentVolumeClaim
│   │   │   ├── serviceaccount.yaml    # ServiceAccount
│   │   │   ├── NOTES.txt              # Post-install notes
│   │   │   └── tests/                 # Helm test resources
│   │   │       └── test-connection.yaml
│   │   │
│   │   ├── examples/                  # Example configurations
│   │   │   ├── values-basic.yaml      # Minimal setup
│   │   │   ├── values-production.yaml # Production-ready
│   │   │   ├── values-with-ingress.yaml # External access
│   │   │   ├── values-gpu.yaml        # GPU acceleration
│   │   │   └── values-1080p.yaml      # Fixed resolution
│   │   │
│   │   └── docs/                      # Detailed documentation
│   │       ├── configuration.md       # Parameter reference
│   │       ├── installation.md        # Installation guide
│   │       ├── troubleshooting.md     # Common issues
│   │       ├── upgrade.md             # Upgrade instructions
│   │       └── auto-update.md         # Auto-update setup
│   │
│   ├── docetl/                        # DocETL document processing chart
│   │   ├── Chart.yaml                 # Chart metadata
│   │   ├── values.yaml                # Default configuration
│   │   ├── README.md                  # Chart documentation
│   │   ├── templates/                 # Kubernetes manifests
│   │   ├── examples/                  # Example configurations
│   │   │   ├── values-dev.yaml        # Development setup
│   │   │   └── values-prod.yaml       # Production setup
│   │   └── docs/                      # Detailed documentation (Phase 3)
│   │
│   ├── docling-serve/                 # Docling-Serve AI document conversion chart
│   │   ├── Chart.yaml                 # Chart metadata
│   │   ├── values.yaml                # Default configuration (CPU-optimized)
│   │   ├── README.md                  # Chart documentation
│   │   │
│   │   ├── templates/                 # Kubernetes manifests
│   │   │   ├── _helpers.tpl           # Template helpers
│   │   │   ├── deployment.yaml        # Deployment resource
│   │   │   ├── service.yaml           # Service resource
│   │   │   ├── ingress.yaml           # Ingress resource (optional)
│   │   │   ├── pvc.yaml               # PersistentVolumeClaim (optional)
│   │   │   ├── secret.yaml            # API key secret (optional)
│   │   │   ├── serviceaccount.yaml    # ServiceAccount (optional)
│   │   │   ├── NOTES.txt              # Post-install notes
│   │   │   └── tests/                 # Helm test resources
│   │   │       └── test-connection.yaml
│   │   │
│   │   ├── examples/                  # Example configurations
│   │   │   ├── values-basic.yaml      # Minimal testing setup
│   │   │   ├── values-production.yaml # Production-ready
│   │   │   ├── values-with-ingress.yaml # External access with TLS
│   │   │   └── values-gpu.yaml        # GPU acceleration (CUDA)
│   │   │
│   │   └── docs/                      # Detailed documentation
│   │       ├── configuration.md       # Parameter reference
│   │       ├── installation.md        # Installation guide
│   │       ├── troubleshooting.md     # Common issues
│   │       └── upgrade.md             # Upgrade instructions
│   │
│   └── [future-charts]/               # Additional charts
│
├── docs/                              # Shared documentation
│   ├── contributing.md                # How to contribute
│   ├── ci-cd.md                       # CI/CD documentation
│   └── repository-structure.md        # This file
│
├── scripts/                           # Utility scripts
│   ├── README.md                      # Scripts documentation
│   └── setup-auto-update.sh           # Auto-update configuration
│
└── .github/                           # GitHub configuration
    ├── workflows/                     # GitHub Actions
    │   ├── lint-test.yaml             # Lint and test charts
    │   ├── release.yaml               # Release and publish
    │   └── auto-update-obsidian.yaml  # Auto-update Obsidian
    │
    └── ct.yaml                        # Chart testing config
```

## Directory Purposes

### Root Level

#### `README.md`
- Repository introduction
- List of all available charts
- Quick start guide for each chart
- Installation and usage overview

#### `LICENSE`
- Apache License 2.0
- Legal terms for repository usage

#### `CLAUDE.md`
- Instructions for Claude Code assistant
- Common commands and workflows
- Chart development guidelines
- CI/CD processes

### `charts/`

Container for all Helm charts. Each subdirectory is an independent chart.

#### Chart Structure (`charts/<chart-name>/`)

**Required Files:**

- `Chart.yaml`: Chart metadata
  ```yaml
  apiVersion: v2
  name: chart-name
  version: 0.1.0          # Chart version (SemVer)
  appVersion: "1.0.0"     # Application version
  description: Brief description
  ```

- `values.yaml`: Default configuration values
  - Well-commented
  - Sensible defaults
  - All configurable parameters

- `README.md`: Chart-specific documentation
  - Installation instructions
  - Configuration guide
  - Examples
  - Links to detailed docs

- `templates/`: Kubernetes resource manifests
  - `_helpers.tpl`: Shared template functions
  - `NOTES.txt`: Post-install instructions
  - Resource files (deployment, service, etc.)
  - `tests/`: Helm test resources

**Recommended Additions:**

- `examples/`: Example configurations
  - At least `values-production.yaml`
  - Use cases (ingress, GPU, etc.)

- `docs/`: Detailed documentation
  - `configuration.md`: All parameters
  - `installation.md`: Step-by-step guide
  - `troubleshooting.md`: Common issues
  - `upgrade.md`: Version migration

- `.helmignore`: Files to exclude from packaging
  - `examples/`
  - `docs/`
  - `*.md` (except README.md)

### `docs/`

Shared documentation applying to all charts.

#### `contributing.md`
- How to add new charts
- Chart development best practices
- Testing requirements
- Release process

#### `ci-cd.md`
- CI/CD pipeline explanation
- Workflow details
- Configuration guide
- Troubleshooting

#### `repository-structure.md` (this file)
- Directory layout
- File purposes
- Conventions and standards

### `scripts/`

Utility scripts for repository management.

#### `setup-auto-update.sh`
- Configures auto-update workflows
- Sets up GitHub secrets/variables
- Interactive setup wizard

#### Future Scripts
- Chart scaffolding
- Version bumping
- Documentation generation

### `.github/`

GitHub-specific configuration and automation.

#### `workflows/`

**`lint-test.yaml`**
- Runs on: PRs and pushes to main
- Validates all charts
- Templates with examples
- Integration testing with kind

**`release.yaml`**
- Runs on: Git tags
- Packages charts
- Creates GitHub releases
- Publishes to GitHub Pages

**`auto-update-<chart>.yaml`**
- Runs on: Schedule or manual trigger
- Monitors upstream releases
- Creates PRs or auto-commits
- Sends notifications

#### `ct.yaml`
Configuration for [chart-testing](https://github.com/helm/chart-testing):
```yaml
chart-dirs:
  - charts
chart-repos:
  - bitnami=https://charts.bitnami.com/bitnami
helm-extra-args: --timeout 600s
validate-maintainers: false
```

## File Naming Conventions

### Charts
- Directory: Lowercase, hyphens (e.g., `my-app`)
- Chart.yaml name: Must match directory
- Templates: Lowercase, hyphens, descriptive

### Documentation
- Markdown files: Lowercase, hyphens
- Example values: `values-<use-case>.yaml`

### Workflows
- Generic: `<action>.yaml` (e.g., `release.yaml`)
- Chart-specific: `<action>-<chart>.yaml` (e.g., `auto-update-obsidian.yaml`)

## Chart Independence

Each chart is **self-contained**:

- ✅ Independent versioning
- ✅ Own documentation
- ✅ Own examples
- ✅ Own CI/CD (if needed)
- ❌ No cross-chart dependencies

This allows:
- Individual chart releases
- Different maintainers per chart
- Varying complexity levels
- Flexible workflows

## GitHub Pages Structure

After `release.yaml` runs, the `gh-pages` branch contains:

```
gh-pages/
├── index.yaml                 # Helm repository index
└── obsidian-0.1.3.tgz        # Packaged charts
```

Users add the repository:
```bash
helm repo add <repo-name> https://<user>.github.io/<repo-name>
```

## Multi-Chart vs Mono-Chart

This repository uses **multi-chart** architecture:

| Aspect | Multi-Chart (This Repo) | Mono-Chart |
|--------|------------------------|------------|
| Charts per repo | Multiple (1+) | One |
| Directory | `charts/<name>/` | `charts/` or root |
| Versioning | Independent | Single |
| Release | Per chart | Entire repo |
| Complexity | Higher | Lower |
| Flexibility | High | Low |

**When to use multi-chart:**
- Related charts (same team/domain)
- Shared CI/CD infrastructure
- Consistent development practices
- Multiple small charts

**When to use mono-chart:**
- Single large chart
- Dedicated repository
- Simpler release process

## Best Practices

### Chart Organization
1. Keep charts independent
2. Document thoroughly
3. Provide examples
4. Test all configurations
5. Follow Helm best practices

### Documentation
1. README in each chart
2. Shared docs in `docs/`
3. Keep examples updated
4. Link between docs

### Version Control
1. Bump chart version on changes
2. Use semantic versioning
3. Tag releases properly
4. Document breaking changes

### CI/CD
1. Test before merging
2. Use automatic detection
3. Keep workflows generic
4. Chart-specific only when needed

## Adding New Charts

Quick checklist:

```bash
# 1. Create chart structure
helm create charts/my-app

# 2. Add required directories
mkdir -p charts/my-app/examples
mkdir -p charts/my-app/docs

# 3. Create documentation
touch charts/my-app/README.md
touch charts/my-app/docs/configuration.md
touch charts/my-app/examples/values-production.yaml

# 4. Develop and test
helm lint charts/my-app/
helm template test charts/my-app/

# 5. Submit PR
git checkout -b feature/add-my-app
git add charts/my-app/
git commit -m "feat: add my-app chart"
```

See [contributing.md](contributing.md) for details.

## Migration from Single Chart

If migrating from a single-chart repository:

1. **Move chart**: `charts/chart-name/`
2. **Move examples**: `charts/chart-name/examples/`
3. **Move docs**: `charts/chart-name/docs/`
4. **Update workflows**: Generic chart detection
5. **Update README**: Repository overview
6. **Create shared docs**: In `docs/`

Original structure preserved, just reorganized!

## Resources

- [Helm Charts Guide](https://helm.sh/docs/topics/charts/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Chart Releaser](https://github.com/helm/chart-releaser)
- [Chart Testing](https://github.com/helm/chart-testing)
- [Artifact Hub](https://artifacthub.io/)

## Questions?

- Check existing charts for examples
- Review [contributing guide](contributing.md)
- See [CI/CD documentation](ci-cd.md)
- Open a GitHub issue
