# Troubleshooting Guide

Common issues and solutions for the Obsidian Helm chart.

## Installation Issues

### Issue: Helm Repository Not Found

**Symptoms:**
```
Error: repo "obsidian" not found
```

**Solution:**
```bash
# Add the repository
helm repo add extreme_structure https://x-tructure.github.io/helm-charts
helm repo update
```

### Issue: PVC Stuck in Pending

**Symptoms:**
```
PersistentVolumeClaim is in Pending state
```

**Diagnosis:**
```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

**Common Causes & Solutions:**

1. **No storage provisioner:**
   ```bash
   # Check available storage classes
   kubectl get storageclass
   
   # Use an existing storage class
   helm upgrade my-obsidian obsidian/obsidian \
     --set persistence.storageClass=standard
   ```

2. **Insufficient storage:**
   - Reduce the requested size or add more storage to your cluster

3. **Access mode not supported:**
   - Some storage classes don't support ReadWriteOnce
   - Check your storage class capabilities

### Issue: Pod CrashLoopBackOff

**Diagnosis:**
```bash
# Check pod status
kubectl get pods -n <namespace>

# View pod logs
kubectl logs <pod-name> -n <namespace>

# Describe pod for events
kubectl describe pod <pod-name> -n <namespace>
```

**Common Causes:**

1. **Insufficient resources:**
   ```bash
   # Reduce resource requests
   helm upgrade my-obsidian obsidian/obsidian \
     --set resources.requests.cpu=250m \
     --set resources.requests.memory=256Mi
   ```

2. **Permission issues:**
   ```bash
   # Check security context
   helm get values my-obsidian
   
   # Ensure PUID/PGID match volume permissions
   ```

3. **Volume mount issues:**
   - Check PVC is bound
   - Verify volume permissions

## Application Issues

### Issue: Cannot Access Web Interface

**Symptoms:**
Connection refused or timeout when accessing the application.

**Solutions:**

1. **Check pod is running:**
   ```bash
   kubectl get pods -n <namespace> -l app.kubernetes.io/name=obsidian
   ```

2. **Verify service:**
   ```bash
   kubectl get svc -n <namespace>
   ```

3. **Test port-forward:**
   ```bash
   kubectl port-forward svc/<service-name> 3001:3001 -n <namespace>
   # Try accessing https://localhost:3001
   ```

4. **Check firewall rules** (for NodePort/LoadBalancer)

5. **Verify Ingress configuration:**
   ```bash
   kubectl get ingress -n <namespace>
   kubectl describe ingress <ingress-name> -n <namespace>
   ```

### Issue: SSL Certificate Warnings

**Symptoms:**
Browser shows "Your connection is not private" or SSL certificate errors.

**Solutions:**

1. **Expected for self-signed cert:** 
   - Click "Advanced" and "Proceed" (for testing only)

2. **For production, use valid certificates:**
   ```yaml
   ingress:
     enabled: true
     annotations:
       cert-manager.io/cluster-issuer: letsencrypt-prod
     tls:
       - secretName: obsidian-tls
         hosts:
           - your-domain.com
   ```

### Issue: Authentication Not Working

**Symptoms:**
No authentication prompt appears, or credentials don't work.

**Diagnosis:**
```bash
# Check environment variables
kubectl exec <pod-name> -n <namespace> -- env | grep -E 'CUSTOM_USER|PASSWORD'
```

**Solutions:**

1. **Verify auth configuration:**
   ```bash
   helm get values my-obsidian | grep -A5 auth
   ```

2. **Update credentials:**
   ```bash
   helm upgrade my-obsidian obsidian/obsidian \
     --set auth.enabled=true \
     --set auth.username=admin \
     --set auth.password=newpassword
   ```

3. **Restart pod:**
   ```bash
   kubectl delete pod -n <namespace> -l app.kubernetes.io/name=obsidian
   ```

### Issue: Slow Performance or Rendering Issues

**Symptoms:**
Laggy interface, slow rendering, or application freezes.

**Solutions:**

1. **Increase shared memory:**
   ```yaml
   shmSize: 2Gi  # Default is 1Gi
   ```

2. **Increase resources:**
   ```yaml
   resources:
     requests:
       cpu: 1000m
       memory: 1Gi
     limits:
       cpu: 4000m
       memory: 4Gi
   ```

3. **Enable GPU acceleration:**
   ```yaml
   extraEnv:
     - name: DRINODE
       value: "/dev/dri/renderD128"
   ```

4. **Optimize video encoding:**
   ```yaml
   selkies:
     encoder: "x264enc"
     framerate: "30"
   ```

5. **Check node resources:**
   ```bash
   kubectl top nodes
   kubectl top pods -n <namespace>
   ```

### Issue: Files Not Persisting

**Symptoms:**
Data is lost after pod restart.

**Diagnosis:**
```bash
# Check PVC
kubectl get pvc -n <namespace>

