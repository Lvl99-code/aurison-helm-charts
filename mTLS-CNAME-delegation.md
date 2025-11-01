# 🎯 Complete Guide: DNS-01 ACME with CNAME Delegation for Kubernetes cert-manager

**Objective:** Issue Let's Encrypt certificates for a Kubernetes ingress using DNS-01 challenge with CNAME delegation, without changing primary domain nameservers.

**Use Case:** mTLS endpoint (`pacs.api.aurison.app`) requiring DNS-01 validation while keeping Squarespace as the primary DNS provider for `aurison.app`.

---

## 📋 Prerequisites

- Domain managed by Squarespace: `aurison.app`
- EKS cluster in AWS (region: `ap-southeast-2`)
- cert-manager installed in cluster (version: `v1.11.0`)
- AWS CLI configured with appropriate permissions
- kubectl access to the cluster

---

## Part 1: AWS Route53 Setup

### Step 1: Create Route53 Hosted Zone for ACME Challenges

```bash
# Create a dedicated hosted zone for ACME validation
aws route53 create-hosted-zone \
  --name acme-validation.aurison.app \
  --caller-reference $(date +%s) \
  --hosted-zone-config Comment="Dedicated zone for Let's Encrypt DNS-01 challenges"
```

**Output:** Note the `HostedZoneId` (e.g., `Z01940152KCX1GQ8ZY0YM`)

### Step 2: Get Route53 Nameservers

```bash
# Retrieve the nameservers for your new hosted zone
aws route53 list-resource-record-sets \
  --hosted-zone-id Z01940152KCX1GQ8ZY0YM \
  --query "ResourceRecordSets[?Type=='NS' && Name=='acme-validation.aurison.app.']"
```

**Expected output:**
```json
[
    {
        "Name": "acme-validation.aurison.app.",
        "Type": "NS",
        "TTL": 172800,
        "ResourceRecords": [
            {"Value": "ns-406.awsdns-50.com."},
            {"Value": "ns-1131.awsdns-13.org."},
            {"Value": "ns-610.awsdns-12.net."},
            {"Value": "ns-1592.awsdns-07.co.uk."}
        ]
    }
]
```

**Save these 4 nameservers** - you'll need them for Squarespace.

---

## Part 2: Squarespace DNS Configuration

### Step 3: Delegate the ACME Validation Subdomain

Log into Squarespace DNS settings and add **4 NS records**:

| Type | Host/Name | Value | TTL |
|------|-----------|-------|-----|
| NS | `acme-validation.aurison.app` | `ns-406.awsdns-50.com` | Auto |
| NS | `acme-validation.aurison.app` | `ns-1131.awsdns-13.org` | Auto |
| NS | `acme-validation.aurison.app` | `ns-610.awsdns-12.net` | Auto |
| NS | `acme-validation.aurison.app` | `ns-1592.awsdns-07.co.uk` | Auto |

*Note: Use the actual nameservers from Step 2*

### Step 4: Add CNAME Record for Your Domain

Add a CNAME record to redirect ACME challenges:

| Type | Host/Name | Value | TTL |
|------|-----------|-------|-----|
| CNAME | `_acme-challenge.pacs.api` | `_acme-challenge.pacs.api.acme-validation.aurison.app` | Auto |

**This is the key:** When Let's Encrypt queries `_acme-challenge.pacs.api.aurison.app`, it will follow the CNAME to Route53.

### Step 5: Verify DNS Propagation

Wait 10-15 minutes, then test:

```bash
# Test NS delegation
dig @8.8.8.8 acme-validation.aurison.app NS +short
# Should return the 4 Route53 nameservers

# Test CNAME resolution
dig @8.8.8.8 _acme-challenge.pacs.api.aurison.app CNAME +short
# Should return: _acme-challenge.pacs.api.acme-validation.aurison.app.
```

---

## Part 3: AWS IAM Setup (IRSA for cert-manager)

### Step 6: Create IAM Policy for Route53

