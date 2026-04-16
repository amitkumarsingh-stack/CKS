**Question 1:**
The ``service-account-audit`` namespace contains an overprivileged service account named ``overprivileged-sa`` and an insecure deployment called ``insecure-app``.

Tasks:

1. Create a new secure service account named ``restricted-sa``, ensuring that automatic token mounting is disabled.
2. Create a minimal RBAC Role named ``restricted-role`` that permits only get and list operations on pods.
3. Create a RoleBinding named ``restricted-binding`` to bind the ``restricted-role`` to the ``restricted-sa`` service account.
4. Modify the existing deployment ``insecure-app`` to utilize the ``restricted-sa`` service account, incorporating a security context.

Requirements for the deployment modification:
* Use the ``restricted-sa`` service account.
* Disable automatic token mounting.
* Configure the security context to run as a ``non-root user (UID 101)``.
* Disable privilege escalation.
* Drop all Linux capabilities.
* Do not modify the existing ``overprivileged-sa``.

## Solution

**Step 1 — Create the restricted service account (token mounting disabled)**
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: restricted-sa
  namespace: service-account-audit
automountServiceAccountToken: false
EOF
```

**Step 2 — Create the minimal RBAC Role**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: restricted-role
  namespace: service-account-audit
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF
```

**Step 3 — Create the RoleBinding**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: restricted-binding
  namespace: service-account-audit
subjects:
- kind: ServiceAccount
  name: restricted-sa
  namespace: service-account-audit
roleRef:
  kind: Role
  apiGroup: rbac.authorization.k8s.io
  name: restricted-role
EOF
```

**Step 4 — Patch the insecure-app deployment**
```
kubectl patch deployment insecure-app \
  -n service-account-audit \
  --type=strategic -p='
spec:
  template:
    spec:
      serviceAccountName: restricted-sa
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
      containers:
      - name: insecure-app
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
```