# Verify mount
kubectl exec <pod-name> -n <namespace> -- df -h /config
```

**Solutions:**

1. **Ensure persistence is enabled:**
   ```yaml
   persistence:
     enabled: true
   ```

2. **Check PVC status:**
   ```bash
   kubectl describe pvc <pvc-name> -n <namespace>
   ```

3. **Verify volume mounts:**
   ```bash
   kubectl describe pod <pod-name> -n <namespace> | grep -A10 Mounts
   ```

## Ingress Issues

### Issue: Ingress Not Working

**Symptoms:**
Cannot access application via Ingress hostname.

**Diagnosis:**
```bash
# Check Ingress
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>

# Check Ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

**Solutions:**

1. **Verify Ingress class:**
   ```bash
   kubectl get ingressclass
   ```

2. **Check backend protocol annotation:**
   ```yaml
   ingress:
     annotations:
       nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
   ```

3. **Verify DNS resolution:**
   ```bash
   nslookup your-domain.com
   ```

4. **Check Ingress controller is running:**
   ```bash
   kubectl get pods -n ingress-nginx
   ```

### Issue: Ingress Backend Errors

**Symptoms:**
502 Bad Gateway or 503 Service Unavailable.

**Solutions:**

1. **Ensure backend-protocol is HTTPS:**
   ```yaml
   ingress:
     annotations:
       nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
   ```

2. **Check pod is ready:**
   ```bash
   kubectl get pods -n <namespace>
   ```

3. **Verify service endpoints:**
   ```bash
   kubectl get endpoints -n <namespace>
   ```

## Health Check Issues

### Issue: Pod Keeps Restarting (Liveness Probe Failing)

**Diagnosis:**
```bash
kubectl describe pod <pod-name> -n <namespace>
# Look for "Liveness probe failed"
```

**Solutions:**

1. **Increase initial delay:**
   ```yaml
   livenessProbe:
     initialDelaySeconds: 60  # Default is 30
   ```

2. **Increase timeout:**
   ```yaml
   livenessProbe:
     timeoutSeconds: 10  # Default is 5
   ```

3. **Check application is responding:**
   ```bash
   kubectl exec <pod-name> -n <namespace> -- curl -k https://localhost:3001/
   ```

### Issue: Pod Never Becomes Ready (Readiness Probe Failing)

**Solutions:**

1. **Increase startup probe threshold:**
   ```yaml
   startupProbe:
     failureThreshold: 30  # Default is 20 (300 seconds total)
   ```

2. **Check application startup logs:**
   ```bash
   kubectl logs <pod-name> -n <namespace>
   ```

## Upgrade Issues

### Issue: Upgrade Fails with PVC

**Symptoms:**
```
Error: cannot patch ... field is immutable
```

**Solutions:**

See [Upgrade Guide](upgrade.md) for PVC-related upgrade procedures.

## Network Issues

### Issue: WebCodecs Not Working

**Symptoms:**
Video/audio features not working, browser console shows WebCodecs errors.

**Solution:**
WebCodecs requires HTTPS. Ensure you're accessing via:
- HTTPS (port 3001), not HTTP (port 3000)
- Valid TLS certificate (or accept self-signed for testing)

### Issue: Slow Network Performance

**Solutions:**

1. **Adjust video quality:**
   ```yaml
   selkies:
     encoder: "jpeg"  # Lower quality, less CPU
     framerate: "15"
   ```

2. **Check network bandwidth:**
   - Ensure adequate bandwidth between client and cluster

## Getting Help

If you're still experiencing issues:

1. **Collect diagnostic information:**
   ```bash
   # Pod status
   kubectl get pods -n <namespace> -l app.kubernetes.io/name=obsidian
   
   # Pod logs
   kubectl logs <pod-name> -n <namespace> > pod-logs.txt
   
   # Pod description
   kubectl describe pod <pod-name> -n <namespace> > pod-description.txt
   
   # Helm values
   helm get values my-obsidian > values.yaml
   
   # Events
   kubectl get events -n <namespace> --sort-by='.lastTimestamp'
   ```

2. **Check documentation:**
   - [Configuration Guide](configuration.md)
   - [Installation Guide](installation.md)
   - [LinuxServer.io Obsidian docs](https://docs.linuxserver.io/images/docker-obsidian)

3. **Open an issue:**
   - GitHub: https://github.com/X-tructure/helm-charts/issues
   - Include diagnostic information from step 1
   - Describe what you expected vs. what happened
   - Include your Helm values (redact sensitive info)
