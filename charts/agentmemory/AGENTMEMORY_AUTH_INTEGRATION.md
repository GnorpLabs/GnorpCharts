# AgentMemory Authentication Integration Plan

## Overview

AgentMemory now supports auto-generated authentication secrets. This document outlines how to integrate the authenticated AgentMemory with mcp-gateway.

## Changes Made to AgentMemory Chart

### 1. Auto-Generated Secret
- **Location:** `templates/secret.yaml`
- **Behavior:** Generates a random 32-character secret on first install
- **Annotation:** `helm.sh/resource-policy: keep` - prevents deletion on upgrade
- **Secret Name:** `<release-name>-auth`
- **Secret Key:** `agentmemory-secret`

### 2. Values Configuration
```yaml
auth:
  enabled: true              # Enable authentication (default: true)
  autoGenerate: true         # Auto-generate secret (default: true)
  existingSecret: ""         # Use existing secret (optional)
  secret: ""                 # Manually specify secret (if autoGenerate=false)

env:
  AGENTMEMORY_VIEWER_HOST: "0.0.0.0"  # Bind viewer to all interfaces when auth enabled
```

### 3. Deployment Changes
- Injects `AGENTMEMORY_SECRET` from Kubernetes Secret when `auth.enabled=true`
- Sets `AGENTMEMORY_VIEWER_HOST=0.0.0.0` when auth is enabled
- Disables `viewerProxy` socat sidecar when auth is enabled
- Uses standard HTTP ingress instead of TCP ingress when auth is enabled

## Integration Options for mcp-gateway

### Option 1: Token Replacement Pattern (Recommended)
**Approach:** Store the secret in GitHub Actions secrets and use `__TOKEN__` placeholders that get replaced by `qetza/replacetokens-action` during deployment. This matches the existing pattern used for `__OMNIROUTE_MCP_API_KEY__`.

**Pros:**
- Single source of truth (GitHub Secrets)
- No coupling between charts
- GitOps-friendly
- Easy secret rotation
- No RBAC complexity
- Works with any CI/CD system

**Cons:**
- Secret visible in Helm values (but not committed to git)
- Requires CI/CD pipeline

**Implementation:**

#### Step 1: Initial Deploy (auto-generate secret)
```yaml
# GnorpLabs/helm/helmfile/values/infra/values-agentmemory.yaml
auth:
  enabled: true
  autoGenerate: true  # AgentMemory generates the secret
```

Deploy, then extract:
```bash
kubectl get secret agentmemory-auth -n mcp-gateway \
  -o jsonpath='{.data.agentmemory-secret}' | base64 -d
```

#### Step 2: Add to GitHub Secrets
- Settings → Secrets → Actions → `AGENTMEMORY_SECRET`
- Paste the auto-generated value from Step 1

#### Step 3: Update values-agentmemory.yaml to use token replacement
```yaml
# GnorpLabs/helm/helmfile/values/infra/values-agentmemory.yaml
auth:
  enabled: true
  autoGenerate: false
  secret: "__AGENTMEMORY_SECRET__"
```

#### Step 3: Update values-mcp-gateway.yaml
```yaml
# GnorpLabs/helm/helmfile/values/infra/values-mcp-gateway.yaml
gateway:
  backends:
    agentmemory:
      http_url: "http://agentmemory.mcp-gateway.svc.cluster.local:3114/mcp"
      description: "Persistent memory for AI agents"
      streamable_http: true
      headers:
        Authorization: "Bearer __AGENTMEMORY_SECRET__"
```

#### Step 4: Existing Workflow Handles Token Replacement
Your existing `qetza/replacetokens-action` step already handles this:
```yaml
# .github/workflows/deploy.yml (existing)
- name: Replace tokens
  uses: qetza/replacetokens-action@v3
  with:
    files: 'helm/helmfile/values/**/*.yaml'
    tokenPrefix: '__'
    tokenSuffix: '__'
```

The `__AGENTMEMORY_SECRET__` tokens are automatically replaced with the GitHub Secret value.

**See `DEPLOYMENT.md` for complete guide.**

### Option 2: Kubernetes Secret Reference (Alternative)
**Approach:** mcp-gateway reads the Kubernetes Secret created by AgentMemory.

**Pros:**
- Kubernetes-native
- No external dependencies

**Cons:**
- Tight coupling between charts
- Requires RBAC permissions for cross-secret access
- More complex than GitHub Actions approach

**Implementation:** See previous version of this doc or use Option 1 instead.

### Option 3: ServiceAccount Token Projection (Future)
**Approach:** Use Kubernetes ServiceAccount tokens for authentication.

**Status:** Not currently supported by AgentMemory v0.11.2 (requires upstream changes)

### Option 4: Separate Secrets (Not Recommended)
**Approach:** Create two separate secrets with the same value.

**Cons:**
- Manual sync required
- Easy to drift
- Violates DRY principle

