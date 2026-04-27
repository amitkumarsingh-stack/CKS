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
---------

**Question 2:**
SSH into the ``cluster2-controlplane`` to address the following tasks:
1. Configure the ``kube-controller-manager`` to use the ``--use-service-account-credentials`` flag.
2. Set the ``--terminated-pod-gc-threshold`` to 50.

The admin kubeconfig file for this cluster is located at:
``/root/controller-config/admin.conf``

Additionally, please utilize this kubeconfig file to delete the cluster role named ``legacy-cluster-role``.

The backup of the original ``kube-controller-manager`` configuration is located at ``/tmp/kube-controller-manager-bak.yaml``. It is essential to ensure that the Controller Manager remains running for the container operations in the upcoming questions.

## Solution
**Step 1 — SSH into cluster2-controlplane**
```
ssh cluster2-controlplane
```

**Step 2 — Edit the kube-controller-manager manifest**
```
vi /etc/kubernetes/manifests/kube-controller-manager.yaml
```

Find the command section and add/update these two flags:
```
spec:
  containers:
  - command:
    - kube-controller-manager
    - --use-service-account-credentials=true   # ← add this
    - --terminated-pod-gc-threshold=50         # ← add this
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    # ... other existing flags remain unchanged
```

**Step 3 — Verify the Controller Manager restarts and is running**
```
# Watch the pod restart (it auto-restarts as a static pod)
watch kubectl get pods -n kube-system | grep controller-manager

# Should show Running within ~30-60 seconds
kubectl get pods -n kube-system | grep controller-manager
```

**Step 4 — Delete the legacy-cluster-role using custom kubeconfig**
```
kubectl delete clusterrole legacy-cluster-role \
  --kubeconfig=/root/controller-config/admin.conf
```

**Question 3:**
Maria is a database administrator who requires varying levels of access across multiple database namespaces.
In the production-db namespace, she must have:
* Full access to all resources (get, list, create, update, delete, patch)

In the ``staging-db`` namespace, her access should be:
* Read-only for pods and services
* No access to secrets or configmaps

In the ``backup-db`` namespace, she should be granted:
* Get and list permissions for persistentvolumeclaims only
* No access to any other resources

The current RBAC configuration is overprivileged. Please update the permissions to align with the principle of least privilege while retaining the same resource names.

## Solution
**Step 1 — Check existing RBAC resource names (to retain them)**
```
kubectl get role,rolebinding -n production-db
kubectl get role,rolebinding -n staging-db
kubectl get role,rolebinding -n backup-db
```

**Step 2 — Fix production-db Role (full access)**
```
kubectl get role -n production-db -o yaml  # note existing name

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: maria-production-role      # use existing name from Step 1
  namespace: production-db
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "create", "update", "delete", "patch"]
EOF
```

**Step 3 — Fix staging-db Role (read-only pods and services only)**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: maria-staging-role         # use existing name from Step 1
  namespace: staging-db
rules:
- apiGroups: [""]
  resources: ["pods", "services"]  # only pods and services
  verbs: ["get", "list"]           # read-only
EOF
```

**Step 4 — Fix backup-db Role (get/list PVCs only)**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: maria-backup-role          # use existing name from Step 1
  namespace: backup-db
rules:
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list"]
EOF
```

**Step 5 — Verify RoleBindings still point to Maria**
```
# Check each namespace binding still references Maria's user/SA
kubectl get rolebinding -n production-db -o yaml | grep -A3 "subjects"
kubectl get rolebinding -n staging-db -o yaml | grep -A3 "subjects"
kubectl get rolebinding -n backup-db -o yaml | grep -A3 "subjects"
```

**Step 6 — Verify permissions with auth can-i**
```
# production-db — all should be yes
kubectl auth can-i create pods -n production-db --as=maria
kubectl auth can-i delete pods -n production-db --as=maria
kubectl auth can-i get secrets -n production-db --as=maria

# staging-db — pods/services yes, secrets/configmaps no
kubectl auth can-i get pods -n staging-db --as=maria
kubectl auth can-i get services -n staging-db --as=maria
kubectl auth can-i get secrets -n staging-db --as=maria      # should be NO
kubectl auth can-i get configmaps -n staging-db --as=maria   # should be NO
kubectl auth can-i create pods -n staging-db --as=maria      # should be NO

# backup-db — only PVC get/list
kubectl auth can-i get persistentvolumeclaims -n backup-db --as=maria
kubectl auth can-i list persistentvolumeclaims -n backup-db --as=maria
kubectl auth can-i get pods -n backup-db --as=maria          # should be NO
kubectl auth can-i delete persistentvolumeclaims -n backup-db --as=maria # NO
```
--------------------------
**Question 4:**

A security team needs ``cross-namespace monitoring`` capabilities with restricted permissions. Create RBAC resources that allow a ServiceAccount to read pods across multiple namespaces while following the principle of least privilege.

Requirements:
1. Create a ServiceAccount named ``cluster-monitor`` in the monitoring-ns namespace
2. Create a ClusterRole named ``pod-reader-clusterrole`` that allows ``get``, ``list``, and ``watch`` operations on pods
3. Grant the ServiceAccount access only in the ``monitoring-ns`` and ``apparmor-demo`` namespaces
4. Label the ``monitoring-ns and apparmor-demo`` namespaces with ``monitoring: enabled``
5. Create a RoleBinding named ``cross-namespace-monitor`` in each of the allowed namespaces (monitoring-ns and apparmor-demo) that references the ClusterRole

Note: Use namespace-scoped RoleBindings (not ClusterRoleBinding) to restrict access to only the specified namespaces.

**Step 1 — Create the ServiceAccount**
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-monitor
  namespace: monitoring-ns
EOF
```

**Step 2 — Create the ClusterRole**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF
```

**Step 3 — Label both namespaces**
```
kubectl label namespace monitoring-ns monitoring=enabled --overwrite
kubectl label namespace apparmor-demo monitoring=enabled --overwrite

# Verify
kubectl get namespace monitoring-ns apparmor-demo --show-labels
```

**Step 4 — Create RoleBinding in monitoring-ns**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cross-namespace-monitor
  namespace: monitoring-ns
subjects:
- kind: ServiceAccount
  name: cluster-monitor
  namespace: monitoring-ns
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: pod-reader-clusterrole
EOF
```

**Step 5 — Create RoleBinding in apparmor-demo**
```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cross-namespace-monitor
  namespace: apparmor-demo
subjects:
- kind: ServiceAccount
  name: cluster-monitor
  namespace: monitoring-ns
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: pod-reader-clusterrole
EOF
```

**Step 6 — Test permissions with auth can-i**
```
# Should be YES in monitoring-ns
kubectl auth can-i get pods -n monitoring-ns \
  --as=system:serviceaccount:monitoring-ns:cluster-monitor

kubectl auth can-i list pods -n monitoring-ns \
  --as=system:serviceaccount:monitoring-ns:cluster-monitor

kubectl auth can-i watch pods -n monitoring-ns \
  --as=system:serviceaccount:monitoring-ns:cluster-monitor

# Should be YES in apparmor-demo
kubectl auth can-i get pods -n apparmor-demo \
  --as=system:serviceaccount:monitoring-ns:cluster-monitor

# Should be NO in other namespaces
kubectl auth can-i get pods -n default \
  --as=system:serviceaccount:monitoring-ns:cluster-monitor

kubectl auth can-i get pods -n kube-system \
  --as=system:serviceaccount:monitoring-ns:cluster-monitor

# Should be NO for other resources
kubectl auth can-i get secrets -n monitoring-ns \
  --as=system:serviceaccount:monitoring-ns:cluster-monitor
```

