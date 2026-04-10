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