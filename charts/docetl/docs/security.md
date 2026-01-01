# DocETL Security Best Practices

This guide covers security best practices for deploying DocETL in production environments.

## ⚠️ Critical Security Warning

**NEVER commit actual API keys to version control!**

The chart provides several secure methods for managing sensitive credentials. Always use one of the recommended approaches below.

## API Key Management

The OpenAI API key is required for DocETL to function. Choose the appropriate method based on your environment:

### Option 1: Command-line Override (Development Only)

**Use Case**: Quick testing, local development, ephemeral environments

**Security Level**: ⚠️ Low - Command history exposure

```bash
helm install docetl ./charts/docetl \
  --set backend.openaiApiKey=sk-your-actual-key \
  --set image.repository=your-registry/docetl
```

**Pros**: Simple, fast
**Cons**: API key visible in command history, shell logs, and process listings

### Option 2: Separate Values File (Development/Staging)

**Use Case**: Development teams with multiple environments

**Security Level**: ⚠️ Medium - Requires discipline

Create `my-secrets.yaml` (add to `.gitignore`):

```yaml
backend:
  openaiApiKey: "sk-your-actual-key"
```

Then install:

```bash
helm install docetl ./charts/docetl \
  -f values.yaml \
  -f my-secrets.yaml
```

**Important**: Ensure `my-secrets.yaml` is in `.gitignore`:

```bash
echo "my-secrets.yaml" >> .gitignore
```

**Pros**: Clean separation, supports multiple environments
**Cons**: Relies on developers not committing secrets

### Option 3: External Secret Management (Production Recommended)

**Use Case**: Production environments, enterprise deployments

**Security Level**: ✅ High - Industry standard

#### Using Sealed Secrets

[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) encrypts secrets that can be safely committed to Git.

```bash
# Install Sealed Secrets controller (one-time setup)
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Create a sealed secret
echo -n 'sk-your-actual-key' | kubectl create secret generic docetl-secret \
  --dry-run=client --from-file=openai-api-key=/dev/stdin -o yaml | \
  kubeseal -o yaml > sealed-secret.yaml

# Commit sealed-secret.yaml to Git (it's encrypted!)
git add sealed-secret.yaml
git commit -m "Add DocETL sealed secret"

# Apply the sealed secret
kubectl apply -f sealed-secret.yaml

# Install DocETL (secret already exists in cluster)
helm install docetl ./charts/docetl --set backend.openaiApiKey=""
```

**Pros**: Secrets encrypted in Git, GitOps-friendly, audit trail
**Cons**: Requires Sealed Secrets controller

#### Using HashiCorp Vault

