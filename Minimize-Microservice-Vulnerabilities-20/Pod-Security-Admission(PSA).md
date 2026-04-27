**Question 1:**

The ``financial-apps`` namespace contains sensitive financial applications that require strict security controls.
Configure Pod Security Admission to:

1. Enforce the restricted policy level on the ``financial-apps`` namespace
2. Use the latest version of the ``Pod Security Standards``
3. Add a warning level for the baseline policy to alert on less strict pods
4. Label the namespace appropriately for the PSA controller

Verify that the configuration is working by attempting to create a privileged pod and observing the rejection.

## Solution

**Setp 1: Inspect the namespace**
```
kubectl get ns | grep financial-apps
```

**Step 2: Add labels to namespace**
```
kubectl label namespace financial-apps \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest
```
Refer document https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/

**Step 3: Verify by Creating a Privileged Pod (should be REJECTED)**
```
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
  namespace: financial-apps
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF
```
Expected rejection output:
```
Error from server (Forbidden): pods "privileged-test" is forbidden:
violates PodSecurity "restricted:latest":
privileged (container "test" must not set securityContext.privileged=true)
```

**PSA Policy Levels — Quick Reference**
| Label Key | Values | Effect |
|-----------|--------|--------|
|``enforce``| ``privileged`` / ``baseline`` / ``restricted`` | Blocks non-compliant pods|
|``warn``| ``privileged`` / ``baseline`` / ``restricted`` | Shows warning but allows pod|
|``audit``| ``privileged`` / ``baseline`` / ``restricted``| Logs to audit log only|

**Policy Levels Explained**
| Level     | Description |
|-----------|-------------|
|``privileged``| No restrictions — fully open|
|``baseline``| Prevents known privilege escalations (no hostNetwork, no privileged)|
|``restricted``| Hardened — requires non-root, dropped caps, seccomp, no privilege escalation|
----------------

**Question 2:**

We want to deploy a PodSecurity admission controller to enforce security standards across the cluster.
Tasks:

1. Fix the error in ``/etc/kubernetes/pki/podsecurity_configuration.yaml`` which will be used by the PodSecurity admission controller.
2. Ensure that the restricted policy level is enforced across all namespaces by default, with audit and warn modes for baseline violations.
3. Enable the plugin on the API server by:
  * Adding PodSecurity to the ``--enable-admission-plugins`` flag (required when using a custom config file)
  * Setting ``--admission-control-config-file`` to point to the configuration file

The PodSecurity admission controller should reject any pods that don't meet the restricted policy standards.

A copy of the kube-apiserver.yaml is available in ``/tmp/kube-apiserver-backup.yaml`` so you can revert if the configuration goes wrong. Ensure that the kube-apiserver is working correctly, as it will be required for grading the exam.

## Solution
** Step 1: Check and fix the PodSecurity configuration file
```
vi /etc/kubernetes/pki/podsecurity_configuration.yaml
```

Corrected File
```
cat <<EOF > /etc/kubernetes/pki/podsecurity_configuration.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "restricted"
      enforce-version: "latest"
      audit: "baseline"
      audit-version: "latest"
      warn: "baseline"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces:
        - kube-system
        - kube-public
        - kube-node-lease
EOF
```

**Step 2 — Edit the kube-apiserver manifest**
```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add these two flags under the command section:
```
spec:
  containers:
  - command:
    - kube-apiserver
    - --enable-admission-plugins=NodeRestriction,PodSecurity   # ← add PodSecurity
    - --admission-control-config-file=/etc/kubernetes/pki/podsecurity_configuration.yaml  # ← add this
    # ... other existing flags
```
**Note:**  If ``--enable-admission-plugins`` already exists, just append ,``PodSecurity`` to it. Don't create a duplicate flag.

**Step 3: — Wait for API server to restart**
```
watch kubectl get pods -n kube-system | grep apiserver
```

**Step 4 — Verify PodSecurity is enforcing**
```
# Try to create a privileged pod — should be REJECTED
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
  namespace: default
spec:
  containers:
  - name: test
    image: busybox
    securityContext:
      privileged: true
EOF
# Expected: Error from server (Forbidden): violates PodSecurity "restricted:latest"

# Check audit mode is working
kubectl get events -n default | grep PodSecurity
```
-----------
**Question 3:**

Create Security Context Constraints using Pod Security Standards for the ``scc-demo`` namespace. Implement the following restrictions:
1. Prevent privileged containers
2. Require runAsNonRoot
3. Block host namespaces
4. Require seccomp profiles
5. Restrict capabilities

Apply these restrictions using Pod Security Standards labels with the following requirements:
* Enforce level: ``restricted``
* Enforce version: ``latest``

The restricted profile will enforce most security requirements automatically.

**Step 1 — Apply Pod Security Standards labels**
```
kubectl label namespace scc-demo \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest \
  --overwrite
```

**Step 2 — Test restrictions are enforced**
Test 1 — Privileged container (should be BLOCKED)
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
  namespace: scc-demo
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF
# Expected: Error — violates PodSecurity restricted
```

Test 2 — Compliant pod (should be ALLOWED)
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-compliant
  namespace: scc-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: test
    image: nginxinc/nginx-unprivileged:alpine
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
EOF
# Expected: pod created successfully
```
