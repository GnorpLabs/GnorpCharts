# Clonarr Helm Chart Deployment Test Guide

## Overview

This document provides a comprehensive testing guide for the Clonarr Helm chart. Use this checklist when deploying to a Kubernetes cluster to verify all functionality.

**Chart Version:** 0.1.0  
**Test Date:** _To be completed when cluster available_  
**Kubernetes Version:** _To be recorded during testing_

---

## Prerequisites

- Kubernetes cluster with kubectl access
- Helm 3.x installed
- Storage class available for PVC binding
- Ingress controller installed (e.g., traefik)

---

## Test Procedures

### 1. Deploy Chart to Cluster

```bash
helm install clonarr charts/clonarr -n plex --create-namespace
```

**Expected Result:**
```
NAME: clonarr
LAST DEPLOYED: [timestamp]
NAMESPACE: plex
STATUS: deployed
REVISION: 1
```

**Status:** ⬜ PASS / ⬜ FAIL

---

### 2. Verify All Resources Created

```bash
kubectl get all,pvc,ingress -n plex -l app.kubernetes.io/name=clonarr
```

**Expected Resources:**
- Deployment: `deployment.apps/clonarr`
- Pod: `pod/clonarr-xxxxx-xxxxx`
- Service: `service/clonarr`
- PVC: `persistentvolumeclaim/clonarr-config`
- Ingress: `ingress.networking.k8s.io/clonarr`

**Status:** ⬜ PASS / ⬜ FAIL

---

### 3. Check Pod Status

```bash
kubectl get pods -n plex -l app.kubernetes.io/name=clonarr
```

**Expected Result:**
- Pod STATUS: `Running` (after ~30 seconds for initial setup)
- READY: `1/1`

**Note:** Initial startup may take longer due to TRaSH repo clone.

**Status:** ⬜ PASS / ⬜ FAIL

---

### 4. Verify Health Checks

```bash
kubectl describe pod -n plex -l app.kubernetes.io/name=clonarr | grep -A5 "Liveness\|Readiness"
```

**Expected Configuration:**
- Liveness probe: http-get on port 6060 path /
- Readiness probe: http-get on port 6060 path /
- Initial delay: 30 seconds
- Period: 10 seconds
- Timeout: 5 seconds

**Status:** ⬜ PASS / ⬜ FAIL

---

### 5. Check PVC Binding

```bash
kubectl get pvc -n plex -l app.kubernetes.io/name=clonarr
```

**Expected Result:**
- NAME: `clonarr-config`
- STATUS: `Bound`
- CAPACITY: `10Gi`
- ACCESS MODES: `RWO` (ReadWriteOnce)

**Status:** ⬜ PASS / ⬜ FAIL

---

### 6. Verify Ingress Configuration

```bash
kubectl get ingress -n plex -l app.kubernetes.io/name=clonarr -o yaml
```

**Expected Configuration:**
- Host: `clonarr.gnorplabs.cc`
- Ingress class: `traefik`
- Backend service: `clonarr`
- Service port: `6060`
- Path: `/`
- PathType: `ImplementationSpecific`

**Status:** ⬜ PASS / ⬜ FAIL

---

### 7. Test Web UI Access

```bash
# Start port-forward
kubectl port-forward -n plex svc/clonarr 6060:6060
```

Then access: `http://localhost:6060`

**Expected Result:**
- Clonarr UI loads successfully
- Redirects to `/setup` (first run)
- UI is responsive

**Status:** ⬜ PASS / ⬜ FAIL

---

### 8. Verify Environment Variables

```bash
kubectl exec -n plex deployment/clonarr -- env | grep -E "TZ|PUID|PGID"
```

**Expected Values:**
- `TZ=UTC`
- `PUID=99`
- `PGID=100`

**Status:** ⬜ PASS / ⬜ FAIL

---

### 9. Check Logs for Errors

```bash
kubectl logs -n plex -l app.kubernetes.io/name=clonarr --tail=50
```

**Expected Result:**
- No error messages
- Successful startup logs
- Application running normally

**Status:** ⬜ PASS / ⬜ FAIL

---

### 10. Clean Up Test Deployment

```bash
helm uninstall clonarr -n plex
```

**Expected Result:**
```
release "clonarr" uninstalled
```

**Verify cleanup:**
```bash
kubectl get all,pvc,ingress -n plex -l app.kubernetes.io/name=clonarr
# Should return: No resources found
```

**Status:** ⬜ PASS / ⬜ FAIL

---

## Test Summary

### Results

| Test | Status | Notes |
|------|--------|-------|
| Chart installation | ⬜ PASS / ⬜ FAIL | |
| Resources created | ⬜ PASS / ⬜ FAIL | |
| Pod startup | ⬜ PASS / ⬜ FAIL | |
| Health checks | ⬜ PASS / ⬜ FAIL | |
| PVC binding | ⬜ PASS / ⬜ FAIL | |
| Ingress config | ⬜ PASS / ⬜ FAIL | |
| Web UI access | ⬜ PASS / ⬜ FAIL | |
| Environment vars | ⬜ PASS / ⬜ FAIL | |
| Clean logs | ⬜ PASS / ⬜ FAIL | |
| Cleanup | ⬜ PASS / ⬜ FAIL | |

### Overall Result

⬜ All tests passed - Chart ready for production  
⬜ Some tests failed - Issues require investigation  
⬜ Tests not run - No cluster available

### Issues Encountered

_Document any issues, errors, or unexpected behavior:_

---

### Environment Details

- **Kubernetes Version:** _____________________
- **Helm Version:** _____________________
- **Storage Class Used:** _____________________
- **Ingress Controller:** _____________________
- **Test Date:** _____________________
- **Tester:** _____________________

---

## Advanced Testing

### Test with Custom Values

```bash
# Create custom values file
cat > test-values.yaml << EOF
env:
  TZ: "America/New_York"
  PUID: "1000"
  PGID: "1000"

persistence:
  config:
    size: 20Gi
    storageClass: "fast-ssd"

ingress:
  hosts:
    - host: clonarr-test.example.com
      paths:
        - path: /
          pathType: Prefix
EOF

# Deploy with custom values
helm install clonarr-test charts/clonarr -n test --create-namespace -f test-values.yaml

# Verify custom values applied
kubectl get pvc -n test  # Should show 20Gi
kubectl exec -n test deployment/clonarr-test -- env | grep TZ  # Should show America/New_York
```

**Status:** ⬜ PASS / ⬜ FAIL

### Test Upgrade Scenario

```bash
# Install initial version
helm install clonarr charts/clonarr -n plex

# Make a change to values
helm upgrade clonarr charts/clonarr -n plex --set replicaCount=1

# Verify upgrade successful
helm history clonarr -n plex

# Rollback if needed
helm rollback clonarr 1 -n plex
```

**Status:** ⬜ PASS / ⬜ FAIL

---

## Conclusion

**Chart Deployment Status:** ⬜ Ready for Production / ⬜ Needs Work

**Recommendations:**

_Document any recommendations for improvements or next steps:_

---

**Tested by:** _____________________  
**Date:** _____________________  
**Signature:** _____________________
