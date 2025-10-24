# Aurison Helm Charts

## Local Testing (Development Only)
```bash
# Test template rendering
helm template aurison-backend ./aurison-backend

# Test with mTLS enabled (for development)
helm template aurison-backend ./aurison-backend \
  --set aurisonBackendPacs.enabled=true \
  --set mtls.enabled=true \
  --set-file mtls.caCertificate=C:/Users/inventor/Downloads/tmp/ca-certificate.pem \
  --set-file mtls.caPrivateKey=C:/Users/inventor/Downloads/tmp/ca-private-key.pem
```

## Production Deployment
Production deployment uses existing secret: `aurison-backend-prod-ca-secret-cert`
No `--set-file` needed - certificates are already in Kubernetes

## To Update Chart:
1. Modify Chart.yaml version
2. Run: `helm package ./aurison-backend`
3. Run: `helm repo index .`
4. Commit and push to trigger ArgoCD sync  