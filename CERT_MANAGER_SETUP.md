# cert-manager Setup Guide for Cloudflare Tunnel

## Overview

This document describes the setup of cert-manager v1.18.2 with DNS-01 challenge for automatic TLS certificate management in a Kubernetes cluster using Cloudflare Tunnel.

## Why cert-manager?

**Problem**: Cloudflare limits file uploads to 100MB for proxied domains.

**Solution**: Bypass Cloudflare proxy for services like MinIO that need large file uploads, but still need valid HTTPS certificates.

**Challenge**: With Cloudflare Tunnel, Let's Encrypt's HTTP-01 challenge doesn't work (no direct HTTP access to the cluster).

**Final Solution**: Use cert-manager with DNS-01 challenge via Cloudflare API.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloudflare DNS                            │
│  - ArgoCD, K8s Dashboard, Mango, UIO MCP (Orange Cloud)     │
│  - MinIO (Grey Cloud - DNS Only)                            │
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴──────────┐
                    │                    │
          ┌─────────▼──────┐   ┌────────▼────────┐
          │ Cloudflare     │   │  Direct Access  │
          │ Tunnel         │   │  (MinIO only)   │
          │ (Port 80/443)  │   │                 │
          └─────────┬──────┘   └────────┬────────┘
                    │                    │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  NGINX Ingress     │
                    │  Controller        │
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │  Services          │
                    │  - ArgoCD          │
                    │  - K8s Dashboard   │
                    │  - Mango           │
                    │  - UIO MCP         │
                    │  - MinIO ✓ (cert)  │
                    └────────────────────┘
```

---

## Installation Steps

### 1. Download cert-manager Manifests

Download the latest cert-manager (v1.18.2):

```bash
curl -sL https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml \
  -o infra/cert-manager.yml
```

### 2. Create ClusterIssuers with DNS-01 Challenge

Create `infra/cert-manager-issuers.yml`:

```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: neurocollab.in@gmail.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: neurocollab.in@gmail.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
```

**Key Differences:**
- **staging**: For testing (fake certs, no rate limits)
- **prod**: For production (real certs, rate limited to 50/week)

### 3. Update Services Ingress for TLS

Updated `services/services-ingress.yml` to add TLS for MinIO only:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: services-ingress
  namespace: services
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - minio.neurocollab.in
    secretName: minio-tls
  rules:
    - host: mango.neurocollab.in
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mango
                port:
                  number: 8080
    - host: uio-mcp.neurocollab.in
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: uio-mcp-gateway
                port:
                  number: 3002
    - host: minio.neurocollab.in
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: minio
                port:
                  number: 9001
```

### 4. Create Cloudflare API Token

**Steps:**
1. Go to: https://dash.cloudflare.com/profile/api-tokens
2. Click **"Create Token"**
3. Use template: **"Edit zone DNS"**
4. Configure:
   - **Permissions**: Zone → DNS → Edit
   - **Zone Resources**: Include → Specific zone → `neurocollab.in`
5. Click "Create Token" and **copy the token**

### 5. Create Kubernetes Secret with API Token

```bash
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=YOUR_CLOUDFLARE_TOKEN_HERE \
  -n cert-manager
```

**Important**: This secret must be created **before** applying the ClusterIssuers.

### 6. Commit and Push to Git

```bash
git add infra/cert-manager.yml infra/cert-manager-issuers.yml services/services-ingress.yml
git commit -m "Add cert-manager with DNS-01 challenge for Cloudflare Tunnel"
git push
```

### 7. Sync ArgoCD

Since the infrastructure uses ArgoCD for GitOps:

1. Go to ArgoCD UI
2. Navigate to the `infra` application
3. Click **"Refresh"** or **"Sync"**
4. Wait for sync to complete

Or wait for automatic sync (configured with `autoSync: true`).

---

## Verification Commands

### Check cert-manager Pods

```bash
kubectl get pods -n cert-manager
```

Expected output:
```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-69f748766f-xxxxx              1/1     Running   0          5m
cert-manager-cainjector-7cf6557c49-xxxxx   1/1     Running   0          5m
cert-manager-webhook-58f4cff74d-xxxxx      1/1     Running   0          5m
```

