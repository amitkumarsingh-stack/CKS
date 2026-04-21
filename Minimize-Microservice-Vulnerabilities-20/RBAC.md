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
-----------------
**Question 2:**

1. Create a TLS secret named ``app-tls`` using:
  * Certificate: ``/root/app-cert.crt``
  * Private key: ``/root/app-key.key``

2.  Mount the secret securely in the deployment secure-app:
  * Volume name: ``tls-secret-volume``
  * Mount path: ``/etc/app/tls``
  * Set file permissions to ``0400`` (read-only by owner)

3. Create a ServiceAccount named secure-sa for the deployment
4, Create RBAC resources to:
  * Create a Role named ``secret-access`` that allows only get and list operations on secrets in the vault namespace
  * Create a RoleBinding named ``secret-access-binding`` to bind the Role to the ServiceAccount
  * Add the label ``app: secure-app`` to both the Role and RoleBinding

5. Update the deployment to use the ServiceAccount and security context:
  * Run as ``non-root user`` (UID 1001)
  * Set ``allowPrivilegeEscalation: false``

## Solution

**Step 1: Create TLS secrets**
```
   kubectl create secret tls app-tls \
     --cert=/root/app-cert.crt \
     --key=/root/app-key.key \
     -n vault
```

**Step 2: Create ServiceAccount**
```
kubectl create serviceaccount secure-sa -n vault
```

**Step 3: Create Role**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-access
  namespace: vault
  labels:
    app: secure-app
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
EOF
```

**Step 4: Create Rolebindings**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-access-binding
  namespace: vault
  labels:
    app: secure-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-access
subjects:
- kind: ServiceAccount
  name: secure-sa
  namespace: vault
EOF
```

**Step 5: Patch the deployment**
```
 spec:
   template:
     spec:
       serviceAccountName: secure-sa
       securityContext:
         runAsNonRoot: true
         runAsUser: 1001
         fsGroup: 1001
       containers:
       - name: app
         securityContext:
           runAsNonRoot: true
           runAsUser: 1001
           allowPrivilegeEscalation: false
           capabilities:
             drop:
               - ALL
         volumeMounts:
         - name: tls-secret-volume
           mountPath: /etc/app/tls
           readOnly: true
       volumes:
       - name: tls-secret-volume
         secret:
           secretName: app-tls
           defaultMode: 0400
```

**Step 6: Verification**
Verify RBAC permissions:
 # Test if ServiceAccount can access secrets in vault namespace
 ```
 kubectl auth can-i get secrets --as=system:serviceaccount:vault:secure-sa -n vault
```
 # Check what the ServiceAccount can do
```
 kubectl auth can-i --list --as=system:serviceaccount:vault:secure-sa -n vault
```


