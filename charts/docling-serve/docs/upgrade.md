# Upgrade Guide

Guide for upgrading the Docling-Serve Helm chart to newer versions.

## General Upgrade Process

### Standard Upgrade

```bash
# Update repository
helm repo update

# Upgrade to latest version
helm upgrade my-docling-serve extreme_structure/docling-serve

# Or specify version
helm upgrade my-docling-serve extreme_structure/docling-serve --version 0.2.0
```

### Upgrade with Values

```bash
# Reuse existing values
helm upgrade my-docling-serve extreme_structure/docling-serve --reuse-values

# Or with custom values file
helm upgrade my-docling-serve extreme_structure/docling-serve -f my-values.yaml
```

### Dry Run First

Always test upgrades with --dry-run:

```bash
helm upgrade my-docling-serve extreme_structure/docling-serve --dry-run --debug
```

## Version Compatibility

### Chart Version 0.1.x

- **Kubernetes:** 1.19+
- **Helm:** 3.2.0+
- **Docling-Serve:** 1.9.0
- **CUDA:** 12.6, 12.8 (GPU variants)

## Breaking Changes

### v0.1.2 → v0.2.0 (Model Persistence Pattern Change)

**Summary:** Version 0.2.0 replaces the **Kubernetes Job pattern** for model downloads with an **init container pattern**. This is a **breaking change** for users who have `models.enabled=true`.

#### What Changed

| Aspect | v0.1.2 (Job Pattern) | v0.2.0 (Init Container) |
|--------|---------------------|-------------------------|
| **Download mechanism** | Separate Kubernetes Job | Init container in pod |
| **Configuration** | `models.job.*` parameters | `models.initContainer.*` parameters |
| **Timing** | Job runs independently | Download before app starts (guaranteed) |
| **Resource management** | Job + Deployment | Deployment only |
| **Idempotency** | None (downloads every time) | Built-in (skips if models exist) |

#### Migration Path

**Option A: Fresh Install (Recommended for testing)**

If you can tolerate downtime:

1. **Uninstall old chart:**
   ```bash
   helm uninstall my-docling
   ```

2. **Keep PVC if models already downloaded** (optional):
   ```bash
   # List PVCs
   kubectl get pvc

   # Note the PVC name (e.g., my-docling-models)
   ```

3. **Install v0.2.0 with existingClaim:**
   ```bash
   helm install my-docling charts/docling-serve/ \
     --set models.enabled=true \
     --set models.pvc.existingClaim=<old-pvc-name>
   ```

**Option B: In-Place Upgrade (Recommended for production)**

If you need minimal downtime:

1. **Update values file** - remove all `models.job.*` parameters:

   ```yaml
   # OLD (v0.1.2):
   models:
     enabled: true
     job:
       resources: {...}
       backoffLimit: 3
       ttlSecondsAfterFinished: 3600
       nodeSelector: {}

   # NEW (v0.2.0):
   models:
     enabled: true
     initContainer:
       resources: {...}
     # Removed: job.backoffLimit, job.ttlSecondsAfterFinished, job.nodeSelector
   ```

2. **Run Helm upgrade:**
   ```bash
   helm upgrade my-docling charts/docling-serve/ -f values.yaml
   ```

3. **Delete old Job** (if still running):
   ```bash
   kubectl delete job <release-name>-model-download
   ```

4. **Wait for pods to restart:**
   ```bash
   kubectl rollout status deployment <release-name>
   ```

#### Configuration Changes Required

| Old Parameter (v0.1.2) | New Parameter (v0.2.0) | Action |
|------------------------|------------------------|--------|
| `models.job.resources` | `models.initContainer.resources` | Rename |
| `models.job.image.*` | `models.initContainer.image.*` | Rename |
| `models.job.backoffLimit` | N/A (removed) | Delete |
| `models.job.ttlSecondsAfterFinished` | N/A (removed) | Delete |
| `models.job.restartPolicy` | N/A (removed) | Delete |
| `models.job.nodeSelector` | N/A (removed) | Delete |
| `models.job.tolerations` | N/A (removed) | Delete |
| `models.job.affinity` | N/A (removed) | Delete |
| `models.job.annotations` | N/A (removed) | Delete |
| N/A | `models.forceRedownload` | Add (optional) |

