# Upgrade Guide: v1.x to v2.0.0

## Overview

DocETL v2.0.0 introduces a **breaking architectural change** that transforms the deployment model from separate frontend and backend deployments to a unified multi-container pod architecture.

## What Changed

### Architecture

**Before (v1.x):**
- Two separate Deployments: `backend-deployment` and `frontend-deployment`
- Two separate Services: `backend-service` and `frontend-service`
- Two separate ConfigMaps: `backend-config` and `frontend-config`
- Frontend → Backend communication via Kubernetes Service DNS

**After (v2.0.0):**
- Single Deployment with two containers: `backend` and `frontend`
- Single Service with two ports: `frontendPort` (3000) and `backendPort` (8000)
- Single ConfigMap with combined configuration
- Frontend → Backend communication via `localhost`

### Benefits

✅ **Simplified Operations**: Single deployment, single service, single replica count
✅ **Better Networking**: Localhost communication, no DNS overhead, no CORS complexity
✅ **Resource Efficiency**: Better resource density, natural fit for ReadWriteOnce storage
✅ **Operational Simplicity**: Fewer Kubernetes resources to manage
✅ **Unified Scaling**: Frontend and backend scale together (acceptable for DocETL workloads)

## Breaking Changes

### 1. Values File Structure

**values.yaml changes:**

```yaml
# OLD (v1.x)
backend:
  replicaCount: 2
  service:
    type: ClusterIP
    port: 8000
    targetPort: 8000

frontend:
  replicaCount: 2
  service:
    type: ClusterIP
    port: 3000
    targetPort: 3000

# NEW (v2.0.0)
replicaCount: 2  # Single replica count for the entire pod

service:  # Single service configuration
  type: ClusterIP
  frontendPort: 3000
  backendPort: 8000

backend:
  # No replicaCount or service config
  resources: { ... }
  livenessProbe: { ... }
  readinessProbe: { ... }

frontend:
  # No replicaCount or service config
  resources: { ... }
  livenessProbe: { ... }
  readinessProbe: { ... }
```

### 2. Health Probe Port References

**OLD:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000  # Port number
```

**NEW:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: backend  # Port name
```

### 3. Kubernetes Resource Names

| Resource Type | v1.x Name | v2.0.0 Name |
|---------------|-----------|-------------|
| Deployment | `<name>-backend`, `<name>-frontend` | `<name>` |
| Service | `<name>-backend`, `<name>-frontend` | `<name>` |
| ConfigMap | `<name>-backend-config`, `<name>-frontend-config` | `<name>-config` |

### 4. Scaling Behavior

**OLD (v1.x):** Frontend and backend scale independently
```yaml
backend:
  replicaCount: 3
frontend:
  replicaCount: 2
```

**NEW (v2.0.0):** Frontend and backend scale together
```yaml
replicaCount: 2  # Both containers scale together
```

## Migration Steps

### Step 1: Backup Your Data

If using persistent storage, backup your data before upgrading:

```bash
# Create a backup of the PVC data
kubectl exec deployment/<release-name>-backend -- tar czf /tmp/backup.tar.gz /docetl-data

# Copy backup to local machine
kubectl cp <backend-pod-name>:/tmp/backup.tar.gz ./docetl-backup-$(date +%Y%m%d).tar.gz
```

### Step 2: Update Your values.yaml

Transform your values file from v1.x to v2.0.0 format:

```bash
# Example transformation
# OLD values-custom.yaml (v1.x)
backend:
  replicaCount: 2
  resources:
    limits:
      cpu: 2000m
      memory: 4Gi

frontend:
  replicaCount: 2
  resources:
    limits:
      cpu: 1000m
      memory: 2Gi

# NEW values-custom.yaml (v2.0.0)
replicaCount: 2

backend:
  resources:
    limits:
      cpu: 2000m
      memory: 4Gi

frontend:
  resources:
    limits:
      cpu: 1000m
      memory: 2Gi
```

