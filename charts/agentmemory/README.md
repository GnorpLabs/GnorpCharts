# Agentmemory Helm Chart

Persistent memory server for AI coding agents (REST API + MCP server + real-time viewer), built on the `iii` engine.

## Introduction

This chart deploys [agentmemory](https://github.com/rohitg00/agentmemory) on a Kubernetes cluster using the Helm package manager, using GnorpLabs' custom image (`ghcr.io/gnorplabs/agentmemory`, built from `GnorpLabs/docker/agentmemory`).

Agentmemory is primarily consumed as a backend by `mcp-gateway` (in-cluster, over the REST/MCP API on port 3111). It also ships a real-time viewer UI (port 3113) intended for LAN access by humans.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PersistentVolume provisioner support in the underlying infrastructure (`freenas-iscsi-csi` by default)

## Installing the Chart

To install the chart with the release name `agentmemory`:

```bash
helm install agentmemory ./charts/agentmemory -n mcp-gateway --create-namespace
```

## Uninstalling the Chart

```bash
helm delete agentmemory -n mcp-gateway
```

## Parameters

### Common parameters

| Name | Description | Value |
|------|-------------|-------|
| `replicaCount` | Number of agentmemory replicas (fixed at 1 -- file-based state store, not designed for concurrent multi-pod access) | `1` |
| `namespace` | Informational only; actual namespace set at the helm/helmfile release level | `"mcp-gateway"` |

### Image parameters

| Name | Description | Value |
|------|-------------|-------|
| `image.repository` | agentmemory image repository | `ghcr.io/gnorplabs/agentmemory` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `image.tag` | Image tag (only `latest` is published today) | `"latest"` |

### Service parameters

| Name | Description | Value |
|------|-------------|-------|
| `service.type` | Service type | `ClusterIP` |
| `service.ports.http` | REST API + MCP HTTP surface | `3111` |
| `service.ports.stream` | Internal iii-engine stream worker | `3112` |
| `service.ports.viewer` | Real-time viewer UI | `3113` |
| `service.ports.engine` | iii-engine websocket (worker registration) | `49134` |
| `service.ports.metrics` | Prometheus-style metrics | `9464` |

### Ingress parameters

| Name | Description | Value |
|------|-------------|-------|
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.className` | Ingress class name | `"traefik"` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.host` | Hostname (viewer only -- see [Networking](#networking)) | `agentmemory.gnorplabs.cc` |
| `ingress.path` | Path | `/` |
| `ingress.pathType` | Path type | `ImplementationSpecific` |
| `ingress.tls` | TLS configuration | `[]` |

### Persistence parameters

| Name | Description | Value |
|------|-------------|-------|
| `persistence.claimName` | Existing PVC name (optional) | `""` |
| `persistence.storageClass` | Storage class | `"freenas-iscsi-csi"` |
| `persistence.size` | PVC size | `5Gi` |
| `persistence.accessMode` | Access mode | `ReadWriteOnce` |

### Config parameters

| Name | Description | Value |
|------|-------------|-------|
| `config.corsExtraOrigins` | Extra CORS allowed_origins appended to the rendered iii-config.yaml | `[]` |

### Other parameters

| Name | Description | Value |
|------|-------------|-------|
| `env.TZ` | Timezone | `"UTC"` |
| `resources` | Resource limits/requests | `{}` |
| `startupProbe` / `livenessProbe` / `readinessProbe` | Health probes | see `values.yaml` |
| `nodeSelector` | Node selector | `{}` |
| `tolerations` | Tolerations | `[]` |
| `affinity` | Affinity rules | `{}` |

## Configuration and Installation Details

### Why this chart ships its own iii-config.yaml

The image bakes in `docker/agentmemory/iii-config.yaml` (from GnorpLabs), which sets `host: [IP_ADDRESS]` on the `iii-http` (3111) and `iii-stream` (3112) workers. Unquoted in YAML, `[IP_ADDRESS]` parses as a one-item array, not a valid bind address. This chart renders a ConfigMap mirroring agentmemory's own known-working `iii-config.docker.yaml` (binding `0.0.0.0`) and mounts it over `/app/config.yaml`, instead of trusting the image's baked-in config. See `docs/superpowers/specs/2026-07-19-agentmemory-helmchart-design.md` in this repo for the full rationale.

### Networking

- **API (port 3111)** is ClusterIP-only and has no ingress. It's meant to be consumed in-cluster (e.g. by `mcp-gateway`) via the Service DNS name: `<release-name>.mcp-gateway.svc.cluster.local:3111`.
- **Viewer (port 3113)** is the only port exposed via ingress. Its frontend proxies REST calls to the API **server-side** (the browser never calls 3111 directly), so no separate API hostname is needed for the viewer to function.
- **Live updates** in the viewer connect directly from the browser to port 3112 via WebSocket, using client-side port arithmetic that assumes a raw `:port` URL. This does not survive being accessed through the hostname-based ingress and gracefully degrades to the viewer's existing 10s HTTP-polling fallback. This is a known upstream limitation and is not solved by this chart.

### Storage

Agentmemory stores its state store (`/data/state_store.db`) and stream store (`/data/stream_store`) on a single PVC mounted at `/data`. `podSecurityContext.fsGroup: 1000` matches the image's `USER node` (uid 1000) so the volume is writable without a chown init container.

### Security

No authentication (`AGENTMEMORY_SECRET`) is configured by default -- the API and viewer are unauthenticated. This is a deliberate v1 tradeoff since agentmemory is only intended to be reachable within the trusted cluster/LAN. See the design doc for details.

## Upgrading

```bash
helm upgrade agentmemory ./charts/agentmemory -n mcp-gateway
```

## Deferred / Out of Scope for v1

- Authentication (`AGENTMEMORY_SECRET`)
- LLM/embedding provider keys (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.) -- TODO placeholder in `values.yaml`, agentmemory runs BM25-only without them
- `ServiceMonitor` for the metrics port (the port is exposed on the Service, but no CRD is created)
- TLS on the viewer ingress

## Support

For questions or issues with agentmemory itself: [rohitg00/agentmemory](https://github.com/rohitg00/agentmemory).
