# Technitium DNS Server Helm Chart

A Helm chart for deploying [Technitium DNS Server](https://technitium.com/dns/) on Kubernetes.

Technitium DNS Server is an open source authoritative as well as recursive DNS server that can be used for self hosting a DNS server for privacy & security. It works out-of-the-box with no or minimal configuration and provides a user friendly web console accessible using any modern web browser.

## Features

- **Authoritative & Recursive DNS Server**: Works as both authoritative and recursive DNS server
- **DNS-over-TLS & DNS-over-HTTPS**: Support for encrypted DNS protocols
- **Ad Blocking**: Built-in ad blocking using block lists
- **DNS Caching**: Intelligent caching with serve stale feature
- **DNSSEC**: Full support for DNSSEC validation
- **Web Console**: User-friendly web interface for management
- **High Availability**: Kubernetes-native with persistent storage support

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure (if persistence is enabled)

## Installing the Chart

To install the chart with the release name `my-dns`:

```bash
helm install my-dns .
```

The command deploys Technitium DNS Server on the Kubernetes cluster with default configuration. The [Parameters](#parameters) section lists the parameters that can be configured during installation.

## Uninstalling the Chart

To uninstall/delete the `my-dns` deployment:

```bash
helm uninstall my-dns
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Parameters

### Global Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `nameOverride` | String to partially override the fullname | `""` |
| `fullnameOverride` | String to fully override the fullname | `""` |

### Image Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Technitium DNS Server image repository | `technitium/dns-server` |
| `image.tag` | Image tag (defaults to Chart appVersion) | `""` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `imagePullSecrets` | Specify docker-registry secret names | `[]` |

### DNS Server Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `dnsServer.domain` | Primary domain name for DNS server identification | `""` |
| `dnsServer.adminPassword` | Admin password (leave empty for auto-generated) | `""` |
| `dnsServer.preferIPv6` | Prefer IPv6 when querying | `false` |
| `dnsServer.webServiceHttpPort` | Web console HTTP port | `5380` |
| `dnsServer.webServiceHttpsPort` | Web console HTTPS port | `53443` |
| `dnsServer.webServiceEnableHttps` | Enable HTTPS for web console | `false` |
| `dnsServer.webServiceUseSelfSignedCert` | Use self-signed TLS certificate | `false` |
| `dnsServer.forwarders` | Comma-separated list of DNS forwarders | `""` |
| `dnsServer.forwarderProtocol` | Protocol for forwarders (Udp, Tcp, Tls, Https, HttpsJson) | `"Tcp"` |
| `dnsServer.enableBlocking` | Enable domain blocking | `false` |
| `dnsServer.blockListUrls` | Comma-separated list of block list URLs | `""` |

### Service Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.web.type` | Web console service type | `ClusterIP` |
| `service.web.port` | Web console service port | `5380` |
| `service.web.annotations` | Annotations for web service | `{}` |
| `service.dns.type` | DNS service type | `LoadBalancer` |
| `service.dns.annotations` | Annotations for DNS service | `{}` |
| `service.dns.loadBalancerIP` | Specific IP for LoadBalancer | `""` |
| `service.dns.loadBalancerSourceRanges` | Source ranges for LoadBalancer | `[]` |
| `service.dnsTls.enabled` | Enable DNS-over-TLS service | `false` |
| `service.dnsTls.type` | DNS-over-TLS service type | `ClusterIP` |
| `service.dnsTls.port` | DNS-over-TLS port | `853` |
| `service.dnsHttps.enabled` | Enable DNS-over-HTTPS service | `false` |
| `service.dnsHttps.type` | DNS-over-HTTPS service type | `ClusterIP` |
| `service.dnsHttps.port` | DNS-over-HTTPS port | `443` |

### Ingress Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `ingress.enabled` | Enable ingress controller resource | `false` |
| `ingress.className` | Ingress class name | `""` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | Hostname(s) for the ingress | `[{"host": "dns.example.local", "paths": [{"path": "/", "pathType": "Prefix"}]}]` |
| `ingress.tls` | TLS configuration | `[]` |

### Persistence Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `persistence.enabled` | Enable persistence using PVC | `true` |
| `persistence.storageClass` | PVC Storage Class | `""` |
| `persistence.accessMode` | PVC Access Mode | `ReadWriteOnce` |
| `persistence.size` | PVC Storage Request | `1Gi` |
| `persistence.existingClaim` | Use existing PVC | `""` |
| `persistence.annotations` | PVC annotations | `{}` |

### Other Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `serviceAccount.create` | Create service account | `true` |
| `serviceAccount.annotations` | Service account annotations | `{}` |
| `serviceAccount.name` | Service account name | `""` |
| `podAnnotations` | Pod annotations | `{}` |
| `podSecurityContext` | Pod security context | `{}` |
| `securityContext` | Container security context | `{}` |
| `resources` | CPU/Memory resource requests/limits | `{}` |
| `nodeSelector` | Node labels for pod assignment | `{}` |
| `tolerations` | Tolerations for pod assignment | `[]` |
| `affinity` | Affinity for pod assignment | `{}` |

## Configuration Examples

### Basic Installation with Custom Admin Password

```yaml
dnsServer:
  adminPassword: "MySecurePassword123"
  domain: "dns.example.com"
```

### Enable Ad Blocking

```yaml
dnsServer:
  enableBlocking: true
  blockListUrls: "https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts"
```

### Configure Custom Forwarders

```yaml
dnsServer:
  forwarders: "1.1.1.1, 8.8.8.8"
  forwarderProtocol: "Tls"
```

### Enable Ingress for Web Console

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: dns.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: dns-tls
      hosts:
        - dns.example.com
```

### Production Configuration with Resources

```yaml
replicaCount: 1

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

persistence:
  enabled: true
  size: 10Gi
  storageClass: "fast-ssd"

service:
  dns:
    type: LoadBalancer
    loadBalancerIP: "203.0.113.10"
```

## Upgrading

To upgrade the chart:

```bash
helm upgrade my-dns .
```

## Access the Web Console

After installation, follow the instructions in the NOTES to access the web console and get the admin credentials.

For a quick port-forward access:

```bash
kubectl port-forward svc/my-dns-technitium-dns-server-web 8080:5380
```

Then open http://localhost:8080 in your browser.

## Troubleshooting

### Pod is not starting

Check the pod logs:
```bash
kubectl logs -l app.kubernetes.io/name=technitium-dns-server
```

### Cannot access DNS service

Verify the service has an external IP (for LoadBalancer type):
```bash
kubectl get svc
```

### Persistence issues

Check PVC status:
```bash
kubectl get pvc
```

## Links

- [Technitium DNS Server Documentation](https://github.com/TechnitiumSoftware/DnsServer)
- [API Documentation](https://github.com/TechnitiumSoftware/DnsServer/blob/master/APIDOCS.md)
- [Docker Hub](https://hub.docker.com/r/technitium/dns-server)
- [Official Website](https://technitium.com/dns/)

## License

This Helm chart is provided as-is. Technitium DNS Server is licensed under GPL-3.0.