#### New Features in v0.2.0

1. **Idempotent downloads**: Models only downloaded once, skipped on pod restarts
2. **Force re-download option**: `models.forceRedownload=true` for updates
3. **Simpler architecture**: No separate Job resource to manage
4. **Guaranteed ordering**: Init container ensures models ready before app starts
5. **Faster restarts**: Subsequent pod starts skip download (~1min total)

#### Troubleshooting Migration

**Issue:** Pods stuck in `Init:0/1` state

**Solution:** Check init container logs
```bash
kubectl logs <pod-name> -c model-downloader
```

**Issue:** `Error: unknown field "models.job"`

**Solution:** Remove all `models.job.*` parameters from values

**Issue:** Models re-downloading every pod restart

**Solution:** Check PVC is persistent and marker file created
```bash
kubectl exec <pod-name> -- cat /modelcache/.models-downloaded
```

#### Rollback Procedure

If you need to rollback to v0.1.2:

```bash
# Rollback Helm release
helm rollback my-docling

# Note: You may need to manually restore Job if it was deleted
```

### v0.1.0 → Future Versions

Breaking changes will be documented here for future versions.

## Upgrade Scenarios

### Scenario 1: Upgrading Application Version

To upgrade to a newer Docling-Serve version:

```bash
# Check available versions
helm search repo extreme_structure/docling-serve --versions

# Upgrade with new app version
helm upgrade my-docling-serve extreme_structure/docling-serve \
  --set image.tag=1.10.0 \
  --reuse-values
```

### Scenario 2: Migrating to Persistent Storage

If you started with ephemeral storage and want persistence:

```bash
# Enable persistence
helm upgrade my-docling-serve extreme_structure/docling-serve \
  --set scratch.persistent=true \
  --set scratch.pvc.size=50Gi \
  --reuse-values

# Note: Existing data in emptyDir will be lost
```

**Important:** This triggers pod recreation and data loss from emptyDir.

### Scenario 3: Enabling API Authentication

To enable API key authentication after initial deployment:

```bash
# Enable with auto-generated key
helm upgrade my-docling-serve extreme_structure/docling-serve \
  --set apiKey.enabled=true \
  --reuse-values

# Retrieve generated key
kubectl get secret my-docling-serve-api-key -o jsonpath='{.data.api-key}' | base64 -d
```

### Scenario 4: CPU to GPU Migration

To migrate from CPU to GPU:

```bash
helm upgrade my-docling-serve extreme_structure/docling-serve \
  -f examples/values-gpu.yaml \
  --reuse-values
```

**Prerequisites:**
- GPU nodes available
- NVIDIA GPU Operator installed

### Scenario 5: Scaling Resources

To adjust resource allocation:

```bash
helm upgrade my-docling-serve extreme_structure/docling-serve \
  --set resources.requests.cpu=1 \
  --set resources.requests.memory=4Gi \
  --set resources.limits.cpu=4 \
  --set resources.limits.memory=16Gi \
  --reuse-values
```

## Rollback

If upgrade fails or causes issues:

```bash
# View release history
helm history my-docling-serve

# Rollback to previous version
helm rollback my-docling-serve

# Or rollback to specific revision
helm rollback my-docling-serve 2
```

## Data Migration

### Persistent Volume Data

If using persistent storage, data persists across upgrades:

```bash
# Data in PVC is preserved
kubectl get pvc my-docling-serve-scratch
```

### Backup Before Upgrade

If you have critical data:

```bash
# Create backup of PVC (method depends on storage provider)
# Example with volumesnapshot
kubectl create -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: docling-backup-$(date +%Y%m%d)
spec:
  source:
    persistentVolumeClaimName: my-docling-serve-scratch
EOF
```

## Post-Upgrade Verification

### 1. Check Pod Status

```bash
# Wait for rollout
kubectl rollout status deployment/my-docling-serve

# Verify pod is running
kubectl get pods -l app.kubernetes.io/name=docling-serve
```