### Check ClusterIssuers

```bash
kubectl get clusterissuer
```

Expected output:
```
NAME                  READY   AGE
letsencrypt-prod      True    5m
letsencrypt-staging   True    5m
```

### Verify ClusterIssuer Configuration

```bash
kubectl get clusterissuer letsencrypt-prod -o yaml | grep -A 10 "solvers:"
```

Should show:
```yaml
solvers:
- dns01:
    cloudflare:
      apiTokenSecretRef:
        key: api-token
        name: cloudflare-api-token
```

### Check Certificate Status

```bash
kubectl get certificate -n services
```

Expected output when ready:
```
NAME        READY   SECRET      AGE
minio-tls   True    minio-tls   5m
```

### Describe Certificate

```bash
kubectl describe certificate minio-tls -n services
```

Look for:
```
Status:
  Conditions:
    Message: Certificate is up to date and has not expired
    Reason:  Ready
    Status:  True
```

### Check Certificate Details

```bash
kubectl get secret minio-tls -n services -o yaml
```

### View Certificate Expiry

```bash
kubectl get certificate minio-tls -n services -o jsonpath='{.status.notAfter}'
```

---

## Troubleshooting

### Issue: Certificate Stuck in Pending

**Check challenges:**
```bash
kubectl get challenge -n services
kubectl describe challenge -n services <challenge-name>
```

**Common Issues:**

1. **HTTP-01 challenge failing** (dial tcp unreachable):
   - **Problem**: Using Cloudflare Tunnel, HTTP-01 won't work
   - **Solution**: Switch to DNS-01 challenge (already done)

2. **DNS-01 challenge pending**:
   - **Problem**: DNS propagation takes time
   - **Solution**: Wait 30-60 seconds
   - **Check**: Look for "DomainVerified" event in challenge description

3. **Cloudflare API token invalid**:
   ```bash
   kubectl logs -n cert-manager deployment/cert-manager | grep -i cloudflare
   ```
   - **Solution**: Recreate the secret with correct token

### Delete and Recreate Certificate

If certificate is stuck:

```bash
kubectl delete certificate minio-tls -n services
kubectl delete challenge --all -n services
```

The certificate will be automatically recreated by the ingress controller.

### Check cert-manager Logs

```bash
kubectl logs -n cert-manager deployment/cert-manager -f
```

---

## Cloudflare Configuration

### DNS Records Setup

| Domain | Type | Content | Proxy Status | Purpose |
|--------|------|---------|--------------|---------|
| `minio.neurocollab.in` | A/CNAME | Cluster IP | **Grey Cloud (DNS Only)** | Direct access, bypasses 100MB limit |
| `mango.neurocollab.in` | A/CNAME | Cluster IP | Orange Cloud (Proxied) | Protected by Cloudflare |
| `uio-mcp.neurocollab.in` | A/CNAME | Cluster IP | Orange Cloud (Proxied) | Protected by Cloudflare |
| `argocd.neurocollab.in` | A/CNAME | Cluster IP | Orange Cloud (Proxied) | Protected by Cloudflare |
| `k8dashboard.neurocollab.in` | A/CNAME | Cluster IP | Orange Cloud (Proxied) | Protected by Cloudflare |

**Important**: MinIO **must** be grey cloud (DNS only) even though we're using Cloudflare Tunnel for initial connection setup.

### Cloudflare Tunnel Configuration

Your `cloudflared` config includes all services:

```yaml
ingress:
  - hostname: k8dashboard.neurocollab.in
    service: http://localhost:80
  - hostname: argocd.neurocollab.in
    service: http://localhost:80
  - hostname: mango.neurocollab.in
    service: http://localhost:80
  - hostname: uio-mcp.neurocollab.in
    service: http://localhost:80
  - hostname: minio.neurocollab.in
    service: http://localhost:80
    originRequest:
      disableChunkedEncoding: true
      disableBuffer: true
```

---

## Maintenance

### Certificate Renewal

Certificates auto-renew **30 days before expiration**.

