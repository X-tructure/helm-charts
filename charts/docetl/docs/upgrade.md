# DocETL Upgrade Guide

This guide covers upgrading the DocETL Helm chart to newer versions.

## Before You Upgrade

### 1. Review the Changelog

Check the [Releases page](https://github.com/X-tructure/helm-charts/releases) for:
- New features
- Breaking changes
- Deprecated parameters
- Migration steps

### 2. Backup Your Data

**Always backup before upgrading**, especially for major version changes:

```bash
# List PVCs
kubectl get pvc -l app.kubernetes.io/instance=docetl

# Create a snapshot (if supported by your storage class)
kubectl get volumesnapshot

# Or backup via pod
kubectl exec <backend-pod> -- tar czf /tmp/backup.tar.gz /docetl-data
kubectl cp <backend-pod>:/tmp/backup.tar.gz ./backup-$(date +%Y%m%d).tar.gz
```

### 3. Check Current Version

```bash
# Current chart version
helm list -n <namespace>

# Current configuration
helm get values docetl > current-values.yaml
```

## Upgrade Methods

### Method 1: Standard Upgrade (Recommended)

Use this for minor/patch version upgrades with no breaking changes:

```bash
# Update Helm repository
helm repo update extreme_structure

# Upgrade to latest version
helm upgrade docetl extreme_structure/docetl \
  -f my-values.yaml \
  --namespace docetl

# Upgrade to specific version
helm upgrade docetl extreme_structure/docetl \
  --version 1.2.0 \
  -f my-values.yaml
```

### Method 2: Upgrade with Dry-Run

Test the upgrade before applying:

```bash
# Dry-run to see what will change
helm upgrade docetl extreme_structure/docetl \
  -f my-values.yaml \
  --dry-run --debug

# If looks good, apply
helm upgrade docetl extreme_structure/docetl \
  -f my-values.yaml
```

### Method 3: Forced Upgrade

Use when pods need recreation (e.g., image updates):

```bash
helm upgrade docetl extreme_structure/docetl \
  -f my-values.yaml \
  --force
```

**Warning**: `--force` deletes and recreates pods, causing downtime.

### Method 4: Atomic Upgrade with Rollback

Automatically rollback if upgrade fails:

```bash
helm upgrade docetl extreme_structure/docetl \
  -f my-values.yaml \
  --atomic \
  --timeout 5m
```

## Version-Specific Upgrade Notes

### Upgrading to 1.1.x from 1.0.x

**Changes:**
- Added Helm test manifests
- Improved documentation structure
- New example configurations

**Migration steps:**
```bash
# No breaking changes, standard upgrade
helm upgrade docetl extreme_structure/docetl -f my-values.yaml

# Verify with tests
helm test docetl
```

### Upgrading to 2.0.x from 1.x.x (Future)

**Hypothetical breaking changes** (template for future major versions):

1. **Parameter renames:**
   ```yaml
   # Old (1.x)
   secrets:
     openaiApiKey: "..."

   # New (2.x)
   backend:
     openaiApiKey: "..."
   ```

2. **Migration steps:**
   ```bash
   # Update values file
   vim my-values.yaml  # Move secrets.openaiApiKey → backend.openaiApiKey

   # Upgrade
   helm upgrade docetl extreme_structure/docetl \
     --version 2.0.0 \
     -f my-values.yaml
   ```

## Rollback Procedures

### Automatic Rollback

If an upgrade fails with `--atomic`, Helm automatically rolls back:

```bash
helm upgrade docetl extreme_structure/docetl \
  -f my-values.yaml \
  --atomic
```

### Manual Rollback

#### View History

```bash
# List release history
helm history docetl

# Output:
# REVISION  UPDATED                   STATUS      CHART           DESCRIPTION
# 1         Mon Jan 1 10:00:00 2024   superseded  docetl-1.0.0    Install complete
# 2         Mon Jan 2 11:00:00 2024   deployed    docetl-1.1.0    Upgrade complete
```

#### Rollback to Previous Version

```bash
# Rollback to previous revision
helm rollback docetl

# Rollback to specific revision
helm rollback docetl 1

# Rollback with dry-run
helm rollback docetl 1 --dry-run
```

#### Emergency Rollback

If Helm rollback fails, manually restore:

```bash
# 1. Scale down current deployment
kubectl scale deployment docetl-backend --replicas=0
kubectl scale deployment docetl-frontend --replicas=0

# 2. Restore from backup (if PVC was changed)
kubectl cp ./backup-20240101.tar.gz <backend-pod>:/tmp/backup.tar.gz
kubectl exec <backend-pod> -- tar xzf /tmp/backup.tar.gz -C /

# 3. Reinstall previous version
helm uninstall docetl
helm install docetl extreme_structure/docetl \
  --version 1.0.0 \
  -f my-values.yaml
```

## Post-Upgrade Verification

### 1. Check Release Status

```bash
helm list
helm status docetl
```

### 2. Verify Pods

```bash
# All pods should be Running
kubectl get pods -l app.kubernetes.io/instance=docetl

# Check pod events
kubectl get events --field-selector involvedObject.kind=Pod
```

### 3. Run Helm Tests

```bash
helm test docetl
```

### 4. Verify Application

```bash
# Port-forward and test
kubectl port-forward svc/docetl-frontend 3000:3000

# Check backend health
curl http://localhost:8000/health

# Check frontend
curl http://localhost:3000
```

### 5. Check Logs

```bash
# No errors in backend logs
kubectl logs -l app.kubernetes.io/component=backend --tail=50

# No errors in frontend logs
kubectl logs -l app.kubernetes.io/component=frontend --tail=50
```

## Data Migration

### Migrating to New PVC

If you need to migrate data to a new PVC (e.g., changing storage class):

```bash
# 1. Create new PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: docetl-data-new
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi
EOF

# 2. Create temporary migration pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: data-migration
spec:
  containers:
  - name: migrate
    image: busybox
    command: ['sh', '-c', 'cp -av /old/* /new/ && sync']
    volumeMounts:
    - name: old-data
      mountPath: /old
    - name: new-data
      mountPath: /new
  volumes:
  - name: old-data
    persistentVolumeClaim:
      claimName: docetl-data
  - name: new-data
    persistentVolumeClaim:
      claimName: docetl-data-new
  restartPolicy: Never
EOF

# 3. Wait for migration
kubectl wait --for=condition=complete pod/data-migration --timeout=600s

# 4. Update values to use new PVC
helm upgrade docetl extreme_structure/docetl \
  --set persistence.existingClaim=docetl-data-new \
  -f my-values.yaml

# 5. Verify data
kubectl exec -it <backend-pod> -- ls -la /docetl-data

# 6. Delete old PVC (after verification)
kubectl delete pvc docetl-data
```

### Migrating Between Clusters

1. **Export data from old cluster:**
   ```bash
   kubectl exec <old-backend-pod> -- tar czf - /docetl-data > backup.tar.gz
   ```

2. **Install chart in new cluster:**
   ```bash
   helm install docetl extreme_structure/docetl -f my-values.yaml
   ```

3. **Import data to new cluster:**
   ```bash
   kubectl exec -i <new-backend-pod> -- tar xzf - -C / < backup.tar.gz
   ```

4. **Restart pods:**
   ```bash
   kubectl rollout restart deployment/docetl-backend
   ```

## Upgrading Application Version

To upgrade the DocETL application (not chart version):

```bash
# Update image tag
helm upgrade docetl extreme_structure/docetl \
  --set image.tag=v0.3.0 \
  --reuse-values

# Or update in values file
helm upgrade docetl extreme_structure/docetl -f my-values.yaml
```

**Note**: Check [DocETL releases](https://github.com/ucbepic/docetl/releases) for breaking changes in the application.

## Automated Upgrade Tracking

For automatic notifications of new chart versions, see the [Auto-Update Guide](auto-update.md).

## Troubleshooting Upgrades

### Upgrade Stuck

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/instance=docetl

# Check events
kubectl get events --sort-by='.lastTimestamp' | grep docetl

# View pod logs
kubectl logs -l app.kubernetes.io/instance=docetl --all-containers
```

### Configuration Errors

```bash
# Validate new values
helm template docetl extreme_structure/docetl -f my-values.yaml

# Compare with current values
helm get values docetl
```

### Rollback Not Working

```bash
# Force delete stuck resources
kubectl delete pod <pod-name> --grace-period=0 --force

# Clean up and reinstall
helm uninstall docetl
helm install docetl extreme_structure/docetl -f my-values.yaml
```

## Best Practices

1. **Always test in non-production first** - Validate upgrades in dev/staging
2. **Use semantic versioning** - Understand what MAJOR.MINOR.PATCH means
3. **Read release notes** - Don't skip this step
4. **Backup before upgrading** - Especially for major versions
5. **Use `--dry-run`** - Preview changes before applying
6. **Keep values file in version control** - Track configuration changes
7. **Monitor after upgrade** - Watch logs and metrics for issues
8. **Have a rollback plan** - Know how to revert quickly

## Support

If you encounter issues during upgrade:

1. Check [Troubleshooting Guide](troubleshooting.md)
2. Review [Configuration Reference](configuration.md)
3. Search [GitHub Issues](https://github.com/X-tructure/helm-charts/issues)
4. Create a new issue with:
   - Current version
   - Target version
   - Error messages
   - Steps to reproduce
