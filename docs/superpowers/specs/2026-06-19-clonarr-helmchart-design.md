# Clonarr Helm Chart Design

**Date:** 2026-06-19  
**Status:** Approved  
**Goal:** Add a Helm chart for Clonarr to the GnorpCharts repository to replace Recyclarr

## Background

Clonarr is a visual TRaSH Guides sync tool for Radarr and Sonarr that provides a web-based UI for managing quality profiles, custom formats, and sync operations. It replaces Recyclarr, which is currently deployed as a CronJob in the GnorpCharts repository.

Unlike Recyclarr (which runs periodically via CronJob), Clonarr is a long-running web service with:
- Web UI on port 6060
- Built-in authentication (requires setup on first run)
- Auto-sync capabilities with configurable behavior
- Persistent storage for configuration, profiles, and TRaSH Guides cache

## Design Decisions

### Deployment Pattern
**Decision:** Full Deployment pattern (similar to radarr/sonarr charts)  
**Rationale:** Clonarr is a long-running web service, not a scheduled job. Users need continuous access to the UI for profile management and monitoring.

### Configuration Approach
**Decision:** PVC-only persistence, no ConfigMap  
**Rationale:** Clonarr is configured entirely through its web UI. All configuration is stored in `/config/` and persisted via PVC. No need for declarative YAML configuration.

### Authentication
**Decision:** Manual setup via `/setup` endpoint  
**Rationale:** Clonarr requires first-run setup through the web UI to create admin credentials. No support for environment-based credential configuration.

### Migration from Recyclarr
**Decision:** Deferred - not included in initial chart  
**Rationale:** Need to test Clonarr UI and evaluate manual reconfiguration effort before deciding on migration strategy. Recyclarr YAML import is supported by Clonarr, but migration complexity needs assessment.

## Chart Structure

### File Organization

```
charts/clonarr/
├── Chart.yaml
├── .helmignore
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── pvc.yaml
    └── serviceaccount.yaml
```

### Chart.yaml

- **name:** `clonarr`
- **description:** "Visual TRaSH Guides sync tool for Radarr and Sonarr"
- **type:** `application`
- **version:** `0.1.0` (initial release)
- **appVersion:** `"latest"` (tracks Clonarr releases)

## Component Details

### 1. Deployment

**Container Image:**
- Repository: `ghcr.io/prophetse7en/clonarr`
- Tag: `latest` (configurable via `image.tag`)
- Pull Policy: `IfNotPresent`

**Environment Variables:**
All Clonarr environment variables included in values.yaml:
- `TZ` - Timezone (default: `UTC`)
- `PUID` - User ID for file ownership (default: `99`)
- `PGID` - Group ID for file ownership (default: `100`)
- `PORT` - Web UI port (default: `6060`)
- `CONFIG_DIR` - Config directory path (default: `/config`)
- `DATA_DIR` - TRaSH Guides data directory (default: `${CONFIG_DIR}/data`)
- `URL_BASE` - URL path prefix for reverse proxy subpath hosting
- `TRUSTED_NETWORKS` - Comma-separated IPs/CIDRs for auth bypass
- `TRUSTED_PROXIES` - Comma-separated proxy IPs for reverse proxy trust

**Volume Mounts:**
- `/config` - Persistent storage (mapped to PVC)

**Deployment Strategy:**
- `Recreate` - Prevents multiple pods from accessing the same PVC simultaneously (following radarr/sonarr pattern)

**Replica Count:**
- `1` - Single replica (stateful application with PVC)

### 2. Service

- **Type:** `ClusterIP` (default, configurable)
- **Port:** `6060`
- **Target Port:** `6060`
- **Protocol:** `TCP`

### 3. Ingress

- **Enabled:** `true` (default)
- **Class:** `traefik`
- **Host:** `clonarr.gnorplabs.cc`
- **Path:** `/`
- **PathType:** `ImplementationSpecific`
- **TLS:** Optional (empty by default)
- **Annotations:** Empty by default (user configurable)

### 4. PersistentVolumeClaim

**Configuration:**
- **Storage Class:** Empty string (uses cluster default)
- **Size:** `10Gi`
- **Access Mode:** `ReadWriteOnce`
- **Optional:** `claimName` override for existing PVCs

**Storage Contents:**
- `/config/clonarr.json` - Main configuration (instances, notifications)
- `/config/auth.json` - User credentials (bcrypt-hashed)
- `/config/sessions.json` - Session data
- `/config/profiles/` - Profile sync rules and history
- `/config/data/trash-guides/` - Cloned TRaSH Guides repository

**Size Rationale:** 10Gi is smaller than Recyclarr's 20Gi because Clonarr stores data more efficiently (JSON vs large YAML files). Provides headroom for TRaSH Guides repository, profile history, and future growth.

### 5. Health Checks

**Liveness Probe:**
- Type: HTTP GET
- Path: `/`
- Port: `6060`
- Initial Delay: `30s` (allow time for TRaSH repo clone on first boot)
- Period: `10s`
- Timeout: `5s`
- Failure Threshold: `3`

**Readiness Probe:**
- Same configuration as liveness probe

### 6. ServiceAccount

