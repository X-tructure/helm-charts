# Upgrade Guide

This guide covers upgrading the Obsidian Helm chart to newer versions.

## Before Upgrading

### 1. Review Release Notes

Check the [GitHub Releases](https://github.com/X-tructure/helm-charts/releases) for:
- Breaking changes
- New features
- Required actions
- Known issues

### 2. Backup Your Data

**Always backup your data before upgrading:**

```bash
# Method 1: Backup PVC data
kubectl exec <pod-name> -n <namespace> -- tar czf /tmp/backup.tar.gz /config
kubectl cp <pod-name>:/tmp/backup.tar.gz ./backup.tar.gz -n <namespace>

# Method 2: Snapshot PVC (if supported by your storage class)
kubectl get pvc <pvc-name> -n <namespace> -o yaml > pvc-backup.yaml
# Create snapshot according to your storage provider's documentation
```

### 3. Check Current Version

```bash
# Check deployed chart version
helm list -n <namespace>

# Check current values
helm get values my-obsidian -n <namespace> > current-values.yaml
```

## Upgrade Methods

### Method 1: Upgrade from Helm Repository

```bash
# Update repository
helm repo update extreme_structure

# Check available versions
helm search repo extreme_structure/obsidian --versions

# Upgrade to latest version
helm upgrade my-obsidian extreme_structure/obsidian -n <namespace>

# Upgrade to specific version
helm upgrade my-obsidian extreme_structure/obsidian --version 0.2.0 -n <namespace>

# Upgrade with custom values
helm upgrade my-obsidian extreme_structure/obsidian \
  -f current-values.yaml \
  -n <namespace>
```

### Method 2: Upgrade from Git

```bash
# Pull latest changes
cd helm-charts
git pull origin main

# Upgrade
helm upgrade my-obsidian ./charts/obsidian -n <namespace>
```

## Upgrade Scenarios

### Scenario 1: Simple Upgrade (No Breaking Changes)

```bash
helm repo update
helm upgrade my-obsidian extreme_structure/obsidian -n <namespace>
```

### Scenario 2: Upgrade with Value Changes

```bash
# Upgrade and modify values
helm upgrade my-obsidian obsidian/obsidian \
  --set persistence.size=20Gi \
  --set resources.limits.memory=4Gi \
  -n <namespace>
```

### Scenario 3: Upgrade with PVC Size Increase

**Note:** PVC size can only be increased, not decreased, and requires:
- Storage class supports volume expansion
- `allowVolumeExpansion: true` in StorageClass

```bash
# 1. Check if storage class supports expansion
kubectl get storageclass <storage-class> -o yaml | grep allowVolumeExpansion

# 2. Upgrade with new size
helm upgrade my-obsidian obsidian/obsidian \
  --set persistence.size=50Gi \
  -n <namespace>

# 3. Monitor PVC resize
kubectl get pvc <pvc-name> -n <namespace> -w
```

### Scenario 4: Migrate to New PVC

If you need to change storage class or decrease size:

```bash
# 1. Backup data (see "Before Upgrading" section)

# 2. Delete old release (keeps PVC by default)
helm uninstall my-obsidian -n <namespace>

# 3. Create new PVC or let chart create it
# (optional) kubectl apply -f new-pvc.yaml

# 4. Reinstall with new configuration
helm install my-obsidian extreme_structure/obsidian \
  --set persistence.storageClass=new-storage-class \
  --set persistence.size=30Gi \
  -n <namespace>

# 5. Restore data
kubectl cp ./backup.tar.gz <new-pod-name>:/tmp/backup.tar.gz -n <namespace>
kubectl exec <new-pod-name> -n <namespace> -- tar xzf /tmp/backup.tar.gz -C /
```

## Version-Specific Upgrade Notes

### Upgrading to 0.2.0 (Example)

**Breaking Changes:**
- None

**New Features:**
- Improved health checks
- Additional Selkies options

**Recommended Actions:**
- Review new probe settings if you customized them
- Consider enabling new Selkies features

## Rolling Back

If an upgrade causes issues:

### Rollback to Previous Version

```bash
# Rollback to previous release
helm rollback my-obsidian -n <namespace>

# Rollback to specific revision
helm history my-obsidian -n <namespace>
helm rollback my-obsidian <revision> -n <namespace>
```

### Rollback and Restore Data

```bash
# 1. Rollback
helm rollback my-obsidian -n <namespace>

# 2. If data was affected, restore from backup
kubectl cp ./backup.tar.gz <pod-name>:/tmp/backup.tar.gz -n <namespace>
kubectl exec <pod-name> -n <namespace> -- tar xzf /tmp/backup.tar.gz -C /
```

## Post-Upgrade Verification

### 1. Check Release Status

```bash
# Verify upgrade succeeded
helm status my-obsidian -n <namespace>

# Check release history
helm history my-obsidian -n <namespace>
```

### 2. Verify Pods are Running

```bash
# Check pod status
kubectl get pods -n <namespace> -l app.kubernetes.io/name=obsidian

# Check pod logs
kubectl logs -n <namespace> -l app.kubernetes.io/name=obsidian

# Wait for pod to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=obsidian \
  -n <namespace> \
  --timeout=300s
```

### 3. Test Application

```bash
# Port-forward and access
kubectl port-forward svc/<service-name> 3001:3001 -n <namespace>
# Visit https://localhost:3001 and verify functionality
```

### 4. Verify Data Integrity

- Log into the application
- Check that your vaults and files are intact
- Test creating/editing notes
- Verify plugins and settings

## Troubleshooting Upgrades

### Issue: Upgrade Stuck or Timeout

```bash
# Check pod status
kubectl get pods -n <namespace>

# Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Force delete stuck pod if necessary
kubectl delete pod <pod-name> -n <namespace> --force --grace-period=0
```

### Issue: Application Not Working After Upgrade

1. **Check logs:**
   ```bash
   kubectl logs -n <namespace> -l app.kubernetes.io/name=obsidian --tail=100
   ```

2. **Compare configurations:**
   ```bash
   # Get new values
   helm get values my-obsidian -n <namespace> > new-values.yaml
   
   # Compare with backup
   diff current-values.yaml new-values.yaml
   ```

3. **Rollback if needed:**
   ```bash
   helm rollback my-obsidian -n <namespace>
   ```

### Issue: PVC Resize Not Working

```bash
# Check PVC events
kubectl describe pvc <pvc-name> -n <namespace>

# Verify storage class supports expansion
kubectl get storageclass <storage-class> -o jsonpath='{.allowVolumeExpansion}'

# Manual PVC resize (if needed)
kubectl patch pvc <pvc-name> -n <namespace> -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

## Automation and GitOps

### Using Helm with ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: obsidian
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://x-tructure.github.io/helm-charts
    chart: obsidian
    targetRevision: 0.1.0
    helm:
      values: |
        persistence:
          size: 20Gi
        auth:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: obsidian
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Using Flux

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: obsidian
  namespace: obsidian
spec:
  interval: 5m
  chart:
    spec:
      chart: obsidian
      version: '>=0.1.0 <1.0.0'
      sourceRef:
        kind: HelmRepository
        name: obsidian
        namespace: flux-system
  values:
    persistence:
      size: 20Gi
    auth:
      enabled: true
```

## Best Practices

1. **Always backup before upgrading**
2. **Test upgrades in a non-production environment first**
3. **Review release notes for breaking changes**
4. **Use version constraints in GitOps tools**
5. **Monitor application after upgrade**
6. **Keep upgrade rollback plan ready**
7. **Document any custom configurations**

## Getting Help

If you encounter issues during upgrade:

1. Check [Troubleshooting Guide](troubleshooting.md)
2. Review [GitHub Issues](https://github.com/X-tructure/helm-charts/issues)
3. Open a new issue with:
   - Current version
   - Target version
   - Error messages
   - Relevant logs
   - Helm values (redacted)
