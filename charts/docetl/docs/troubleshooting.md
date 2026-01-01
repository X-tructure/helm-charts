# DocETL Troubleshooting Guide

This guide helps you diagnose and resolve common issues with the DocETL Helm chart.

## Quick Diagnostics

Run these commands first to gather information:

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/instance=docetl

# View all resources
kubectl get all -l app.kubernetes.io/instance=docetl

# Describe problematic pod
kubectl describe pod <pod-name>

# Check events
kubectl get events --sort-by='.lastTimestamp' | grep docetl
```

## Common Issues

### 1. Pods Not Starting

#### Symptom: Pods stuck in `Pending` state

**Check resource availability:**

```bash
kubectl describe pod <pod-name> | grep -A 5 Events
```

**Possible causes:**

a) **Insufficient cluster resources**

```
Events:
  Warning  FailedScheduling  pod didn't trigger scale-up: 1 Insufficient cpu
```

**Solution**: Reduce resource requests or add nodes
```yaml
backend:
  resources:
    requests:
      cpu: 250m      # Reduce from 500m
      memory: 512Mi  # Reduce from 1Gi
```

b) **PVC not binding**

```
Events:
  Warning  FailedMount  MountVolume.SetUp failed: no volume plugin matched
```

**Solution**: Check PVC status
```bash
kubectl get pvc -l app.kubernetes.io/instance=docetl

# If Pending, check StorageClass
kubectl get storageclass
kubectl describe pvc docetl-data
```

Fix by specifying valid StorageClass:
```yaml
persistence:
  storageClassName: "standard"  # or your cluster's default
```

c) **Image pull errors**

```
Events:
  Warning  Failed  Failed to pull image "docetl:latest": rpc error: code = Unknown
```

**Solution**: Verify image exists and add pull secrets if needed
```bash
# Test image pull locally
docker pull your-registry/docetl:latest

# Add image pull secret
kubectl create secret docker-registry regcred \
  --docker-server=your-registry.com \
  --docker-username=your-user \
  --docker-password=your-password
```

```yaml
image:
  pullSecrets:
    - name: regcred
```

#### Symptom: Pods in `CrashLoopBackOff`

**Check logs:**

```bash
# Backend logs
kubectl logs -l app.kubernetes.io/component=backend --tail=100

# Frontend logs
kubectl logs -l app.kubernetes.io/component=frontend --tail=100

# Previous container logs (after crash)
kubectl logs <pod-name> --previous
```

**Possible causes:**

a) **Missing or invalid API key**

```
Error: OPENAI_API_KEY environment variable is not set
```

**Solution**: Verify secret exists and is correctly mounted
```bash
# Check secret
kubectl get secret docetl-secret -o yaml
kubectl get secret docetl-secret -o jsonpath='{.data.openai-api-key}' | base64 -d

# Verify environment variable in pod
kubectl exec <pod-name> -- env | grep OPENAI_API_KEY
```

b) **Backend not ready before frontend starts**

```
Error: Failed to connect to backend at http://docetl-backend:8000
```

**Solution**: Increase startup probe timing or check backend health
```bash
# Check backend health directly
kubectl port-forward svc/docetl-backend 8000:8000
curl http://localhost:8000/health
```

### 2. Backend/Frontend Connection Issues

#### Symptom: Frontend can't reach backend

**Check service DNS:**

```bash
# From within frontend pod
kubectl exec -it <frontend-pod> -- nslookup docetl-backend

# Test backend connectivity
kubectl exec -it <frontend-pod> -- wget -O- http://docetl-backend:8000/health
```

**Check backend environment variables:**

```bash
kubectl exec <backend-pod> -- env | grep BACKEND
```

**Verify CORS configuration:**

```bash
# Check backend logs for CORS errors
kubectl logs -l app.kubernetes.io/component=backend | grep -i cors
```

**Solution**: Ensure `BACKEND_ALLOW_ORIGINS` includes frontend URL
```yaml
backend:
  env:
    BACKEND_ALLOW_ORIGINS: "http://docetl-frontend:3000"
```

### 3. Persistence Issues

#### Symptom: Data not persisting after pod restart

**Check PVC mount:**

```bash
# Verify PVC is bound
kubectl get pvc

# Check mount point in pod
kubectl exec <backend-pod> -- df -h | grep docetl-data
kubectl exec <backend-pod> -- ls -la /docetl-data
```

**Check file permissions:**

```bash
kubectl exec <backend-pod> -- ls -ld /docetl-data
# Should show: drwxr-xr-x ... 1000 1000 ... /docetl-data
```

**Solution**: If permissions are wrong, adjust `fsGroup`
```yaml
podSecurityContext:
  fsGroup: 1000
```

#### Symptom: Multiple replicas can't access storage

```
Error: Multi-Attach error for volume "pvc-xxx"
```

**Solution**: Change to ReadWriteMany and use compatible storage
```yaml
backend:
  replicaCount: 3

persistence:
  accessMode: ReadWriteMany
  storageClassName: "nfs-client"  # Must support RWX
```

### 4. Ingress Issues

#### Symptom: Can't access application via Ingress

**Check Ingress resource:**

```bash
kubectl get ingress
kubectl describe ingress docetl

# Check Ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

**Verify DNS:**