- **Create:** `true` (default)
- **Automount:** `true`
- **Annotations:** Empty (user configurable)
- **Name:** Auto-generated from chart fullname (user can override)

### 7. Resources

- **Limits:** Empty by default (user configurable)
- **Requests:** Empty by default (user configurable)
- **Rationale:** Clonarr is lightweight (Go backend + Alpine.js frontend). Let Kubernetes scheduler decide unless user has specific requirements.

## Values.yaml Structure

```yaml
# Image configuration
image:
  repository: ghcr.io/prophetse7en/clonarr
  pullPolicy: IfNotPresent
  tag: "latest"

# Deployment settings
replicaCount: 1
namespace: "plex"

# Environment variables (all Clonarr env vars included)
env:
  TZ: "UTC"
  PUID: "99"
  PGID: "100"
  # Optional: PORT, CONFIG_DIR, DATA_DIR, URL_BASE, TRUSTED_NETWORKS, TRUSTED_PROXIES

# Service configuration
service:
  type: ClusterIP
  port: 6060

# Ingress configuration
ingress:
  enabled: true
  className: "traefik"
  annotations: {}
  hosts:
    - host: clonarr.gnorplabs.cc
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# Persistence
persistence:
  config:
    # claimName: ""  # Optional: use existing PVC
    storageClass: ""
    size: 10Gi
    accessMode: ReadWriteOnce

# ServiceAccount
serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

# Pod configuration
podAnnotations: {}
podLabels: {}
podSecurityContext: {}
securityContext: {}

# Resources
resources: {}

# Health probes
livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

# Additional configuration
nodeSelector: {}
tolerations: []
affinity: {}
imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""
```

## Template Implementation

### _helpers.tpl

Standard Helm template helpers following GnorpCharts patterns:
- Chart name
- Fullname
- Chart label
- Common labels
- Selector labels
- ServiceAccount name

### deployment.yaml

- Uses `apps/v1` Deployment
- Strategy: `Recreate`
- Single replica
- Pod labels from values + chart labels
- Container with image from values
- Environment variables from values.env
- Volume mount for `/config`
- Liveness and readiness probes
- Optional resources, security context, node selector, tolerations, affinity

### service.yaml

- Standard ClusterIP service
- Port 6060 targeting container port 6060
- Selector matches deployment labels

### ingress.yaml

- Conditional on `ingress.enabled`
- Uses `networking.k8s.io/v1`
- Ingress class from values
- Host and path configuration from values
- Optional TLS configuration

### pvc.yaml

- Conditional creation (only if `persistence.config.claimName` is empty)
- Uses storage class, size, and access mode from values
- Standard labels

### serviceaccount.yaml

- Conditional on `serviceAccount.create`
- Uses name from values or generated from helpers
- Optional annotations

## First-Run Experience

1. Deploy chart via Helm
2. Access `http://clonarr.gnorplabs.cc` (or via ingress)
3. Automatic redirect to `/setup`
4. Create admin username and password (min 10 chars, complexity requirements)
5. Redirected to main UI
6. Add Radarr/Sonarr instances via Settings
7. Pull TRaSH Guides repository
8. Begin syncing profiles

## Testing Plan

1. Deploy to cluster with default values
2. Access web UI and complete initial setup
3. Add test Radarr/Sonarr instances
4. Test profile sync functionality
5. Evaluate manual configuration effort
6. Determine migration strategy from Recyclarr based on UI experience

## Future Enhancements

### Potential Migration Support (Post-Testing)

If manual reconfiguration is too complex, consider:

**Option 1: ConfigMap-based Import**
- Add optional ConfigMap for Recyclarr YAML
- InitContainer checks for ConfigMap presence
- If found and `/config/auth.json` doesn't exist (fresh install), import YAML via Clonarr API
- User-friendly for GitOps workflows

**Option 2: Documentation Only**
- Provide migration guide in chart README
- Users manually export from Recyclarr, import via Clonarr UI
- Simpler chart, more user effort

Decision deferred until Clonarr UI testing is complete.

## Maintenance Considerations

- **Chart versioning:** Follow SemVer, increment on any template/values changes
- **AppVersion tracking:** Keep aligned with Clonarr releases
- **Dependency:** No external chart dependencies
- **Compatibility:** Kubernetes 1.19+ (networking.k8s.io/v1 Ingress)
- **Recyclarr deprecation:** After successful Clonarr deployment and migration strategy determined, plan Recyclarr chart deprecation

## Security Considerations

- **Credentials:** Stored in PVC (`/config/auth.json`), bcrypt-hashed
- **Sessions:** Persisted in PVC (`/config/sessions.json`)
- **Network security:** Ingress can be disabled, service remains ClusterIP
- **TLS:** User configurable via ingress.tls
- **Trusted networks:** Configurable via env vars for auth bypass (use carefully)
- **API keys:** Clonarr generates API key on first setup, stored in config
- **PVC security:** Consider using encrypted storage classes for sensitive data

## Success Criteria

- Chart follows GnorpCharts patterns (structure, naming, labels)
- Deploys successfully to Kubernetes cluster
- Web UI accessible via ingress
- Configuration persists across pod restarts
- Health checks pass after initialization
- All Clonarr features functional (instance management, profile sync, auto-sync)
- Documentation clear for first-time users