### 2. Verify API Functionality

```bash
# Port-forward
kubectl port-forward svc/my-docling-serve 5001:5001

# Test endpoints
curl http://localhost:5001/version
curl http://localhost:5001/docs
```

### 3. Check Logs

```bash
kubectl logs -l app.kubernetes.io/name=docling-serve --tail=50
```

### 4. Run Helm Test

```bash
helm test my-docling-serve
```

## Troubleshooting Upgrades

### Upgrade Stuck

**Symptoms:**
- Helm upgrade hangs
- New pods not starting

**Solutions:**

1. **Check Pod Status**
   ```bash
   kubectl get pods -l app.kubernetes.io/name=docling-serve
   kubectl describe pod <pod-name>
   ```

2. **Force Recreation**
   ```bash
   # Delete stuck pod
   kubectl delete pod <pod-name>
   ```

3. **Rollback if Necessary**
   ```bash
   helm rollback my-docling-serve
   ```

### Configuration Conflicts

**Symptoms:**
- Helm reports validation errors
- Invalid value combinations

**Solutions:**

1. **Review Current Values**
   ```bash
   helm get values my-docling-serve
   ```

2. **Reset to Defaults**
   ```bash
   # Don't use --reuse-values, provide clean values
   helm upgrade my-docling-serve extreme_structure/docling-serve \
     -f new-values.yaml
   ```

### Resource Quota Exceeded

**Symptoms:**
- "exceeded quota" errors
- Pods stuck in Pending

**Solutions:**

1. **Check Quota**
   ```bash
   kubectl describe quota -n <namespace>
   ```

2. **Adjust Resources**
   ```bash
   helm upgrade my-docling-serve extreme_structure/docling-serve \
     --set resources.requests.cpu=250m \
     --set resources.requests.memory=1Gi \
     --reuse-values
   ```

## Best Practices

### 1. Test in Non-Production First

Always test upgrades in development/staging:

```bash
# Deploy test instance
helm install docling-test extreme_structure/docling-serve \
  --namespace test \
  -f production-values.yaml

# Verify functionality
# Then upgrade production
```

### 2. Use Version Pinning

Pin chart versions in production:

```bash
# Install specific version
helm install my-docling-serve extreme_structure/docling-serve \
  --version 0.1.0

# Upgrade to specific version only
helm upgrade my-docling-serve extreme_structure/docling-serve \
  --version 0.2.0
```

### 3. Maintain Values Files

Keep values in version control:

```bash
# values.yaml in git
helm upgrade my-docling-serve extreme_structure/docling-serve \
  -f values.yaml
```

### 4. Monitor After Upgrade

Watch metrics and logs:

```bash
# Monitor pods
kubectl get pods -l app.kubernetes.io/name=docling-serve -w

# Follow logs
kubectl logs -l app.kubernetes.io/name=docling-serve -f
```

## Migration Guides

### Migrating from Manual Deployment

If migrating from manual Kubernetes manifests:

1. **Export Current Configuration**
   ```bash
   kubectl get deployment docling-serve -o yaml > deployment.yaml
   kubectl get svc docling-serve -o yaml > service.yaml
   ```

2. **Create Equivalent Values**
   ```yaml
   # Match existing configuration
   replicaCount: 1  # From deployment
   resources: {}    # From deployment.spec.template.spec.containers[0].resources
   ```

3. **Install Chart**
   ```bash
   helm install my-docling-serve extreme_structure/docling-serve \
     -f migrated-values.yaml
   ```

4. **Delete Manual Resources**
   ```bash
   kubectl delete deployment docling-serve
   kubectl delete svc docling-serve
   ```

## Support

For upgrade issues:
- [Troubleshooting Guide](troubleshooting.md)
- [GitHub Issues](https://github.com/X-tructure/helm-charts/issues)
- [Helm Documentation](https://helm.sh/docs/helm/helm_upgrade/)

## Changelog

### v0.1.0 (Initial Release)

- Initial chart release
- CPU-optimized defaults
- Optional GPU support
- API key authentication
- Ephemeral and persistent storage options
- Comprehensive documentation
