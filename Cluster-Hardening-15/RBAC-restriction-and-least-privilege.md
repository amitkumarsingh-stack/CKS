**Question 1:** A service account named ``ci-bot`` in the ``ci-cd`` namespace has been granted excessive permissions that could potentially enable it to create ``cluster-admin`` bindings.
Your task is to modify the RBAC configuration as follows:

1. Prevent the ``ci-bot`` service account from binding to any role that has ``"admin"`` or ``"cluster-admin"`` in its name.
2. Ensure that the service account retains its current permissions within the ``ci-cd`` namespace.
3. Apply this restriction broadly across the entire cluster.

To achieve this, it may be necessary to delete and recreate the existing role binding with the appropriate restrictions.

## Solution

**Step 1 — Investigate current RBAC setup**
```
# Check the service account
kubectl get serviceaccount ci-bot -n ci-cd

# Find ALL rolebindings for ci-bot in ci-cd namespace
kubectl get rolebindings -n ci-cd \
  -o wide | grep ci-bot

# Find ALL clusterrolebindings for ci-bot
kubectl get clusterrolebindings \
  -o wide | grep ci-bot

# Describe each binding found
kubectl describe rolebinding <name> -n ci-cd
kubectl describe clusterrolebinding <name>
```

**Step 2 — Document current legitimate permissions**

Before making changes, note what permissions ``ci-bot`` legitimately needs:
```
# Check what the current role/clusterrole allows
kubectl describe clusterrole <role-name>
# OR
kubectl describe role <role-name> -n ci-cd

# Check all permissions ci-bot currently has
kubectl auth can-i --list \
  --as=system:serviceaccount:ci-cd:ci-bot \
  -n ci-cd
```

There are two way you can solve this question
* Method 1: Either you can delete and recreate the role
* Method 2: You can edit the existing Role and remove the overly permission 

### Method 1

**Step 3 - Remove the overly permissive ClusterRole**
```
kubectl delete clusterrole ci-bot-role
```

**Step 4 - Create a restricted ClusterRole that omits binding creation permissions:**
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ci-bot-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "delete"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterrolebindings", "rolebindings"]
  verbs: ["get", "list", "watch"]
  # Note: This role does not include create, update, or patch permissions for bindings
```
Refer document: https://kubernetes.io/docs/reference/access-authn-authz/rbac/

**Step 5 - Apply the restricted role**
```
kubectl apply -f - <<EOF
[above yaml content]
EOF
```

### Method 2: Edit the existing cluster role

Current Cluster Role
```
k edit clusterrole -n ci-cd ci-bot-role

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRole","metadata":{"annotations":{},"name":"ci-bot-role"},"rules":[{"apiGroups":["rbac.authorization.k8s.io"],"resources":["clusterrolebindings","rolebindings"],"verbs":["create","update","patch"]},{"apiGroups":[""],"resources":["pods","services","configmaps"],"verbs":["get","list","watch","create","update","delete"]}]}
  creationTimestamp: "2026-04-07T07:18:12Z"
  name: ci-bot-role
  resourceVersion: "8087"
  uid: 8a644ace-da94-4875-8186-ccdd7f81926c
rules:
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  - rolebindings
  verbs:
  - create
  - update
  - patch
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - configmaps
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - delete
```

Modify the permission for Clusterrolebindings as below
```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRole","metadata":{"annotations":{},"name":"ci-bot-role"},"rules":[{"apiGroups":["rbac.authorization.k8s.io"],"resources":["clusterrolebindings","rolebindings"],"verbs":["create","update","patch"]},{"apiGroups":[""],"resources":["pods","services","configmaps"],"verbs":["get","list","watch","create","update","delete"]}]}
  creationTimestamp: "2026-04-07T07:18:12Z"
  name: ci-bot-role
  resourceVersion: "8087"
  uid: 8a644ace-da94-4875-8186-ccdd7f81926c
rules:
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  - rolebindings
  verbs:
  - get             # Replaced with create
  - list            # Replaced with update
  - watch           # Replaced with patch
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - configmaps
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - delete
```
**Step 6 - Verify standard operations (expected result: "yes"):**
```
kubectl auth can-i get pods --as=system:serviceaccount:ci-cd:ci-bot
kubectl auth can-i create deployments --as=system:serviceaccount:ci-cd:ci-bot
```

**Step 7 - Verify binding restrictions (expected result: "no"):**
```
kubectl auth can-i create clusterrolebindings --as=system:serviceaccount:ci-cd:ci-bot
kubectl auth can-i create rolebindings --as=system:serviceaccount:ci-cd:ci-bot
```