```bash
# Create policy document
cat > route53-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/Z01940152KCX1GQ8ZY0YM"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ListHostedZonesByName",
      "Resource": "*"
    }
  ]
}
EOF

# Create the IAM policy
aws iam create-policy \
  --policy-name cert-manager-route53-policy \
  --policy-document file://route53-policy.json
```

**Output:** Note the policy ARN (e.g., `arn:aws:iam::937747935506:policy/cert-manager-route53-policy`)

### Step 7: Create IAM Role for Service Account (IRSA)

```bash
# Get your EKS cluster's OIDC provider
aws eks describe-cluster \
  --name your-cluster-name \
  --query "cluster.identity.oidc.issuer" \
  --output text
# Example output: https://oidc.eks.ap-southeast-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

# Extract the OIDC ID (the part after /id/)
OIDC_ID="EXAMPLED539D4633E53DE1B71EXAMPLE"

# Create trust policy
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::937747935506:oidc-provider/oidc.eks.ap-southeast-2.amazonaws.com/id/${OIDC_ID}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-southeast-2.amazonaws.com/id/${OIDC_ID}:sub": "system:serviceaccount:cert-manager:cert-manager",
          "oidc.eks.ap-southeast-2.amazonaws.com/id/${OIDC_ID}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name cert-manager-route53 \
  --assume-role-policy-document file://trust-policy.json

# Attach the policy to the role
aws iam attach-role-policy \
  --role-name cert-manager-route53 \
  --policy-arn arn:aws:iam::937747935506:policy/cert-manager-route53-policy
```

### Step 8: Annotate cert-manager Service Account

```bash
kubectl annotate serviceaccount cert-manager -n cert-manager \
  eks.amazonaws.com/role-arn=arn:aws:iam::937747935506:role/cert-manager-route53 \
  --overwrite
```

---

## Part 4: Configure cert-manager

### Step 9: Update cert-manager to Use Public DNS Resolvers

cert-manager needs to use public DNS (not cluster DNS) to properly follow CNAME records:

```bash
kubectl edit deployment cert-manager -n cert-manager
```

Add these two lines to the `args:` section:

```yaml
spec:
  template:
    spec:
      containers:
      - name: cert-manager-controller
        args:
        - --v=2
        - --cluster-resource-namespace=$(POD_NAMESPACE)
        - --leader-election-namespace=kube-system
        - --acme-http01-solver-image=quay.io/jetstack/cert-manager-acmesolver:v1.11.0
        - --max-concurrent-challenges=60
        # ADD THESE TWO LINES:
        - --dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53
        - --dns01-recursive-nameservers-only
```

Save and exit (`:wq` in vim).

### Step 10: Restart cert-manager

```bash
kubectl rollout restart deployment cert-manager -n cert-manager
kubectl rollout status deployment cert-manager -n cert-manager
```

**Verify the configuration:**

```bash
kubectl get deployment cert-manager -n cert-manager -o yaml | grep -A 2 "dns01-recursive"
```

---

## Part 5: Create cert-manager Resources

### Step 11: Create ClusterIssuer with DNS-01

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: david.oluwafemi@aurison.app
    privateKeySecretRef:
      name: letsencrypt-dns01-prod-key
    solvers:
    - selector:
        dnsNames:
        - "pacs.api.aurison.app"
      dns01:
        cnameStrategy: Follow  # Critical: enables CNAME following
        route53:
          region: ap-southeast-2
          hostedZoneID: Z01940152KCX1GQ8ZY0YM
