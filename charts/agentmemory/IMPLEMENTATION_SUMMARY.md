# AgentMemory Authentication Implementation Summary

## What Was Done

Implemented auto-generated authentication for AgentMemory Helm chart with GitHub Actions integration for mcp-gateway.

## Files Changed

### AgentMemory Chart

1. **`charts/agentmemory/values.yaml`**
   - Added `auth` configuration block
   - Default: `auth.enabled=true`, `auth.autoGenerate=true`
   - Added `env.AGENTMEMORY_VIEWER_HOST: "0.0.0.0"`

2. **`charts/agentmemory/templates/secret.yaml`** (new)
   - Auto-generates 32-char random secret
   - Uses `helm.sh/resource-policy: keep` annotation
   - Secret name: `<release>-auth`

3. **`charts/agentmemory/templates/deployment.yaml`**
   - Injects `AGENTMEMORY_SECRET` from secret when auth enabled
   - Sets `AGENTMEMORY_VIEWER_HOST` when auth enabled
   - Conditionally disables viewerProxy sidecar

4. **`charts/agentmemory/templates/ingress.yaml`**
   - Only renders when `auth.enabled=true`

5. **`charts/agentmemory/templates/ingress-tcp.yaml`**
   - Only renders when `auth.enabled=false`

6. **`charts/agentmemory/README.md`**
   - Added authentication parameters table
   - Updated security section

### mcp-gateway Chart

1. **`charts/mcp-gateway/values.yaml`**
   - Added `gateway.agentmemory.secret` (injected via GitHub Actions)
   - Added `gateway.agentmemory.url`

2. **`charts/mcp-gateway/templates/deployment.yaml`**
   - Injects `AGENTMEMORY_SECRET` and `AGENTMEMORY_URL` env vars
   - Only when `gateway.agentmemory.secret` is set

### Documentation

1. **`AGENTMEMORY_AUTH_INTEGRATION.md`** (new)
   - Comprehensive integration guide
   - Three implementation options
   - Troubleshooting section

2. **`GITHUB_ACTIONS_DEPLOYMENT.md`** (new)
   - Complete GitHub Actions workflow examples
   - Secret rotation procedures
   - Local development guide
   - Security best practices

3. **`IMPLEMENTATION_SUMMARY.md`** (this file)

## How It Works

### Architecture

```
┌──────────────────────────────┐
│  GitHub Actions Secret       │
│  AGENTMEMORY_SECRET          │
└───────────┬──────────────────┘
            │
            ├─────────────────────┐
            │                     │
            ▼                     ▼
    ┌───────────────┐     ┌──────────────┐
    │ AgentMemory   │     │ mcp-gateway  │
    │ Deployment    │     │ Deployment   │
    ├───────────────┤     ├──────────────┤
    │ --set-string  │     │ --set-string │
    │ auth.secret=  │     │ gateway.     │
    │ $SECRET       │     │ agentmemory. │
    │               │     │ secret=$SEC  │
    └───────┬───────┘     └──────┬───────┘
            │                    │
            ▼                    │
    ┌───────────────┐            │
    │ Pod: agentmem │            │
    │ ENV:          │            │
    │ AGENTMEMORY_  │            │
    │ SECRET=xxx    │◄───────────┤
    │               │    HTTP w/ │
    │ API: :3111    │    Bearer  │
    │ (auth req'd)  │    Token   │
    └───────────────┘            │
                                 │
                         ┌───────▼───────┐
                         │ Pod: mcp-gw   │
                         │ ENV:          │
                         │ AGENTMEMORY_  │
                         │ SECRET=xxx    │
                         └───────────────┘
```

### Deployment Flow

1. **Generate Secret**: `openssl rand -base64 32`
2. **Store in GitHub**: Settings → Secrets → `AGENTMEMORY_SECRET`
3. **Deploy AgentMemory**: Inject secret via `--set-string auth.secret="${{ secrets.AGENTMEMORY_SECRET }}"`
4. **Deploy mcp-gateway**: Inject same secret via `--set-string gateway.agentmemory.secret="${{ secrets.AGENTMEMORY_SECRET }}"`
5. **mcp-gateway authenticates**: Uses env var `AGENTMEMORY_SECRET` as Bearer token

