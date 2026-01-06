# Troubleshooting Guide

Common issues and solutions for the Docling-Serve Helm chart.

## Table of Contents

- [Pod Issues](#pod-issues)
- [API Issues](#api-issues)
- [Storage Issues](#storage-issues)
- [Model Persistence Issues](#model-persistence-issues)
- [GPU Issues](#gpu-issues)
- [Performance Issues](#performance-issues)
- [Authentication Issues](#authentication-issues)
- [Network Issues](#network-issues)

## Pod Issues

### Pod Not Starting

**Symptoms:**
- Pod stuck in `Pending` state
- Pod in `CrashLoopBackOff`
- Pod in `ImagePullBackOff`

**Diagnosis:**
```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=docling-serve

# Describe pod for events
kubectl describe pod -l app.kubernetes.io/name=docling-serve

# Check logs
kubectl logs -l app.kubernetes.io/name=docling-serve
```

**Solutions:**

1. **Insufficient Resources**
   ```yaml
   # Reduce resource requirements
   resources:
     requests:
       cpu: 100m
       memory: 512Mi
   ```

2. **Image Pull Errors**
   ```bash
   # Verify image exists
   docker pull ghcr.io/docling-project/docling-serve-cpu:1.9.0

   # Check imagePullSecrets if using private registry
   kubectl get secret my-registry-secret
   ```

3. **Node Selector Mismatch**
   ```bash
   # Check node labels
   kubectl get nodes --show-labels

   # Remove or adjust nodeSelector
   helm upgrade my-docling --set nodeSelector=null
   ```

### Startup Probe Timeout

**Symptoms:**
- Pod restarts after 200 seconds
- Logs show model loading in progress

**Cause:** Model loading takes longer than startup probe allows

**Solutions:**

1. **Increase Startup Probe Timeout**
   ```yaml
   startupProbe:
     failureThreshold: 40  # 400 seconds
   ```

2. **Disable Model Loading at Boot**
   ```yaml
   env:
     loadModelsAtBoot: "false"
   ```

#### Understanding Probe Configuration for Different Scenarios

The appropriate startup probe configuration depends on your deployment scenario:

| Scenario | Model Loading | Typical Startup Time | Recommended Window |
|----------|---------------|---------------------|-------------------|
| Basic (CPU, minimal resources) | Disabled | 20-40s | 65s (see values-basic.yaml) |
| Production (CPU, good resources) | Enabled | 60-120s | 205s (default) |
| GPU (CUDA, high resources) | Enabled | 90-180s | 315s (see values-gpu.yaml) |

**Key Relationship:**
- When `loadModelsAtBoot: "false"` → Models load on first request, startup is fast
- When `loadModelsAtBoot: "true"` → Models load during startup, need longer window

**Calculating Startup Window:**
```
Total window = initialDelaySeconds + (failureThreshold × periodSeconds)
```

**Diagnostic Steps:**

1. **Check actual startup time:**
   ```bash
   kubectl logs <pod-name> | grep "Application startup complete"
   ```

2. **Monitor probe failures:**
   ```bash
   kubectl describe pod <pod-name> | grep -A 5 "Liveness\|Readiness\|Startup"
   ```

3. **Adjust failureThreshold if needed:**
   - Calculate observed startup time from logs
   - Add 30-50% buffer for safety
   - Update failureThreshold: `(desired_window - initialDelay) / periodSeconds`

### Memory Issues

**Symptoms:**
- Pod killed with `OOMKilled` status
- Out of memory errors in logs

**Solutions:**

1. **Increase Memory Limits**
   ```yaml
   resources:
     limits:
       memory: 8Gi
   ```

2. **Use Disk-backed Scratch Storage**
   ```yaml
   scratch:
     emptyDir:
       medium: ""  # Don't use Memory
   ```

3. **Reduce Workers**
   ```yaml
   env:
     engineLocalNumWorkers: "1"
     numThreads: "2"
   ```

## API Issues

### API Not Responding

**Symptoms:**
- Connection refused errors
- Timeout accessing API

**Diagnosis:**
```bash
# Check service endpoints
kubectl get endpoints -l app.kubernetes.io/name=docling-serve

# Test from within cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://my-docling-serve:5001/version

# Check pod logs
kubectl logs -l app.kubernetes.io/name=docling-serve
```

**Solutions:**

1. **Service Not Ready**
   ```bash
   # Wait for readiness probe to pass
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=docling-serve --timeout=300s
   ```

2. **Port Configuration Mismatch**
   ```bash
   # Verify port configuration
   kubectl get svc my-docling-serve -o yaml | grep port
   ```

### 500 Internal Server Error

**Diagnosis:**
```bash
# Check application logs
kubectl logs -l app.kubernetes.io/name=docling-serve --tail=100
```

**Common Causes:**
- Model loading failed
- Insufficient memory during processing
- Invalid environment variable configuration

**Solutions:**
```yaml
# Ensure models loaded successfully
env:
  loadModelsAtBoot: "true"

# Increase resources
resources:
  limits:
    memory: 4Gi
```

### 401 Unauthorized

**Cause:** API key mismatch or missing

**Solutions:**

1. **Retrieve API Key**
   ```bash
   kubectl get secret my-docling-serve-api-key \
     -o jsonpath='{.data.api-key}' | base64 -d
   ```

2. **Test with API Key**
   ```bash
   API_KEY=$(kubectl get secret my-docling-serve-api-key -o jsonpath='{.data.api-key}' | base64 -d)
   curl -H "X-API-Key: $API_KEY" http://localhost:5001/version
   ```

3. **Disable Authentication**
   ```yaml
   apiKey:
     enabled: false
   ```

## Storage Issues

### PVC Not Binding

**Symptoms:**
- PVC stuck in `Pending` state
- Pod can't start due to volume mount failure

**Diagnosis:**
```bash
# Check PVC status
kubectl get pvc

# Describe PVC
kubectl describe pvc my-docling-serve-scratch

# Check available PVs
kubectl get pv
```

**Solutions:**

1. **No Storage Class Available**
   ```yaml
   scratch:
     pvc:
       storageClass: "standard"  # Or your cluster's default
   ```

2. **Insufficient Storage**
   ```bash
   # Check node disk space
   kubectl describe nodes | grep -A 5 "Allocated resources"
   ```

3. **Use emptyDir Instead**
   ```yaml
   scratch:
     persistent: false
   ```

### Disk Full Errors

**Symptoms:**
- "No space left on device" in logs
- Conversion failures

**Solutions:**

1. **Increase PVC Size**
   ```bash
   # Some storage classes support volume expansion
   kubectl patch pvc my-docling-serve-scratch -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
   ```

2. **Enable Result Cleanup**
   ```yaml
   env:
     singleUseResults: "true"
   ```

## Model Persistence Issues

### Model Download Job Failed

**Symptoms:**
- Job shows `BackoffLimitExceeded` status
- Job pods in `Error` or `CrashLoopBackOff` state

**Diagnosis:**
```bash
# Check Job status
kubectl get job -l app.kubernetes.io/component=model-download

# Check Job logs
kubectl logs job/<release-name>-docling-serve-model-download

# Describe Job for events
kubectl describe job <release-name>-docling-serve-model-download
```

**Common Causes:**

1. **Network Connectivity Issues**
   ```bash
   # Test network from Job pod
   kubectl run -it --rm test --image=curlimages/curl --restart=Never -- \
     curl -I https://huggingface.co
   ```

2. **Insufficient PVC Size**
   ```yaml
   # Increase PVC size
   models:
     pvc:
       size: 30Gi  # Increase from default 15Gi
   ```

3. **Proxy/Firewall Blocking Downloads**
   ```yaml
   # Add proxy environment variables
   models:
     job:
       extraEnv:
         - name: HTTP_PROXY
           value: "http://proxy.example.com:8080"
         - name: HTTPS_PROXY
           value: "http://proxy.example.com:8080"
   ```

**Solutions:**

1. **Increase Backoff Limit**
   ```yaml
   models:
     job:
       backoffLimit: 5  # Increase from default 3
   ```

2. **Delete and Recreate Job**
   ```bash
   kubectl delete job <release-name>-docling-serve-model-download
   helm upgrade <release-name> ./charts/docling-serve --reuse-values
   ```

### Insufficient Model Storage

**Symptoms:**
- "No space left on device" in Job logs
- Job fails during model download
- PVC usage at 100%

**Diagnosis:**
```bash
# Check PVC usage
kubectl exec deployment/<release-name>-docling-serve -- df -h /modelcache

# Check PVC size
kubectl get pvc <release-name>-docling-serve-models -o jsonpath='{.spec.resources.requests.storage}'
```

**Solutions:**

1. **Increase PVC Size**
   ```yaml
   models:
     pvc:
       size: 30Gi  # For all models
   ```

   Storage sizing guide:
   - Basic models (layout, tableformer, code_formula, picture_classifier): 10-15Gi
   - With vision models (+ smolvlm, granite_vision): 20-25Gi
   - All models (+ easyocr): 25-30Gi

2. **Select Fewer Models**
   ```yaml
   models:
     download:
       - layout
       - tableformer
       # Remove unnecessary models
   ```

3. **Use Existing PVC with More Space**
   ```yaml
   models:
     pvc:
       existingClaim: "my-large-pvc"
   ```

### Models Not Loading in Deployment

**Symptoms:**
- Deployment runs but models not found
- API returns errors about missing models
- Application uses container-baked models instead of PVC

**Diagnosis:**
```bash
# Check if DOCLING_SERVE_ARTIFACTS_PATH is set
kubectl exec deployment/<release-name>-docling-serve -- env | grep ARTIFACTS_PATH

# List models in PVC
kubectl exec deployment/<release-name>-docling-serve -- ls -la /modelcache

# Check volume mount
kubectl describe pod -l app.kubernetes.io/name=docling-serve | grep -A 10 "Mounts:"

# Verify PVC is bound
kubectl get pvc <release-name>-docling-serve-models
```

**Solutions:**

1. **Verify Model Persistence is Enabled**
   ```yaml
   models:
     enabled: true  # Must be true
   ```

2. **Check Job Completed Successfully**
   ```bash
   kubectl get job <release-name>-docling-serve-model-download
   # Should show COMPLETIONS: 1/1
   ```

3. **Restart Deployment**
   ```bash
   kubectl rollout restart deployment/<release-name>-docling-serve
   ```

### Job Stuck in Pending

**Symptoms:**
- Job pod never starts
- Pod stuck in `Pending` state

**Diagnosis:**
```bash
# Check Job pod events
kubectl describe pod -l app.kubernetes.io/component=model-download

# Check PVC status
kubectl get pvc <release-name>-docling-serve-models
```

**Solutions:**

1. **PVC Not Bound**
   - See [PVC Not Binding](#pvc-not-binding) section

2. **Resource Constraints**
   ```yaml
   models:
     job:
       resources:
         requests:
           cpu: 250m  # Reduce from 500m
           memory: 512Mi  # Reduce from 1Gi
   ```

3. **Node Selector Mismatch**
   ```yaml
   models:
     job:
       nodeSelector: {}  # Remove restrictions
   ```

### Multiple Pods with ReadWriteOnce PVC

**Symptoms:**
- Cannot scale deployment beyond 1 replica
- Pods fail to mount models PVC

**Cause:** Models PVC uses `ReadWriteOnce` access mode

**Solutions:**

1. **Use ReadOnly Mount** (Default behavior)
   - Models are mounted read-only in deployment
   - RWO PVC can be shared when mounted read-only
   - This is already configured correctly

2. **Scale After Model Download**
   ```bash
   # Wait for Job to complete
   kubectl wait --for=condition=complete job/<release-name>-docling-serve-model-download --timeout=600s

   # Then scale
   kubectl scale deployment/<release-name>-docling-serve --replicas=3
   ```

3. **Use ReadWriteMany (Advanced)**
   ```yaml
   models:
     pvc:
       accessMode: ReadWriteMany
       storageClass: "nfs-client"  # Or other RWX-capable storage
   ```

### Updating Models

**How to update models to newer versions:**

1. **Delete Existing Job**
   ```bash
   kubectl delete job <release-name>-docling-serve-model-download
   ```

2. **Delete PVC (Models will be lost)**
   ```bash
   kubectl delete pvc <release-name>-docling-serve-models
   ```

3. **Upgrade Chart**
   ```bash
   helm upgrade <release-name> ./charts/docling-serve --reuse-values
   ```

4. **Wait for New Download**
   ```bash
   kubectl wait --for=condition=complete job/<release-name>-docling-serve-model-download
   ```

**Alternative: Update in-place (Advanced)**
```bash
# Scale down deployment
kubectl scale deployment/<release-name>-docling-serve --replicas=0

# Run update Job manually
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: model-update
spec:
  template:
    spec:
      containers:
      - name: updater
        image: ghcr.io/docling-project/docling-serve-cpu:latest
        command: ["/bin/sh", "-c", "docling-tools models download --output-dir=/modelcache --all --force"]
        volumeMounts:
        - name: models
          mountPath: /modelcache
      volumes:
      - name: models
        persistentVolumeClaim:
          claimName: <release-name>-docling-serve-models
      restartPolicy: Never
EOF

# Scale deployment back up
kubectl scale deployment/<release-name>-docling-serve --replicas=1
```

## GPU Issues

### GPU Not Detected

**Symptoms:**
- CUDA errors in logs
- Running on CPU despite GPU configuration

**Diagnosis:**
```bash
# Check GPU resources on nodes
kubectl describe nodes | grep nvidia.com/gpu

# Check pod GPU allocation
kubectl describe pod -l app.kubernetes.io/name=docling-serve | grep nvidia.com/gpu
```

**Solutions:**

1. **Install GPU Operator**
   ```bash
   # Verify GPU operator is running
   kubectl get pods -n gpu-operator-resources
   ```

2. **Verify GPU Image**
   ```yaml
   image:
     repository: ghcr.io/docling-project/docling-serve-cu126
   env:
     device: "cuda"
   ```

3. **Check Node Labels**
   ```bash
   kubectl label nodes <node-name> nvidia.com/gpu.present=true
   ```

### CUDA Version Mismatch

**Symptoms:**
- "CUDA driver version is insufficient" errors

**Solutions:**

1. **Use Correct Image**
   - CUDA 12.6: `docling-serve-cu126`
   - CUDA 12.8: `docling-serve-cu128`

2. **Verify Driver Version**
   ```bash
   # On GPU node
   nvidia-smi
   ```

Required: Driver ≥550.54.14

## Performance Issues

### Slow Document Processing

**Diagnosis:**
```bash
# Check resource utilization
kubectl top pods -l app.kubernetes.io/name=docling-serve

# Check for CPU throttling
kubectl describe pod -l app.kubernetes.io/name=docling-serve | grep -A 5 "Limits"
```

**Solutions:**

1. **Increase Workers and Threads**
   ```yaml
   env:
     uvicornWorkers: "2"
     engineLocalNumWorkers: "4"
     numThreads: "8"
   ```

2. **Use Memory-Backed Storage**
   ```yaml
   scratch:
     emptyDir:
       medium: "Memory"
   ```

3. **Increase CPU Resources**
   ```yaml
   resources:
     requests:
       cpu: 1
     limits:
       cpu: 4
   ```

### Model Loading Takes Too Long

**Solutions:**

1. **Disable Model Loading at Boot**
   ```yaml
   env:
     loadModelsAtBoot: "false"
   ```

2. **Use Persistent Storage for Models**
   ```yaml
   scratch:
     persistent: true
     pvc:
       size: 20Gi
   ```

## Authentication Issues

### API Key Not Working

**Diagnosis:**
```bash
# Verify secret exists
kubectl get secret my-docling-serve-api-key

# Check secret value
kubectl get secret my-docling-serve-api-key -o yaml
```

**Solutions:**

1. **Recreate Secret**
   ```bash
   kubectl delete secret my-docling-serve-api-key
   helm upgrade my-docling --set apiKey.enabled=true --reuse-values
   ```

2. **Use Base64 Decoded Value**
   ```bash
   # Secret is base64 encoded
   kubectl get secret my-docling-serve-api-key \
     -o jsonpath='{.data.api-key}' | base64 -d
   ```

## Network Issues

### Ingress Not Working

**Symptoms:**
- 404 errors
- Can't access via domain

**Diagnosis:**
```bash
# Check ingress status
kubectl get ingress

# Describe ingress
kubectl describe ingress my-docling-serve

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

**Solutions:**

1. **Verify Ingress Controller**
   ```bash
   kubectl get pods -n ingress-nginx
   ```

2. **Check DNS**
   ```bash
   nslookup docling.example.com
   ```

3. **Review Annotations**
   ```yaml
   ingress:
     annotations:
       nginx.ingress.kubernetes.io/proxy-body-size: "100m"
   ```

### Port Forward Fails

**Solutions:**

1. **Check Pod is Running**
   ```bash
   kubectl get pods -l app.kubernetes.io/name=docling-serve
   ```

2. **Use Correct Port**
   ```bash
   kubectl port-forward svc/my-docling-serve 5001:5001
   # Not 8080 or other port
   ```

## Debugging Tips

### Enable Debug Logging

```yaml
extraEnv:
  - name: LOG_LEVEL
    value: "DEBUG"
```

### Check All Resources

```bash
# Get all resources for the release
helm list
helm status my-docling-serve
kubectl get all -l app.kubernetes.io/instance=my-docling-serve
```

### Exec into Pod

```bash
POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=docling-serve -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -- /bin/bash

# Check disk usage
df -h

# Check processes
ps aux

# Check environment variables
env | grep DOCLING
```

## Getting Help

If you can't resolve the issue:

1. **Collect Debug Information**
   ```bash
   # Pod description
   kubectl describe pod -l app.kubernetes.io/name=docling-serve > pod-describe.txt

   # Pod logs
   kubectl logs -l app.kubernetes.io/name=docling-serve > pod-logs.txt

   # Helm values
   helm get values my-docling-serve > values.txt
   ```

2. **Report Issue**
   - [Chart Issues](https://github.com/X-tructure/helm-charts/issues)
   - [Docling-Serve Issues](https://github.com/docling-project/docling-serve/issues)

3. **Include Information**
   - Kubernetes version
   - Helm version
   - Chart version
   - Values configuration
   - Error messages and logs