EOF
```

**Verify:**

```bash
kubectl get clusterissuer letsencrypt-dns01-prod
```

Expected output: `READY=True`

### Step 12: Configure Helm Chart (values.yaml)

Update your ArgoCD Application's Helm values:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  name: aurison-prod-backend
  namespace: argocd
spec:
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
  destination:
    namespace: aurison
    server: 'https://kubernetes.default.svc'
  project: default
  source:
    chart: aurison-backend
    targetRevision: 0.1.*
    repoURL: https://lvl99-code.github.io/aurison-helm-charts/
    helm:
      values: |
        # ... your existing config ...
        
        # PACS ingress with mTLS
        ingressPacs:
          enabled: true
          annotations:
            cert-manager.io/cluster-issuer: letsencrypt-dns01-prod  # Use DNS-01
            kubernetes.io/ingress.class: nginx
            nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
            nginx.ingress.kubernetes.io/auth-tls-secret: "aurison/aurison-backend-ca-secret-cert"
            nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
          hosts:
            - host: pacs.api.aurison.app
              paths:
              - path: /ws/pacs/connect
                backend:
                  serviceName: aurison-backend-pacs
                  servicePort: 80
          tls:
            - secretName: pacs-api-tls
              hosts:
                - pacs.api.aurison.app
        
        # cert-manager configuration
        certManager:
          clusterIssuer:
            enabled: false  # Managed manually via kubectl
          
          certificate:
            enabled: true  # Helm manages the Certificate resource
            name: pacs-api-tls
            secretName: pacs-api-tls
            issuerRef:
              name: letsencrypt-dns01-prod
              kind: ClusterIssuer
            dnsNames:
              - "pacs.api.aurison.app"
        
        mtls:
          enabled: true
```

---

## Part 6: Verification

### Step 13: Monitor Certificate Issuance

```bash
# Watch certificate status
kubectl get certificate -n aurison -w

# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager -f | grep -i "pacs.api"

# Check challenge status
kubectl get challenges -n aurison
```

**What to expect:**

1. Challenge created: `pacs-api-tls-xxxxx-xxxxxxxxxx-xxxxxxxxxx`
2. cert-manager creates TXT record in Route53
3. DNS propagation check (may take 2-3 minutes)
4. Challenge validated
5. Certificate issued: `READY=True`

### Step 14: Verify the TXT Record (During Challenge)

While the challenge is active:

```bash
# Check Route53 has the TXT record
aws route53 list-resource-record-sets \
  --hosted-zone-id Z01940152KCX1GQ8ZY0YM \
  --query "ResourceRecordSets[?contains(Name, '_acme-challenge')]"

# Verify public DNS can resolve it
dig @8.8.8.8 _acme-challenge.pacs.api.aurison.app TXT +short
```

### Step 15: Confirm Certificate is Ready

```bash
kubectl get certificate pacs-api-tls -n aurison

# Expected output:
# NAME           READY   SECRET         AGE
# pacs-api-tls   True    pacs-api-tls   1s
```

```bash
# Check certificate details
kubectl describe certificate pacs-api-tls -n aurison

# Verify the secret exists
kubectl get secret pacs-api-tls -n aurison
```

---

## Part 7: mTLS Setup (Optional but Recommended)

### Step 16: Create CA Certificate for Client Validation

```bash
# Generate CA private key
openssl genrsa -out ca-private-key.pem 4096

# Generate CA certificate (valid for 10 years)
openssl req -new -x509 -days 3650 -key ca-private-key.pem -out ca-certificate.pem \
  -subj "/C=AU/ST=NSW/L=Sydney/O=Aurison/OU=IT/CN=Aurison Client CA"

# Create Kubernetes secret
kubectl create secret generic aurison-backend-ca-secret-cert \
  --from-file=ca.crt=ca-certificate.pem \
  --from-file=ca-certificate.pem=ca-certificate.pem \
  --from-file=ca-private-key.pem=ca-private-key.pem \
  -n aurison
```

### Step 17: Generate Client Certificate (for testing)

```bash
# Generate client private key
openssl genrsa -out client-private-key.pem 2048

# Generate certificate signing request
openssl req -new -key client-private-key.pem -out client.csr \
  -subj "/C=AU/ST=NSW/L=Sydney/O=Aurison/OU=IT/CN=test-client"

# Sign with your CA
openssl x509 -req -days 365 -in client.csr \
  -CA ca-certificate.pem -CAkey ca-private-key.pem \
  -CAcreateserial -out client-certificate.pem

# Test mTLS endpoint
curl --cert client-certificate.pem --key client-private-key.pem \
  https://pacs.api.aurison.app/ws/pacs/connect
```

---

