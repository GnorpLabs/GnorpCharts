# Clonarr Helm Chart

Visual TRaSH Guides sync tool for Radarr and Sonarr.

## Introduction

This chart deploys [Clonarr](https://github.com/ProphetSe7en/clonarr) on a Kubernetes cluster using the Helm package manager.

Clonarr provides a web-based UI for managing quality profiles, custom formats, and sync operations for Radarr and Sonarr based on [TRaSH Guides](https://trash-guides.info/).

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PersistentVolume provisioner support in the underlying infrastructure

## Installing the Chart

To install the chart with the release name `clonarr`:

```bash
helm install clonarr ./charts/clonarr -n plex --create-namespace
```

The command deploys Clonarr on the Kubernetes cluster with default configuration. The [Parameters](#parameters) section lists the parameters that can be configured during installation.

## Uninstalling the Chart

To uninstall/delete the `clonarr` deployment:

```bash
helm delete clonarr -n plex
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Parameters

### Common parameters

| Name | Description | Value |
|------|-------------|-------|
| `replicaCount` | Number of Clonarr replicas | `1` |
| `namespace` | Namespace to deploy Clonarr | `"plex"` |

### Image parameters

| Name | Description | Value |
|------|-------------|-------|
| `image.repository` | Clonarr image repository | `ghcr.io/prophetse7en/clonarr` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `image.tag` | Image tag (defaults to chart appVersion) | `"latest"` |

### Environment variables

| Name | Description | Value |
|------|-------------|-------|
| `env.TZ` | Timezone | `"UTC"` |
| `env.PUID` | User ID for file ownership | `"99"` |
| `env.PGID` | Group ID for file ownership | `"100"` |
| `env.PORT` | Web UI port (optional) | `""` |
| `env.CONFIG_DIR` | Config directory path (optional) | `""` |
| `env.DATA_DIR` | TRaSH Guides data directory (optional) | `""` |
| `env.URL_BASE` | URL path prefix for reverse proxy (optional) | `""` |
| `env.TRUSTED_NETWORKS` | Comma-separated IPs/CIDRs for auth bypass (optional) | `""` |
| `env.TRUSTED_PROXIES` | Comma-separated proxy IPs (optional) | `""` |

### Service parameters

| Name | Description | Value |
|------|-------------|-------|
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `6060` |

### Ingress parameters

| Name | Description | Value |
|------|-------------|-------|
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.className` | Ingress class name | `"traefik"` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts[0].host` | Hostname | `clonarr.gnorplabs.cc` |
| `ingress.hosts[0].paths[0].path` | Path | `/` |
| `ingress.hosts[0].paths[0].pathType` | Path type | `ImplementationSpecific` |
| `ingress.tls` | TLS configuration | `[]` |

### Persistence parameters

| Name | Description | Value |
|------|-------------|-------|
| `persistence.config.claimName` | Existing PVC name (optional) | `""` |
| `persistence.config.storageClass` | Storage class | `""` |
| `persistence.config.size` | PVC size | `10Gi` |
| `persistence.config.accessMode` | Access mode | `ReadWriteOnce` |

### Other parameters

| Name | Description | Value |
|------|-------------|-------|
| `resources` | Resource limits/requests | `{}` |
| `nodeSelector` | Node selector | `{}` |
| `tolerations` | Tolerations | `[]` |
| `affinity` | Affinity rules | `{}` |

## Configuration and Installation Details

### First-Run Setup

After deploying the chart:

1. Access the Clonarr web UI via your configured ingress (e.g., `http://clonarr.gnorplabs.cc`)
2. You will be redirected to `/setup` to create an admin account
3. Create a username and password (minimum 10 characters)
4. Configure your Radarr/Sonarr instances via Settings > Instances
5. Pull the TRaSH Guides repository
6. Begin syncing profiles

### Storage

Clonarr requires persistent storage for:
- Configuration files (`/config/clonarr.json`)
- User credentials (`/config/auth.json`)
- Profile sync rules (`/config/profiles/`)
- TRaSH Guides repository cache (`/config/data/trash-guides/`)

The default storage size is 10Gi. Adjust via `persistence.config.size` if needed.

### Networking

By default, the chart creates:
- A ClusterIP service on port 6060
- An Ingress resource (if `ingress.enabled=true`)

For external access without ingress, change the service type:

```bash
helm install clonarr ./charts/clonarr --set service.type=LoadBalancer
```

### Security

Clonarr includes built-in authentication. All credentials are stored encrypted in the persistent volume.

For enhanced security:
- Use TLS with ingress (configure `ingress.tls`)
- Restrict trusted networks via `env.TRUSTED_NETWORKS`
- Configure trusted proxies via `env.TRUSTED_PROXIES` if using a reverse proxy

## Upgrading

To upgrade the chart:

```bash
helm upgrade clonarr ./charts/clonarr -n plex
```

## Support

For questions or issues:
- GitHub: [ProphetSe7en/clonarr](https://github.com/ProphetSe7en/clonarr/issues)
- Discord: [TRaSH Guides Discord](https://trash-guides.info/discord) - `#clonarr` channel