```bash
nslookup docetl.example.com
dig docetl.example.com
```

**Test backend directly:**

```bash
# Port-forward to bypass Ingress
kubectl port-forward svc/docetl-frontend 3000:3000

# Access at http://localhost:3000
```

**Common fixes:**

a) **Ingress class not specified**
```yaml
ingress:
  className: nginx  # Add this
```

b) **TLS secret missing**
```bash
kubectl get secret docetl-tls
```

c) **Ingress controller not installed**
```bash
kubectl get pods -n ingress-nginx
```

### 5. TLS/Certificate Issues

#### Symptom: cert-manager not issuing certificates

**Check certificate status:**

```bash
kubectl get certificate
kubectl describe certificate docetl-tls

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
```

**Common issues:**

a) **HTTP-01 challenge failing**
```bash
kubectl get challenges
kubectl describe challenge <challenge-name>
```

**Solution**: Ensure port 80 is accessible for HTTP-01 validation

b) **Rate limit hit**

**Solution**: Use Let's Encrypt staging for testing
```yaml
ingress:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
```

### 6. Performance Issues

#### Symptom: Slow response times

**Check resource usage:**

```bash
kubectl top pods -l app.kubernetes.io/instance=docetl
kubectl top nodes
```

**Check for CPU/memory throttling:**

```bash
kubectl describe pod <pod-name> | grep -A 5 "Limits\|Requests"
```

**Solution**: Increase resource limits
```yaml
backend:
  resources:
    limits:
      cpu: 4000m      # Increase from 2000m
      memory: 8Gi     # Increase from 4Gi
```

#### Symptom: OpenAI API rate limits

**Check backend logs:**

```bash
kubectl logs -l app.kubernetes.io/component=backend | grep -i "rate limit"
```

**Solution**: Implement request queuing or upgrade OpenAI plan

## Debug Commands

### View Logs

```bash
# Backend logs (live)
kubectl logs -l app.kubernetes.io/component=backend -f

# Frontend logs (live)
kubectl logs -l app.kubernetes.io/component=frontend -f

# Last 100 lines
kubectl logs -l app.kubernetes.io/component=backend --tail=100

# Logs from crashed container
kubectl logs <pod-name> --previous

# All pods
kubectl logs -l app.kubernetes.io/instance=docetl --all-containers=true
```

### Interactive Debugging

```bash
# Shell into backend pod
kubectl exec -it <backend-pod> -- /bin/bash

# Shell into frontend pod
kubectl exec -it <frontend-pod> -- /bin/sh

# Run command in pod
kubectl exec <backend-pod> -- curl http://localhost:8000/health
```

### Network Debugging

```bash
# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup docetl-backend

# Test HTTP connectivity
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl -v http://docetl-backend:8000/health

# Check network policies
kubectl get networkpolicies
kubectl describe networkpolicy <policy-name>
```

### Resource Inspection

```bash
# Get all DocETL resources
kubectl get all,pvc,secret,ingress -l app.kubernetes.io/instance=docetl

# Describe deployment
kubectl describe deployment docetl-backend
kubectl describe deployment docetl-frontend

# Check resource usage
kubectl top pods
kubectl top nodes

# Check events (last 1 hour)
kubectl get events --field-selector involvedObject.name=<pod-name>
```

## Helm Test

Run Helm tests to verify connectivity:

```bash
helm test docetl

# View test logs
kubectl logs docetl-test-connection

# Clean up test pods
kubectl delete pod docetl-test-connection
```

## Reset and Reinstall

If all else fails, perform a clean reinstall:

```bash
# 1. Uninstall release
helm uninstall docetl

# 2. Delete PVC (if you want fresh data)
kubectl delete pvc -l app.kubernetes.io/instance=docetl

# 3. Delete any stuck resources
kubectl delete all -l app.kubernetes.io/instance=docetl --grace-period=0 --force

# 4. Reinstall
helm install docetl ./charts/docetl -f my-values.yaml
```

## Getting Help

If you're still experiencing issues:

1. **Gather diagnostic information:**
   ```bash
   # Create diagnostic bundle
   kubectl describe pods -l app.kubernetes.io/instance=docetl > pods.txt
   kubectl logs -l app.kubernetes.io/instance=docetl --all-containers=true > logs.txt
   kubectl get events --sort-by='.lastTimestamp' > events.txt
   helm get values docetl > values.txt
   ```

2. **Check documentation:**
   - [Configuration Reference](configuration.md)
   - [Security Guide](security.md)
   - [Installation Guide](installation.md)

3. **Search existing issues:**
   - [DocETL Issues](https://github.com/ucbepic/docetl/issues)
   - [Helm Chart Issues](https://github.com/X-tructure/helm-charts/issues)

4. **Create a new issue:**
   Include:
   - Kubernetes version: `kubectl version`
   - Helm version: `helm version`
   - Chart version: `helm list`
   - Full error messages and logs
   - Steps to reproduce
   - Your values.yaml (redact secrets!)

## Additional Resources

- [Kubernetes Debugging Guide](https://kubernetes.io/docs/tasks/debug/)
- [Helm Debugging Tips](https://helm.sh/docs/howto/charts_tips_and_tricks/)
- [DocETL Documentation](https://ucbepic.github.io/docetl)
