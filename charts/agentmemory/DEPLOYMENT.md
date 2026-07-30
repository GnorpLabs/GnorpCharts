# AgentMemory Deployment Guide

## Secret Management

AgentMemory supports authentication with an auto-generated secret. The secret can be shared with mcp-gateway using **two patterns** - choose the one that fits your workflow.

## Two Deployment Patterns

### Option A: Kubernetes Secret (Recommended for Cluster)

**Fully automated - zero manual steps.**

AgentMemory auto-generates a secret, stores it in K8s Secret `agentmemory-auth`, and mcp-gateway reads it via `envFromSecrets`.

#### AgentMemory values
```yaml
# GnorpLabs/helm/helmfile/values/infra/values-agentmemory.yaml
auth:
  enabled: true
  autoGenerate: true  # Auto-generates on first install
```

#### mcp-gateway values
```yaml
# GnorpLabs/helm/helmfile/values/infra/values-mcp-gateway.yaml
gateway:
  envFromSecrets:
    - name: AGENTMEMORY_SECRET
      secretName: agentmemory-auth
      secretKey: agentmemory-secret
  backends:
    agentmemory:
      http_url: "http://agentmemory.mcp-gateway.svc.cluster.local:3114/mcp"
      streamable_http: true
      headers:
        Authorization: "Bearer ${AGENTMEMORY_SECRET}"
```

**Deploy and done.** No extraction, no GitHub Secrets, no token replacement needed.

---

### Option B: Token Replacement (GitOps Pattern)

**Best for secret rotation via GitHub Actions.**

#### Step 1: Deploy AgentMemory (auto-generates secret)
```yaml
# values-agentmemory.yaml
auth:
  enabled: true
  autoGenerate: true
```

#### Step 2: Extract the generated secret
```bash
kubectl get secret agentmemory-auth -n mcp-gateway \
  -o jsonpath='{.data.agentmemory-secret}' | base64 -d
```

#### Step 3: Add to GitHub Secrets
- Settings → Secrets → Actions → `AGENTMEMORY_SECRET`

#### Step 4: Update values to use token replacement
```yaml
# values-agentmemory.yaml
auth:
  enabled: true
  autoGenerate: false
  secret: "__AGENTMEMORY_SECRET__"

# values-mcp-gateway.yaml
gateway:
  backends:
    agentmemory:
      headers:
        Authorization: "Bearer __AGENTMEMORY_SECRET__"
```

Your existing `qetza/replacetokens-action` handles the replacement.

#### mcp-gateway (`values-mcp-gateway.yaml`)
```yaml
gateway:
  backends:
    agentmemory:
      http_url: "http://agentmemory.mcp-gateway.svc.cluster.local:3114/mcp"
      description: "Persistent memory for AI agents"
      streamable_http: true
      headers:
        Authorization: "Bearer __AGENTMEMORY_SECRET__"
```

### Existing Deployment Workflow

Your existing workflow already handles token replacement:

```yaml
# .github/workflows/deploy.yml (your existing workflow)
- name: Replace tokens
  uses: qetza/replacetokens-action@v3
  with:
    files: |
      helm/helmfile/values/**/*.yaml
    tokenPrefix: '__'
    tokenSuffix: '__'

# Then your normal helm/helmfile deployment
- name: Deploy
  run: helmfile apply
```

The `__AGENTMEMORY_SECRET__` tokens in both values files will be replaced with the actual secret from GitHub Secrets.

## Local Development

For local testing without token replacement:

```bash
# Set environment variable
export AGENTMEMORY_SECRET="local-dev-secret-change-me"

# Deploy agentmemory
helm upgrade --install agentmemory ./charts/agentmemory \
  -n mcp-gateway --create-namespace \
  --set auth.enabled=true \
  --set auth.autoGenerate=false \
  --set-string auth.secret="$AGENTMEMORY_SECRET"

# Update mcp-gateway values manually or use sed
sed "s/__AGENTMEMORY_SECRET__/$AGENTMEMORY_SECRET/g" \
  values/infra/values-mcp-gateway.yaml > /tmp/values-mcp-gateway-local.yaml

helm upgrade --install mcp-gateway ./charts/mcp-gateway \
  -n mcp-gateway \
  -f /tmp/values-mcp-gateway-local.yaml
```

## Secret Rotation

1. Generate new secret: `openssl rand -base64 32`
2. Update GitHub Secret: `AGENTMEMORY_SECRET`
3. Trigger deployment (push to main or manual workflow dispatch)
4. Both agentmemory and mcp-gateway will restart with new secret

## Verification

```bash
# Check agentmemory has the secret
kubectl exec -it deployment/agentmemory -n mcp-gateway -- env | grep AGENTMEMORY_SECRET

# Check mcp-gateway backend config
kubectl exec -it deployment/mcp-gateway -n mcp-gateway -- cat /config.yaml | grep -A 5 agentmemory

# Test authenticated access
kubectl run -it --rm curl --image=curlimages/curl -n mcp-gateway -- \
  curl -H "Authorization: Bearer $SECRET" \
  http://agentmemory.mcp-gateway.svc.cluster.local:3111/agentmemory/health
```
