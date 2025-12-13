# Technitium DNS Server - Certificate Setup Guide

This guide covers how to set up locally-trusted certificates for Technitium DNS Server clustering and HTTPS in Kubernetes.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1: Create a Local Certificate Authority (CA)](#step-1-create-a-local-certificate-authority-ca)
- [Step 2: Generate Server Certificates](#step-2-generate-server-certificates)
- [Step 3: Create PFX Certificate](#step-3-create-pfx-certificate)
- [Step 4: Deploy Certificates to Kubernetes](#step-4-deploy-certificates-to-kubernetes)
- [Step 5: Configure Helm Chart](#step-5-configure-helm-chart)
- [Step 6: Configure Technitium](#step-6-configure-technitium)
- [Troubleshooting](#troubleshooting)
- [Reference](#reference)

## Overview

For Technitium DNS clustering to work properly with HTTPS, you need:

1. **Custom CA Certificate**: A locally-trusted root CA that all cluster nodes trust
2. **PFX Certificate**: A server certificate in PFX/P12 format that Technitium uses for HTTPS

This setup allows you to use self-signed certificates that are trusted across your network without certificate warnings.

## Prerequisites

- `openssl` installed on your local machine
- `kubectl` configured to access your Kubernetes cluster
- `helm` 3.x installed
- Access to the namespace where Technitium will be deployed

## Step 1: Create a Local Certificate Authority (CA)

First, create your own Certificate Authority that will sign your server certificates.

```bash
# Generate CA private key
openssl genrsa -out myLocalCA.key 4096

# Create CA certificate (valid for 10 years)
openssl req -x509 -new -nodes -key myLocalCA.key \
  -sha256 -days 3650 -out myLocalCA.crt \
  -subj "/C=US/ST=State/L=City/O=My Home Network/CN=My Local CA"
```

**Important**: Store `myLocalCA.key` securely - this is your root CA's private key. Anyone with this key can create certificates that your network will trust.

### Install CA on Client Devices

To avoid certificate warnings when accessing Technitium's web interface:

**Windows:**
1. Double-click `myLocalCA.crt`
2. Click "Install Certificate"
3. Select "Local Machine" → Next
4. Choose "Place all certificates in the following store"
5. Browse and select "Trusted Root Certification Authorities"
6. Complete the wizard

**Linux:**
```bash
sudo cp myLocalCA.crt /usr/local/share/ca-certificates/myLocalCA.crt
sudo update-ca-certificates
```

**macOS:**
```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain myLocalCA.crt
```

## Step 2: Generate Server Certificates

Create certificates for your Technitium DNS servers.

### Create Subject Alternative Names (SAN) Configuration

Create a file `technitium.ext` with your server's DNS names and IP addresses:

```bash
cat > technitium.ext << EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = technitium.local
DNS.2 = technitium1.local
DNS.3 = technitium2.local
DNS.4 = *.technitium.local
IP.1 = 192.168.1.10
IP.2 = 192.168.1.11
EOF
```

**Customize the alt_names section:**
- Add all DNS names your Technitium servers will use
- Add all IP addresses (including LoadBalancer IPs)
- Use wildcards if needed (e.g., `*.technitium.local`)

### Generate Server Certificate

```bash
# Generate server private key
openssl genrsa -out technitium.key 2048

# Create certificate signing request (CSR)
openssl req -new -key technitium.key -out technitium.csr \
  -subj "/CN=technitium.local"

# Sign the certificate with your CA
openssl x509 -req -in technitium.csr \
  -CA myLocalCA.crt -CAkey myLocalCA.key \
  -CAcreateserial -out technitium.crt \
  -days 825 -sha256 -extfile technitium.ext
```

**Note**: Modern browsers require certificates with a validity period of 825 days or less.

### Verify the Certificate

```bash
# View certificate details
openssl x509 -in technitium.crt -text -noout

# Verify the certificate chain
openssl verify -CAfile myLocalCA.crt technitium.crt
```

You should see: `technitium.crt: OK`

## Step 3: Create PFX Certificate

Technitium requires certificates in PFX (PKCS#12) format.

```bash
# Convert to PFX format
openssl pkcs12 -export \
  -out technitium.pfx \
  -inkey technitium.key \
  -in technitium.crt \
  -certfile myLocalCA.crt \
  -name "Technitium DNS Server"
```

You'll be prompted to enter an export password. **Remember this password** - you'll need it for Kubernetes and Technitium configuration.

**Creating a PFX without password** (not recommended for production):
```bash
openssl pkcs12 -export \
  -out technitium.pfx \
  -inkey technitium.key \
  -in technitium.crt \
  -certfile myLocalCA.crt \
  -passout pass:
```

## Step 4: Deploy Certificates to Kubernetes

### Create Kubernetes Secrets

```bash
# Create namespace if it doesn't exist
kubectl create namespace technitium

# Create CA certificate secret
kubectl create secret generic local-ca-cert \
  --from-file=ca.crt=myLocalCA.crt \
  -n technitium

# Create PFX certificate secret
kubectl create secret generic technitium-pfx-cert \
  --from-file=certificate.pfx=technitium.pfx \
  --from-literal=password='YourPfxPassword' \
  -n technitium
```

### Verify Secrets

```bash
# List secrets
kubectl get secrets -n technitium

# View secret details (base64 encoded)
kubectl describe secret local-ca-cert -n technitium
kubectl describe secret technitium-pfx-cert -n technitium
```

## Step 5: Configure Helm Chart

### Option A: Using values.yaml

Create a `custom-values.yaml` file:

```yaml
# Enable custom CA certificate trust
customCA:
  enabled: true
  secret:
    name: local-ca-cert
    key: ca.crt

# Enable PFX certificate for HTTPS
pfxCertificate:
  enabled: true
  secret:
    name: technitium-pfx-cert
    pfxKey: certificate.pfx
    passwordKey: password
  mountPath: /etc/dns/certs
  filename: certificate.pfx

# Configure Technitium DNS Server
dnsServer:
  domain: technitium.local
  adminPassword: "ChangeMe123!"
  webServiceHttpPort: 5380
  webServiceHttpsPort: 5381
  webServiceEnableHttps: true
  webServiceUseSelfSignedCert: false  # We're using our own cert

# Enable persistence for certificate configuration
persistence:
  enabled: true
  size: 1Gi
```

### Option B: Using --set Flags

```bash
helm upgrade --install technitium ./technitium-dns-server \
  --namespace technitium \
  --set customCA.enabled=true \
  --set customCA.secret.name=local-ca-cert \
  --set pfxCertificate.enabled=true \
  --set pfxCertificate.secret.name=technitium-pfx-cert \
  --set dnsServer.webServiceEnableHttps=true \
  --set dnsServer.webServiceUseSelfSignedCert=false
```

### Deploy the Chart

```bash
# Install or upgrade
helm upgrade --install technitium ./technitium-dns-server \
  -f custom-values.yaml \
  -n technitium \
  --create-namespace

# Watch the pods start
kubectl get pods -n technitium -w
```

## Step 6: Configure Technitium

### Access the Web Interface

```bash
# Get the service IP/hostname
kubectl get svc -n technitium

# Port forward for local access (if needed)
kubectl port-forward -n technitium svc/technitium-dns-server 5380:5380
```

Access the web interface at: `http://localhost:5380`

### Import the Certificate

1. Log in to Technitium web interface
2. Go to **Settings** → **Certificates**
3. Click **Import Certificate**
4. Select the certificate file: `/etc/dns/certs/certificate.pfx`
5. Enter the password from your secret
6. Click **Import**
7. Enable HTTPS and select the imported certificate
8. Save settings and restart if prompted

### Configure Clustering

With certificates in place, you can now configure clustering:

1. Deploy a second Technitium instance with the same certificates
2. Go to **Settings** → **Clustering** on the primary node
3. Add the secondary node using its HTTPS URL
4. The nodes should now trust each other's certificates

## Helm Chart Configuration Reference

### Custom CA Certificate Options

```yaml
customCA:
  enabled: false                    # Enable custom CA certificate trust
  
  # Option 1: Reference existing secret
  secret:
    name: ""                        # Name of secret containing CA cert
    key: "ca.crt"                   # Key in secret with certificate data
  
  # Option 2: Reference existing configmap
  configmap:
    name: ""                        # Name of configmap containing CA cert
    key: "ca.crt"                   # Key in configmap with certificate data
  
  # Option 3: Provide certificate inline (creates secret automatically)
  certificate: ""                   # PEM-encoded CA certificate content
```

### PFX Certificate Options

```yaml
pfxCertificate:
  enabled: false                    # Enable PFX certificate mounting
  mountPath: /etc/dns/certs         # Where to mount certificate in pod
  filename: certificate.pfx         # Filename for the certificate
  
  # Reference existing secret
  secret:
    name: ""                        # Name of secret containing PFX
    pfxKey: "certificate.pfx"       # Key in secret with PFX data
    passwordKey: "password"         # Key in secret with password
  
  # Or provide PFX inline (creates secret automatically)
  pfxData: ""                       # Base64-encoded PFX file content
  password: ""                      # PFX password (will be base64 encoded)
```

## Troubleshooting

### Certificate Not Trusted in Pod

Check if the CA certificate was installed correctly:

```bash
# Exec into the pod
kubectl exec -it -n technitium deployment/technitium-dns-server -- sh

# Check if CA is present
ls -la /etc/ssl/certs/ | grep custom-ca

# Test certificate verification
openssl verify -CAfile /etc/ssl/certs/custom-ca.pem /path/to/server/cert
```

### PFX Certificate Not Found

Verify the certificate is mounted:

```bash
# Check mounted files
kubectl exec -it -n technitium deployment/technitium-dns-server -- \
  ls -la /etc/dns/certs/

# View certificate info
kubectl exec -it -n technitium deployment/technitium-dns-server -- \
  openssl pkcs12 -info -in /etc/dns/certs/certificate.pfx -nodes -passin pass:YourPassword
```

### Init Container Fails

Check the init container logs:

```bash
# View init container logs
kubectl logs -n technitium deployment/technitium-dns-server -c install-ca-cert

# Describe pod for events
kubectl describe pod -n technitium -l app.kubernetes.io/name=technitium-dns-server
```

### Certificate Expired

Certificates have a limited validity period. To renew:

```bash
# Check expiration date
openssl x509 -in technitium.crt -noout -enddate

# If expired, regenerate from Step 2 onwards
# Then update the Kubernetes secret
kubectl create secret generic technitium-pfx-cert \
  --from-file=certificate.pfx=technitium-new.pfx \
  --from-literal=password='YourPassword' \
  -n technitium \
  --dry-run=client -o yaml | kubectl apply -f -

# Restart pods to pick up new certificate
kubectl rollout restart deployment/technitium-dns-server -n technitium
```

### Browser Still Shows Certificate Warning

Ensure:
1. The CA certificate is installed on your client device
2. The server certificate's SAN includes the hostname/IP you're accessing
3. Browser cache is cleared (or try incognito mode)
4. Technitium is actually using the imported certificate (check Settings → Certificates)

### Clustering Not Working

Common issues:
1. **DNS resolution**: Ensure nodes can resolve each other's hostnames
2. **Certificate mismatch**: Both nodes must trust the same CA
3. **SAN missing**: Server certificate must include all node hostnames/IPs
4. **Firewall**: Ensure clustering ports are open between nodes

## Advanced Configurations

### Using Cert-Manager

For automated certificate management with cert-manager:

```yaml
# Create a CA Issuer
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: technitium
spec:
  ca:
    secretName: local-ca-cert

# Create Certificate resource
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: technitium-cert
  namespace: technitium
spec:
  secretName: technitium-tls
  issuerRef:
    name: ca-issuer
    kind: Issuer
  dnsNames:
    - technitium.local
    - technitium1.local
    - technitium2.local
  ipAddresses:
    - 192.168.1.10
    - 192.168.1.11
```

### Using External Secrets Operator

For GitOps-friendly secret management:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: technitium-pfx-cert
  namespace: technitium
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: technitium-pfx-cert
  data:
    - secretKey: certificate.pfx
      remoteRef:
        key: technitium/pfx
    - secretKey: password
      remoteRef:
        key: technitium/pfx-password
```

### Multiple Certificates for Different Environments

You can deploy multiple Technitium instances with different certificates:

```bash
# Production
helm upgrade --install technitium-prod ./technitium-dns-server \
  -f values-prod.yaml \
  --set pfxCertificate.secret.name=technitium-prod-cert \
  -n technitium-prod

# Staging
helm upgrade --install technitium-staging ./technitium-dns-server \
  -f values-staging.yaml \
  --set pfxCertificate.secret.name=technitium-staging-cert \
  -n technitium-staging
```

## Security Best Practices

1. **Protect CA Private Key**: Store `myLocalCA.key` in a secure location (password manager, hardware security module)
2. **Use Strong Passwords**: For PFX files, use strong passwords (20+ characters)
3. **Rotate Certificates**: Set reminders to rotate certificates before expiration
4. **Limit CA Usage**: Only use your CA for internal certificates, not public-facing services
5. **Use Secrets Management**: Use tools like Sealed Secrets or External Secrets Operator for GitOps
6. **Enable RBAC**: Restrict access to secrets containing certificates
7. **Audit Access**: Monitor who accesses certificate secrets

## Files Generated

After following this guide, you'll have:

```
.
├── myLocalCA.key          # CA private key (keep secure!)
├── myLocalCA.crt          # CA certificate (distribute to clients)
├── myLocalCA.srl          # CA serial number file
├── technitium.key         # Server private key
├── technitium.csr         # Certificate signing request
├── technitium.crt         # Server certificate
├── technitium.ext         # SAN configuration
├── technitium.pfx         # PFX certificate for Technitium
└── custom-values.yaml     # Helm values file
```

**Important**: Only `myLocalCA.crt` and `technitium.pfx` need to be distributed. Keep all `.key` files secure and never commit them to version control.

## Additional Resources

- [Technitium DNS Server Documentation](https://technitium.com/dns/)
- [OpenSSL Documentation](https://www.openssl.org/docs/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Helm Documentation](https://helm.sh/docs/)
- [Certificate Management Best Practices](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

---

**Last Updated**: December 2025