**Check renewal time:**
```bash
kubectl get certificate minio-tls -n services -o jsonpath='{.status.renewalTime}'
```

**Force renewal:**
```bash
kubectl delete certificate minio-tls -n services
```

### Updating cert-manager

1. Download new version:
```bash
curl -sL https://github.com/cert-manager/cert-manager/releases/download/vX.Y.Z/cert-manager.yaml \
  -o infra/cert-manager.yml
```

2. Commit and push:
```bash
git add infra/cert-manager.yml
git commit -m "Update cert-manager to vX.Y.Z"
git push
```

3. Sync ArgoCD

### Rotate Cloudflare API Token

1. Create new token in Cloudflare
2. Update the secret:
```bash
kubectl delete secret cloudflare-api-token -n cert-manager
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=NEW_TOKEN_HERE \
  -n cert-manager
```

3. Restart cert-manager:
```bash
kubectl rollout restart deployment cert-manager -n cert-manager
```

---

## Files Created/Modified

### New Files

1. **`infra/cert-manager.yml`**
   - cert-manager v1.18.2 complete installation
   - Includes CRDs, deployments, RBAC

2. **`infra/cert-manager-issuers.yml`**
   - ClusterIssuers for Let's Encrypt (staging + prod)
   - Configured with DNS-01 challenge via Cloudflare

### Modified Files

1. **`services/services-ingress.yml`**
   - Added TLS configuration for MinIO
   - Added cert-manager annotation

---

## Important Notes

### Why DNS-01 Instead of HTTP-01?

**HTTP-01 Requirements:**
- Direct HTTP access on port 80
- Let's Encrypt needs to reach `http://domain/.well-known/acme-challenge/`

**Problem with Cloudflare Tunnel:**
- Tunnel doesn't expose origin server's IP
- No direct port 80 access
- HTTP-01 challenge fails: "network is unreachable"

**DNS-01 Solution:**
- Validates ownership via DNS TXT records
- Works perfectly with Cloudflare Tunnel
- cert-manager creates TXT record `_acme-challenge.domain.com`
- Let's Encrypt checks DNS
- Certificate issued

### Security Considerations

1. **API Token Permissions**: Token only has DNS Edit for specific zone
2. **Secret Storage**: Never commit secrets to git
3. **Namespace**: Secret is in `cert-manager` namespace (restricted access)
4. **Token Rotation**: Rotate tokens periodically

### Rate Limits

**Let's Encrypt Production:**
- 50 certificates per registered domain per week
- Use staging for testing!

**Let's Encrypt Staging:**
- Much higher rate limits
- Certificates are not trusted (for testing only)

---

## Quick Reference Commands

```bash
# Check all cert-manager resources
kubectl get clusterissuer
kubectl get certificate -A
kubectl get certificaterequest -A
kubectl get challenge -A

# View cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager -f

# Describe certificate
kubectl describe certificate minio-tls -n services

# View certificate secret
kubectl get secret minio-tls -n services -o yaml

# Delete and recreate certificate
kubectl delete certificate minio-tls -n services

# Check ClusterIssuer configuration
kubectl get clusterissuer letsencrypt-prod -o yaml

# Verify secret exists
kubectl get secret cloudflare-api-token -n cert-manager

# Restart cert-manager
kubectl rollout restart deployment cert-manager -n cert-manager
```

---

## Success Criteria

✅ cert-manager pods running  
✅ ClusterIssuers ready  
✅ Certificate status: `READY = True`  
✅ No pending challenges  
✅ Certificate expires in ~90 days  
✅ Auto-renewal configured  
✅ MinIO accessible via HTTPS with valid certificate  
✅ Large file uploads working (>100MB)  

---

## Support & References

- **cert-manager Docs**: https://cert-manager.io/docs/
- **DNS-01 Cloudflare**: https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/
- **Let's Encrypt**: https://letsencrypt.org/docs/
- **Cloudflare API Tokens**: https://dash.cloudflare.com/profile/api-tokens

---

**Document Version**: 1.0  
**Last Updated**: October 2, 2025  
**Author**: Setup completed with AI assistance  
**Environment**: MicroK8s on neurocollab.in