## 🎯 Architecture Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    Let's Encrypt ACME Server                     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 1. Request certificate for
                           │    pacs.api.aurison.app
                           │ 2. Issue DNS-01 challenge
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      cert-manager (K8s)                          │
│  - ClusterIssuer: letsencrypt-dns01-prod                        │
│  - Certificate: pacs-api-tls                                     │
│  - IRSA: Uses IAM role to write to Route53                      │
└──────────────────────────┬──────────────────────────────────────┘
                           │ 3. Create TXT record
                           │    _acme-challenge.pacs.api.acme-validation.aurison.app
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│            Route53: acme-validation.aurison.app                  │
│            (Hosted Zone: Z01940152KCX1GQ8ZY0YM)                  │
│                                                                   │
│  TXT: _acme-challenge.pacs.api.acme-validation.aurison.app      │
│       = "challenge-token-here"                                   │
└─────────────────────────────────────────────────────────────────┘
                           ▲
                           │ 4. NS delegation
┌─────────────────────────────────────────────────────────────────┐
│                    Squarespace DNS (aurison.app)                 │
│                                                                   │
│  CNAME: _acme-challenge.pacs.api                                │
│         → _acme-challenge.pacs.api.acme-validation.aurison.app  │
│                                                                   │
│  NS: acme-validation.aurison.app                                │
│      → ns-406.awsdns-50.com (Route53)                           │
│      → ns-1131.awsdns-13.org                                     │
│      → ns-610.awsdns-12.net                                      │
│      → ns-1592.awsdns-07.co.uk                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Flow:**
1. Let's Encrypt asks: "Prove you own `pacs.api.aurison.app`"
2. cert-manager creates TXT record in Route53
3. Let's Encrypt queries `_acme-challenge.pacs.api.aurison.app`
4. Squarespace CNAME redirects to Route53's `acme-validation.aurison.app` zone
5. Route53 returns the TXT record
6. Let's Encrypt validates and issues certificate

---

## 🔧 Troubleshooting

### Problem: "DNS record not yet propagated"

```bash
# Check if CNAME exists
dig _acme-challenge.pacs.api.aurison.app CNAME +short

# Check if NS delegation works
dig acme-validation.aurison.app NS +short

# Check end-to-end resolution
dig @8.8.8.8 _acme-challenge.pacs.api.aurison.app TXT +short
```

**Solution:** Wait 10-15 minutes for DNS propagation.

### Problem: cert-manager can't write to Route53

```bash
# Check IRSA annotation
kubectl get sa cert-manager -n cert-manager -o yaml | grep eks.amazonaws.com

# Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager | grep -i route53
```

**Solution:** Verify IAM role, policy, and service account annotation.

### Problem: Certificate stuck in "Pending"

```bash
# Check challenge status
kubectl describe challenge -n aurison

# Check orders
kubectl get orders -n aurison
kubectl describe order -n aurison <order-name>
```

**Solution:** Delete and recreate certificate:
```bash
kubectl delete certificate pacs-api-tls -n aurison
kubectl delete challenges --all -n aurison
```

---

## 📝 Key Takeaways

1. **CNAME Delegation** allows DNS-01 validation without changing nameservers
2. **NS Records** must be added to delegate the `acme-validation` subdomain
3. **CNAME Record** redirects ACME challenges to the delegated zone
4. **cert-manager** must use public DNS resolvers (`--dns01-recursive-nameservers`)
5. **IRSA** (IAM Roles for Service Accounts) provides secure AWS API access
6. **cnameStrategy: Follow** is critical in the ClusterIssuer configuration

---

## 🎉 Success Criteria

- ✅ `kubectl get certificate pacs-api-tls -n aurison` shows `READY=True`
- ✅ TLS certificate is issued by Let's Encrypt
- ✅ Certificate auto-renews before expiration
- ✅ mTLS ingress validates client certificates
- ✅ No changes to primary domain nameservers

---

**Created:** October 28, 2025  
**Cluster:** EKS ap-southeast-2  
**cert-manager:** v1.11.0  
**Domain:** aurison.app (Squarespace)  
**mTLS Endpoint:** pacs.api.aurison.app