## Key Design Decisions

### Why GitHub Actions Secrets?

**Rejected Approaches:**
- ❌ **Auto-generated K8s Secret + secretRef**: Tight coupling, RBAC complexity
- ❌ **Separate secrets**: Sync issues, violates DRY
- ❌ **ServiceAccount tokens**: Not supported by AgentMemory v0.11.2

**Selected Approach:**
- ✅ **GitHub Actions Secret Injection**: Single source of truth, GitOps-friendly, no coupling

### Why Default Auth to Enabled?

**Previous state**: `auth.enabled=false` (legacy no-auth mode)
- Required socat sidecar hack
- No protection for API or viewer
- Confusing documentation

**New state**: `auth.enabled=true` by default
- Secure by default
- Standard HTTP ingress
- No sidecar complexity
- Opt-in to no-auth if needed

### Why Not Commit Secrets to values.yaml?

**Security principle**: Secrets should never be in version control.

**Implementation**:
- `values.yaml` has `secret: ""` (empty default)
- Secrets injected at deployment time via `--set-string`
- `.gitignore` protects local `values.local.yaml` files

## Usage Examples

### Deploy with GitHub Actions (Production)

```yaml
# .github/workflows/deploy.yml
- name: Deploy Stack
  run: |
    helm upgrade --install agentmemory ./charts/agentmemory \
      --namespace mcp-gateway \
      --create-namespace \
      --set auth.enabled=true \
      --set auth.autoGenerate=false \
      --set-string auth.secret="${{ secrets.AGENTMEMORY_SECRET }}"
    
    helm upgrade --install mcp-gateway ./charts/mcp-gateway \
      --namespace mcp-gateway \
      --set-string gateway.agentmemory.secret="${{ secrets.AGENTMEMORY_SECRET }}"
```

### Deploy Locally (Development)

```bash
# Set in environment
export AGENTMEMORY_SECRET="local-dev-secret-do-not-use-in-prod"

# Deploy
helm upgrade --install agentmemory ./charts/agentmemory \
  -n mcp-gateway --create-namespace \
  --set auth.enabled=true \
  --set auth.autoGenerate=false \
  --set-string auth.secret="$AGENTMEMORY_SECRET"

helm upgrade --install mcp-gateway ./charts/mcp-gateway \
  -n mcp-gateway \
  --set-string gateway.agentmemory.secret="$AGENTMEMORY_SECRET"
```

### Disable Auth (Legacy Mode)

```bash
helm upgrade --install agentmemory ./charts/agentmemory \
  -n mcp-gateway \
  --set auth.enabled=false \
  --set viewerProxy.enabled=true \
  --set ingressTCP.enabled=true \
  --set ingress.enabled=false
```

## Testing

### 1. Verify Secret Injection

```bash
# AgentMemory pod
kubectl exec -it deployment/agentmemory -n mcp-gateway -- env | grep AGENTMEMORY_SECRET

# mcp-gateway pod
kubectl exec -it deployment/mcp-gateway -n mcp-gateway -- env | grep AGENTMEMORY_SECRET

# Both should show the same value
```

### 2. Test Authenticated Access

```bash
# Get secret from GitHub (or local env)
SECRET="${AGENTMEMORY_SECRET}"

# Test from outside cluster
kubectl run -it --rm curl --image=curlimages/curl -n mcp-gateway -- \
  curl -H "Authorization: Bearer $SECRET" \
  http://agentmemory.mcp-gateway.svc.cluster.local:3111/agentmemory/health

# Should return: {"status":"healthy"}
```

### 3. Test Unauthorized Access (Should Fail)