**Key Changes:**
1. Move `backend.replicaCount` and `frontend.replicaCount` → top-level `replicaCount`
2. Remove `backend.service` and `frontend.service` sections (if overridden)
3. Keep resource specifications under `backend` and `frontend`

### Step 3: Uninstall v1.x

```bash
# Uninstall the old version (keeps PVC by default)
helm uninstall <release-name> -n <namespace>

# Verify PVC is retained
kubectl get pvc -n <namespace>
```

### Step 4: Install v2.0.0

```bash
# Install new version
helm install <release-name> oci://your-registry/docetl \
  --version 2.0.0 \
  -f values-custom.yaml \
  --set secrets.openaiApiKey=<your-api-key> \
  -n <namespace>

# Wait for deployment to be ready
kubectl wait --for=condition=available --timeout=300s \
  deployment/<release-name> -n <namespace>
```

### Step 5: Verify Deployment

```bash
# Check deployment has both containers
kubectl get deployment <release-name> -n <namespace>
# Should show: READY 2/2

# Check pod has both containers
kubectl get pods -n <namespace>
# Each pod should show: READY 2/2

# Verify both containers are running
kubectl get pods -n <namespace> -o jsonpath='{.items[0].spec.containers[*].name}'
# Should output: backend frontend

# Check service has both ports
kubectl get svc <release-name> -n <namespace>
# Should show ports: 3000,8000

# Test backend health
kubectl port-forward -n <namespace> deployment/<release-name> 8000:8000 &
curl http://localhost:8000/health
# Should return: {"status": "healthy"}

# Test frontend
kubectl port-forward -n <namespace> deployment/<release-name> 3000:3000 &
curl http://localhost:3000/
# Should return: HTML content
```

### Step 6: Test Application Functionality

1. Access the application via Ingress or port-forward
2. Verify frontend can communicate with backend
3. Test file upload and processing workflows
4. Check that data from v1.x is accessible

### Step 7: Restore Data (if needed)

If you need to restore from backup:

```bash
# Copy backup to pod
kubectl cp ./docetl-backup-<date>.tar.gz <release-name>-<pod-hash>:/tmp/backup.tar.gz -n <namespace>

# Restore data
kubectl exec -n <namespace> deployment/<release-name> -c backend -- \
  tar xzf /tmp/backup.tar.gz -C /

# Verify data restored
kubectl exec -n <namespace> deployment/<release-name> -c backend -- \
  ls -la /docetl-data
```

## Rollback Instructions

If issues occur, rollback to v1.x:

### Option 1: Helm Rollback (if upgrading)

```bash
helm rollback <release-name> -n <namespace>
```

### Option 2: Fresh v1.x Installation

```bash
# Uninstall v2.0.0
helm uninstall <release-name> -n <namespace>

# Reinstall v1.x
helm install <release-name> oci://your-registry/docetl \
  --version 1.1.0 \
  -f values-v1-backup.yaml \
  --set secrets.openaiApiKey=<your-api-key> \
  -n <namespace>
```

## Important Notes

### Multi-Replica Production Deployments

If running multiple replicas (production), ensure:

1. **ReadWriteMany (RWX) Storage**: Use RWX-capable storage class
   ```yaml
   persistence:
     enabled: true
     accessMode: ReadWriteMany
     storageClass: "nfs-client"  # Example
   ```

2. **Storage Class Compatibility**: Verify your storage class supports RWX
   ```bash
   kubectl get storageclass
   kubectl describe storageclass <name>
   ```

3. **Pod Anti-Affinity**: Spread pods across nodes
   ```yaml
   affinity:
     podAntiAffinity:
       preferredDuringSchedulingIgnoredDuringExecution:
         - weight: 100
           podAffinityTerm:
             labelSelector:
               matchExpressions:
                 - key: app.kubernetes.io/name
                   operator: In
                   values:
                     - docetl
             topologyKey: kubernetes.io/hostname
   ```