## Recommended Implementation Steps

### 1. Deploy AgentMemory with auth enabled
```bash
helm upgrade --install agentmemory ./charts/agentmemory \
  -n mcp-gateway \
  --set auth.enabled=true \
  --set auth.autoGenerate=true
```

### 2. Retrieve the generated secret
```bash
kubectl get secret agentmemory-auth -n mcp-gateway -o jsonpath='{.data.agentmemory-secret}' | base64 -d
```

### 3. Update mcp-gateway values
```yaml
# values/mcp-gateway.yaml
agentmemory:
  secretRef:
    name: "agentmemory-auth"
    key: "agentmemory-secret"
  url: "http://agentmemory.mcp-gateway.svc.cluster.local:3111"
```

### 4. Update mcp-gateway deployment template
Add the environment variables to read the secret (see Option 1 above).

### 5. Configure mcp-gateway backend
Update the AgentMemory backend in mcp-gateway's config to include auth headers.

### 6. Deploy mcp-gateway
```bash
helm upgrade --install mcp-gateway ./charts/mcp-gateway \
  -n mcp-gateway \
  -f values/mcp-gateway.yaml
```

## Verification

### 1. Check AgentMemory secret exists
```bash
kubectl get secret agentmemory-auth -n mcp-gateway
```

### 2. Test authenticated access
```bash
# Get the secret value
SECRET=$(kubectl get secret agentmemory-auth -n mcp-gateway -o jsonpath='{.data.agentmemory-secret}' | base64 -d)

# Test API with auth
kubectl run -it --rm curl --image=curlimages/curl -n mcp-gateway -- \
  curl -H "Authorization: Bearer $SECRET" \
  http://agentmemory.mcp-gateway.svc.cluster.local:3111/agentmemory/health
```

### 3. Check mcp-gateway can access AgentMemory
```bash
kubectl logs -n mcp-gateway deployment/mcp-gateway | grep -i agentmemory
```

## Migration from No-Auth Setup

If you're migrating from the previous no-auth setup:

1. **Before migration:** Document current mcp-gateway configuration
2. **Enable auth on AgentMemory:**
   ```bash
   helm upgrade agentmemory ./charts/agentmemory \
     -n mcp-gateway \
     --set auth.enabled=true \
     --reuse-values
   ```
3. **Update mcp-gateway** to use the secret (see steps above)
4. **Verify functionality** before removing old TCP ingress configuration
5. **Clean up:** Remove TCP ingress resources if no longer needed

## Security Considerations

1. **Secret Rotation:** To rotate the secret:
   ```bash
   # Delete the secret (will be regenerated on next upgrade)
   kubectl delete secret agentmemory-auth -n mcp-gateway
   
   # Upgrade to regenerate
   helm upgrade agentmemory ./charts/agentmemory -n mcp-gateway
   
   # Restart mcp-gateway to pick up new secret
   kubectl rollout restart deployment/mcp-gateway -n mcp-gateway
   ```

2. **RBAC:** Ensure mcp-gateway ServiceAccount has permission to read the secret:
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: Role
   metadata:
     name: mcp-gateway-secret-reader
     namespace: mcp-gateway
   rules:
   - apiGroups: [""]
     resources: ["secrets"]
     resourceNames: ["agentmemory-auth"]
     verbs: ["get"]
   ```

3. **Network Policies:** With auth enabled, consider restricting network access:
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: agentmemory-api-access
     namespace: mcp-gateway
   spec:
     podSelector:
       matchLabels:
         app.kubernetes.io/name: agentmemory
     policyTypes:
     - Ingress
     ingress:
     - from:
       - podSelector:
           matchLabels:
             app.kubernetes.io/name: mcp-gateway
       ports:
       - protocol: TCP
         port: 3111
   ```

## Troubleshooting

### mcp-gateway can't authenticate
- Verify secret exists: `kubectl get secret agentmemory-auth -n mcp-gateway`
- Check secret is mounted: `kubectl describe pod <mcp-gateway-pod> -n mcp-gateway`
- Verify secret value: `kubectl get secret agentmemory-auth -n mcp-gateway -o jsonpath='{.data.agentmemory-secret}' | base64 -d`

### Viewer not accessible
- Check ingress is HTTP (not TCP): `kubectl get ingress -n mcp-gateway`
- Verify `AGENTMEMORY_VIEWER_HOST=0.0.0.0` is set in pod env
- Check auth is enabled: `helm get values agentmemory -n mcp-gateway`

### Secret regenerates on every upgrade
- Ensure `helm.sh/resource-policy: keep` annotation is present
- Use `--reuse-values` flag during upgrades
- Consider using `existingSecret` for production

## Next Steps

1. Implement Option 1 (recommended) in mcp-gateway chart
2. Test in staging environment
3. Document upgrade path for production
4. Consider implementing NetworkPolicies for additional security
5. Plan for secret rotation strategy