```bash
# Without auth header
kubectl run -it --rm curl --image=curlimages/curl -n mcp-gateway -- \
  curl http://agentmemory.mcp-gateway.svc.cluster.local:3111/agentmemory/health

# Should return: 401 Unauthorized
```

### 4. Test mcp-gateway Integration

```bash
# Check mcp-gateway logs
kubectl logs -n mcp-gateway deployment/mcp-gateway --tail=50

# Look for successful AgentMemory backend connection
# Should see no auth errors
```

## Migration Path

### From No-Auth (v1) to Auth-Enabled

```bash
# 1. Generate secret
SECRET=$(openssl rand -base64 32)

# 2. Add to GitHub Secrets: AGENTMEMORY_SECRET

# 3. Redeploy AgentMemory with auth
helm upgrade agentmemory ./charts/agentmemory \
  -n mcp-gateway \
  --set auth.enabled=true \
  --set auth.autoGenerate=false \
  --set-string auth.secret="$SECRET" \
  --reuse-values

# 4. Update mcp-gateway
helm upgrade mcp-gateway ./charts/mcp-gateway \
  -n mcp-gateway \
  --set-string gateway.agentmemory.secret="$SECRET"

# 5. Verify both pods restart and connect
kubectl rollout status deployment/agentmemory -n mcp-gateway
kubectl rollout status deployment/mcp-gateway -n mcp-gateway
```

### From Auto-Generated to GitHub Actions

```bash
# 1. Extract existing auto-generated secret
EXISTING=$(kubectl get secret agentmemory-auth -n mcp-gateway \
  -o jsonpath='{.data.agentmemory-secret}' | base64 -d)

# 2. Add to GitHub Secrets (use $EXISTING value)

# 3. Redeploy with autoGenerate=false
helm upgrade agentmemory ./charts/agentmemory \
  -n mcp-gateway \
  --set auth.enabled=true \
  --set auth.autoGenerate=false \
  --set-string auth.secret="$EXISTING"

# 4. Optional: Delete old K8s secret
kubectl delete secret agentmemory-auth -n mcp-gateway
```

## Secret Rotation

```bash
# 1. Let AgentMemory generate a new secret
kubectl delete secret agentmemory-auth -n mcp-gateway

helm upgrade agentmemory ./charts/agentmemory \
  -n mcp-gateway \
  --set auth.enabled=true \
  --set auth.autoGenerate=true

# 2. Extract the new secret
NEW_SECRET=$(kubectl get secret agentmemory-auth -n mcp-gateway \
  -o jsonpath='{.data.agentmemory-secret}' | base64 -d)

# 3. Update GitHub Secret: AGENTMEMORY_SECRET with $NEW_SECRET

# 4. Trigger redeployment with token replacement
git commit --allow-empty -m "Rotate AgentMemory secret"
git push
```

## Security Considerations

1. **Never commit secrets to git**
   - Use `--set-string` for injection
   - Add `values.local.yaml` to `.gitignore`

2. **Use GitHub Environments for production**
   - Requires manual approval
   - Separates dev/staging/prod secrets

3. **Enable secret scanning**
   - GitHub Settings → Code security → Secret scanning

4. **Rotate secrets regularly**
   - Recommended: every 90 days
   - Automated rotation: future enhancement

5. **Limit secret access**
   - GitHub: Restrict who can manage Actions secrets
   - Kubernetes: Use RBAC to limit secret access

## Next Steps

1. ✅ **Implement** - Complete (this PR)
2. 🔄 **Test** - Deploy to dev environment
3. 📝 **Document** - Update main repo README
4. 🚀 **Deploy** - Roll out to production
5. 🔒 **Harden** - Add NetworkPolicies (future)
6. 🔄 **Automate** - Secret rotation automation (future)

## Related Documentation

- `AGENTMEMORY_AUTH_INTEGRATION.md` - Detailed integration guide
- `GITHUB_ACTIONS_DEPLOYMENT.md` - CI/CD examples and best practices
- `README.md` - Chart overview and basic usage
- `values.yaml` - All configuration options with comments