### Ingress Configuration

Ingress configuration remains compatible - no changes needed to path routing. The chart automatically uses named ports (`frontend` and `backend`).

### External Service References

If external services reference DocETL services:

**OLD:**
```yaml
- http://<release-name>-backend:8000/api
- http://<release-name>-frontend:3000/
```

**NEW:**
```yaml
- http://<release-name>:8000/api
- http://<release-name>:3000/
```

Update external references to use the unified service name.

## Troubleshooting

### Issue: Pods stuck in `Pending` state

**Cause:** PVC not mounting correctly

**Solution:**
```bash
# Check PVC status
kubectl get pvc -n <namespace>

# Describe PVC for errors
kubectl describe pvc <release-name>-pvc -n <namespace>

# For multi-replica: Ensure storage class supports ReadWriteMany
kubectl get storageclass
```

### Issue: Backend container not starting

**Cause:** Incorrect environment variables or missing API key

**Solution:**
```bash
# Check backend container logs
kubectl logs -n <namespace> deployment/<release-name> -c backend

# Verify secret exists
kubectl get secret <release-name>-secret -n <namespace>

# Check configmap
kubectl get configmap <release-name>-config -n <namespace> -o yaml
```

### Issue: Frontend container not starting

**Cause:** npm or Next.js startup issue

**Solution:**
```bash
# Check frontend container logs
kubectl logs -n <namespace> deployment/<release-name> -c frontend

# Check if data volume is mounted
kubectl exec -n <namespace> deployment/<release-name> -c frontend -- \
  ls -la /docetl-data
```

### Issue: Frontend cannot reach backend

**Cause:** Localhost communication not working

**Solution:**
```bash
# Verify both containers in same pod
kubectl get pods -n <namespace> -o wide

# Test inter-container communication
kubectl exec -n <namespace> deployment/<release-name> -c frontend -- \
  curl -s http://localhost:8000/health

# Check configmap has localhost values
kubectl get configmap <release-name>-config -n <namespace> -o yaml | grep BACKEND
# Should show: NEXT_PUBLIC_BACKEND_HOST: "localhost"
```

### Issue: 502 Bad Gateway from Ingress

**Cause:** Service port misconfiguration

**Solution:**
```bash
# Verify service has both ports
kubectl get svc <release-name> -n <namespace> -o yaml

# Should show:
# ports:
# - name: frontend
#   port: 3000
#   targetPort: frontend
# - name: backend
#   port: 8000
#   targetPort: backend

# Test service endpoints
kubectl get endpoints <release-name> -n <namespace>
```

## FAQ

**Q: Can I still scale frontend and backend independently?**
A: No, in v2.0.0 they scale together. If you need independent scaling, stay on v1.x.

**Q: Will this affect my existing data?**
A: No, the PVC and data remain unchanged. Only the deployment architecture changes.

**Q: Do I need to change my Ingress configuration?**
A: No, Ingress configuration is automatically updated by the chart.

**Q: Can I use ReadWriteOnce storage with multiple replicas?**
A: No, you must use ReadWriteMany (RWX) storage for multi-replica deployments.

**Q: What happens if one container crashes?**
A: Kubernetes will restart the entire pod, affecting both containers. This is expected behavior for multi-container pods.

**Q: Can I monitor containers separately?**
A: Yes, use container-specific commands:
```bash
kubectl logs deployment/<name> -c backend
kubectl logs deployment/<name> -c frontend
```

## Support

If you encounter issues during migration:

1. Check the [troubleshooting section](#troubleshooting)
2. Review pod logs: `kubectl logs -n <namespace> deployment/<release-name> --all-containers`
3. File an issue at: https://github.com/ucbepic/docetl/issues
4. Include:
   - Helm chart version (2.0.0)
   - Kubernetes version
   - values.yaml (sanitize secrets)
   - Pod descriptions and logs
