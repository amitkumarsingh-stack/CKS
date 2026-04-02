**Question 1:** Use context: ``kubectl config use-context workload-prod``
The Kubernetes Dashboard is installed in Namespace ``kubernetes-dashboard`` and is configured to:

* Allow users to "skip login"
* Allow insecure access (HTTP without authentication)
* Allow basic authentication
* Allow access from outside the cluster

You are asked to make it more secure by:

* Deny users to "skip login"
* Deny insecure access, enforce HTTPS (self signed certificates are ok for now)
* Add the --auto-generate-certificates argument
* Enforce authentication using a token (with possibility to use RBAC)
* Allow only cluster internal access

**Step 1 — Switch context**
```
kubectl config use-context workload-prod
```

**Step 2 — Investigate the existing Dashboard setup**
```
# Find the dashboard deployment
kubectl get all -n kubernetes-dashboard
```

```
# Check the existing deployment args
kubectl get deployment kubernetes-dashboard \
  -n kubernetes-dashboard -o yaml
```

You'll see something like this in the current (insecure) state:
```
containers:
- name: kubernetes-dashboard
  args:
  - --namespace=kubernetes-dashboard
  - --enable-skip-login          # ← insecure: allows skip
  - --enable-insecure-login      # ← insecure: HTTP allowed
  - --authentication-mode=basic  # ← insecure: basic auth
```

Also check the Service type:
```
kubectl get svc -n kubernetes-dashboard
# Currently: Type = NodePort  ← accessible from outside
```

**Step 3 — Edit the Dashboard Deployment**
```
kubectl edit deployment kubernetes-dashboard \
  -n kubernetes-dashboard
```
Find the ``args`` section under ``containers`` and make these changes:
```
# BEFORE (insecure)
containers:
- name: kubernetes-dashboard
  args:
  - --namespace=kubernetes-dashboard
  - --enable-skip-login
  - --enable-insecure-login
  - --authentication-mode=basic

# AFTER (secure)
containers:
- name: kubernetes-dashboard
  args:
  - --namespace=kubernetes-dashboard
  - --auto-generate-certificates        # fix 3: HTTPS with self-signed cert
  - --authentication-mode=token         # fix 4: token auth (supports RBAC)
```

Save and exit — Kubernetes will do a rolling restart.

**Step 4 — Change Service from NodePort to ClusterIP**
```
kubectl edit svc kubernetes-dashboard \
  -n kubernetes-dashboard
```

Change:
```
# BEFORE
spec:
  type: NodePort
  ports:
  - port: 443
    nodePort: 32000    # ← remove this line

# AFTER
spec:
  type: ClusterIP      # ← fix 5: only internal access
  ports:
  - port: 443
```
Save and exit.

**Step 5 — Verify dashboard is no longer externally accessible**
```
# Should NOT work from outside
curl -k https://<node-ip>:<old-nodeport>
# Connection refused ✅

# Should work from inside cluster only
kubectl run test-curl --image=curlimages/curl -it --rm \
  -- curl -k https://kubernetes-dashboard.kubernetes-dashboard.svc
# Should get a response ✅
```

---

## Summary of All 5 Fixes

| # | Security Issue | Fix | How |
|---|---|---|---|
| 1 | Skip login allowed | Remove `--enable-skip-login` | Edit deployment args |
| 2 | HTTP insecure access | Remove `--enable-insecure-login` | Edit deployment args |
| 3 | No HTTPS | Add `--auto-generate-certificates` | Edit deployment args |
| 4 | Basic auth | Change to `--authentication-mode=token` | Edit deployment args |
| 5 | External access via NodePort | Change to `ClusterIP` | Edit service type |

---

## Key Concepts Tested

| Concept | Detail |
|---|---|
| **Dashboard args** | Flags that control authentication and access mode |
| **ClusterIP vs NodePort** | ClusterIP = internal only, NodePort = external access |
| **Token authentication** | Required for RBAC integration with dashboard |
| **Self-signed certs** | `--auto-generate-certificates` handles this automatically |
| **Defence in depth** | Multiple layers: auth + transport + network access |

---

## Docs to Bookmark
```
# Kubernetes Dashboard arguments reference
https://github.com/kubernetes/dashboard/blob/master/docs/common/arguments.md

# Kubernetes Service types
https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