[Vault](https://www.vaultproject.io/) provides centralized secret management with access control.

```bash
# Store secret in Vault
vault kv put secret/docetl/openai-api-key value=sk-your-actual-key

# Option A: Use Vault Secrets Operator
# Install operator: https://developer.hashicorp.com/vault/docs/platform/k8s/vso

# Option B: Use Vault CSI Driver
# Install driver: https://developer.hashicorp.com/vault/docs/platform/k8s/csi
```

Example VaultStaticSecret resource:

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: docetl-secret
spec:
  vaultAuthRef: default
  mount: secret
  path: docetl/openai-api-key
  destination:
    name: docetl-secret
    create: true
  refreshAfter: 1h
```

**Pros**: Enterprise-grade, audit logs, dynamic secrets, access policies
**Cons**: Complex setup, requires Vault infrastructure

#### Using AWS Secrets Manager

[External Secrets Operator](https://external-secrets.io/) syncs cloud provider secrets to Kubernetes.

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace

# Store secret in AWS Secrets Manager
aws secretsmanager create-secret \
  --name docetl/openai-api-key \
  --secret-string sk-your-actual-key \
  --region us-east-1
```

Example ExternalSecret resource:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: docetl-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  target:
    name: docetl-secret
    creationPolicy: Owner
  data:
    - secretKey: openai-api-key
      remoteRef:
        key: docetl/openai-api-key
```

**Pros**: Cloud-native, integrates with IAM, automatic rotation
**Cons**: Cloud provider lock-in, requires External Secrets Operator

#### Using Google Secret Manager / Azure Key Vault

Similar to AWS, use External Secrets Operator with the appropriate provider:

- **GCP**: [SecretStore for Google Secret Manager](https://external-secrets.io/latest/provider/google-secrets-manager/)
- **Azure**: [SecretStore for Azure Key Vault](https://external-secrets.io/latest/provider/azure-key-vault/)

### Option 4: Create Secret Manually

**Use Case**: Simple production deployments, manual secret management

**Security Level**: ✅ Medium-High - Direct Kubernetes secret

```bash
kubectl create secret generic docetl-secret \
  --from-literal=openai-api-key=sk-your-actual-key \
  --namespace docetl

# Install without setting secret in values
helm install docetl ./charts/docetl --set backend.openaiApiKey=""
```

**Pros**: Simple, no extra tools required
**Cons**: Manual secret rotation, no GitOps support

## TLS Configuration

**Always enable TLS for production deployments** to encrypt traffic and protect credentials.

### Using cert-manager (Recommended)

[cert-manager](https://cert-manager.io/) automates certificate management with Let's Encrypt.

```bash
# Install cert-manager (one-time setup)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Create ClusterIssuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

Configure in `values.yaml`:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: docetl.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    enabled: true
    secretName: docetl-tls
```

### Manual Certificate

For custom certificates:

```bash
# Create TLS secret
kubectl create secret tls docetl-tls \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key \
  --namespace docetl

# Deploy with TLS
helm install docetl ./charts/docetl -f examples/values-prod.yaml
```

## Additional Security Measures

### 1. Use Specific Image Tags

**Never use `latest` in production** - it's unpredictable and not reproducible.

```yaml
image:
  repository: myregistry/docetl
  tag: v1.2.3  # ✅ Good: Specific version
  # tag: latest  # ❌ Bad: Unpredictable
```

### 2. Enable Pod Security Standards

Apply [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) at the namespace level:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: docetl
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 3. Implement Network Policies

Restrict pod-to-pod communication with [NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: docetl-backend
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/component: frontend
      ports:
        - protocol: TCP
          port: 8000
  egress:
    - to:
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 443  # Allow HTTPS to OpenAI API
```

### 4. Regular Security Scans

Scan Docker images for vulnerabilities:

```bash
# Using Trivy
trivy image your-registry/docetl:v1.2.3

# Using Grype
grype your-registry/docetl:v1.2.3
```

Integrate into CI/CD:

```yaml
# GitHub Actions example
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'your-registry/docetl:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
```

### 5. Least Privilege

The chart already runs as non-root (UID 65534/nobody):

```yaml
securityContext:
  runAsUser: 65534
  runAsNonRoot: true
  capabilities:
    drop:
      - ALL
```

**Do not modify these settings** unless absolutely necessary.

### 6. Resource Limits

Prevent resource exhaustion attacks:

```yaml
backend:
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m      # ✅ Prevents CPU monopolization
      memory: 4Gi     # ✅ Prevents memory exhaustion
```

### 7. RBAC Configuration

Minimize ServiceAccount permissions:

```yaml
serviceAccount:
  create: true
  annotations:
    # Workload Identity (GKE)
    iam.gke.io/gcp-service-account: docetl@project.iam.gserviceaccount.com
    # IAM Roles for Service Accounts (EKS)
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/docetl
```

## Security Checklist

Before deploying to production:

- [ ] API keys stored in external secret manager (Sealed Secrets, Vault, etc.)
- [ ] TLS enabled with valid certificates (cert-manager or manual)
- [ ] Specific image tags (not `latest`)
- [ ] Pod Security Standards enforced
- [ ] Network Policies implemented
- [ ] Container images scanned for vulnerabilities
- [ ] Resource limits configured
- [ ] RBAC permissions minimized
- [ ] Regular security updates scheduled
- [ ] Audit logging enabled (Kubernetes, Vault, etc.)

## Compliance Considerations

### GDPR / Data Privacy

- Store persistent data in compliant regions
- Implement data retention policies
- Enable audit logging for access tracking
- Use encryption at rest for PersistentVolumes

### SOC 2 / ISO 27001

- Use external secret management with audit trails
- Implement network segmentation with NetworkPolicies
- Regular vulnerability scanning in CI/CD
- Document security procedures

## Incident Response

If an API key is compromised:

1. **Immediately revoke** the compromised key at OpenAI
2. **Rotate** to a new API key
3. **Update** the Kubernetes secret:
   ```bash
   kubectl create secret generic docetl-secret \
     --from-literal=openai-api-key=sk-new-key \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
4. **Restart** pods to pick up new secret:
   ```bash
   kubectl rollout restart deployment/docetl-backend
   ```
5. **Audit** access logs to determine exposure scope
6. **Review** security practices to prevent recurrence

## Additional Resources

- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/security-best-practices/)
- [OWASP Kubernetes Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [Sealed Secrets Documentation](https://github.com/bitnami-labs/sealed-secrets)
- [HashiCorp Vault on Kubernetes](https://developer.hashicorp.com/vault/docs/platform/k8s)
- [External Secrets Operator](https://external-secrets.io/)

## Support

For security-related questions or to report vulnerabilities:

- Security issues: Report privately via GitHub Security Advisories
- General questions: [GitHub Discussions](https://github.com/X-tructure/helm-charts/discussions)
