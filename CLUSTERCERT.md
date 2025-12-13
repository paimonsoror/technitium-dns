# Technitium DNS Server Helm Chart

## Quick Start with Certificates

For Technitium DNS clustering with HTTPS, you need to set up locally-trusted certificates. See [CERTIFICATE_SETUP.md](./CERTIFICATE_SETUP.md) for detailed instructions.

### TL;DR - Certificate Setup

```bash
# 1. Create CA and server certificates
openssl genrsa -out myLocalCA.key 4096
openssl req -x509 -new -nodes -key myLocalCA.key -sha256 -days 3650 -out myLocalCA.crt
openssl genrsa -out technitium.key 2048
openssl req -new -key technitium.key -out technitium.csr
openssl x509 -req -in technitium.csr -CA myLocalCA.crt -CAkey myLocalCA.key \
  -CAcreateserial -out technitium.crt -days 825 -sha256

# 2. Create PFX certificate
openssl pkcs12 -export -out technitium.pfx \
  -inkey technitium.key -in technitium.crt -certfile myLocalCA.crt

# 3. Create Kubernetes secrets
kubectl create secret generic local-ca-cert \
  --from-file=ca.crt=myLocalCA.crt -n technitium
kubectl create secret generic technitium-pfx-cert \
  --from-file=certificate.pfx=technitium.pfx \
  --from-literal=password='YourPassword' -n technitium

# 4. Install with Helm
helm install technitium . \
  --set customCA.enabled=true \
  --set customCA.secret.name=local-ca-cert \
  --set pfxCertificate.enabled=true \
  --set pfxCertificate.secret.name=technitium-pfx-cert \
  --set dnsServer.webServiceEnableHttps=true \
  -n technitium --create-namespace
```

## Configuration

### Certificate Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `customCA.enabled` | Enable custom CA certificate trust | `false` |
| `customCA.secret.name` | Name of secret containing CA cert | `""` |
| `customCA.secret.key` | Key in secret with CA cert | `"ca.crt"` |
| `pfxCertificate.enabled` | Enable PFX certificate for HTTPS | `false` |
| `pfxCertificate.secret.name` | Name of secret containing PFX | `""` |
| `pfxCertificate.secret.pfxKey` | Key in secret with PFX data | `"certificate.pfx"` |
| `pfxCertificate.secret.passwordKey` | Key in secret with PFX password | `"password"` |
| `pfxCertificate.mountPath` | Mount path for certificate | `"/etc/dns/certs"` |

### DNS Server Options

| Parameter | Description | Default |
|-----------|-------------|---------|
| `dnsServer.domain` | DNS server domain name | `""` |
| `dnsServer.adminPassword` | Admin password | `""` |
| `dnsServer.webServiceHttpPort` | HTTP port | `5380` |
| `dnsServer.webServiceHttpsPort` | HTTPS port | `5381` |
| `dnsServer.webServiceEnableHttps` | Enable HTTPS | `false` |
| `dnsServer.webServiceUseSelfSignedCert` | Use self-signed cert | `true` |

See [values.yaml](./values.yaml) for all available options.

## Examples

### Basic Installation (No Certificates)

```yaml
# values.yaml
dnsServer:
  domain: dns.example.local
  adminPassword: "SecurePassword123!"

persistence:
  enabled: true
  size: 1Gi
```

### Production Setup with Custom Certificates

```yaml
# values.yaml
customCA:
  enabled: true
  secret:
    name: local-ca-cert

pfxCertificate:
  enabled: true
  secret:
    name: technitium-pfx-cert
    pfxKey: certificate.pfx
    passwordKey: password

dnsServer:
  domain: dns.company.local
  adminPassword: "VerySecurePassword123!"
  webServiceEnableHttps: true
  webServiceUseSelfSignedCert: false
  forwarders: "1.1.1.1, 8.8.8.8"
  
persistence:
  enabled: true
  size: 5Gi
  storageClass: fast-ssd

resources:
  limits:
    cpu: 1000m
    memory: 512Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

### Clustered Setup (Multiple Instances)

Deploy primary node:
```bash
helm install technitium-primary . \
  -f values-primary.yaml \
  -n technitium
```

Deploy secondary node:
```bash
helm install technitium-secondary . \
  -f values-secondary.yaml \
  -n technitium
```

Both nodes should use the same CA and PFX certificates for proper clustering.

## Upgrading

```bash
# Upgrade to latest version
helm upgrade technitium . -n technitium

# Upgrade with new values
helm upgrade technitium . -f new-values.yaml -n technitium

# Upgrade and rotate certificate
kubectl create secret generic technitium-pfx-cert-new \
  --from-file=certificate.pfx=technitium-new.pfx \
  --from-literal=password='NewPassword' -n technitium
helm upgrade technitium . \
  --set pfxCertificate.secret.name=technitium-pfx-cert-new \
  -n technitium
```

## Uninstalling

```bash
# Remove the release
helm uninstall technitium -n technitium

# Remove PVC if persistence was enabled
kubectl delete pvc -n technitium -l app.kubernetes.io/name=technitium-dns-server

# Remove secrets (if desired)
kubectl delete secret local-ca-cert technitium-pfx-cert -n technitium
```

## Troubleshooting

See [CERTIFICATE_SETUP.md](./CERTIFICATE_SETUP.md#troubleshooting) for detailed troubleshooting steps.

Quick checks:

```bash
# Check pod status
kubectl get pods -n technitium

# View logs
kubectl logs -n technitium deployment/technitium-dns-server

# Check if CA cert is installed
kubectl exec -n technitium deployment/technitium-dns-server -- \
  ls -la /etc/ssl/certs/ | grep custom-ca

# Verify PFX is mounted
kubectl exec -n technitium deployment/technitium-dns-server -- \
  ls -la /etc/dns/certs/
```

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## License

[Your License Here]

## Credits

- Technitium DNS Server: https://github.com/TechnitiumSoftware/DnsServer