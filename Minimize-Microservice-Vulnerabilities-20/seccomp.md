**Question 1:**
A security audit identified that containers in the ``secure-runtime`` namespace could be vulnerable to process debugging attacks. There is already a pod named ``secure-app`` running in this namespace.
Your task is to:

1. Create a custom seccomp profile that:
    * Blocks the ``ptrace`` and ``process_vm_readv`` syscalls to prevent process debugging
    * Allows all other syscalls to maintain application functionality (use ``SCMP_ACT_ALLOW`` as the default action)
    * Uses ``SCMP_ACT_ERRNO`` action for the blocked syscalls

2. Save the profile as ``/var/lib/kubelet/seccomp/profiles/block-debug.json``
3. Configure the existing secure-app pod to use this custom seccomp profile
4. Ensure the pod remains functional and can still serve nginx content after applying the seccomp profile

**Note:** You may need to restart or recreate the pod to apply the seccomp profile changes.

## Solution

**Step 1 — Create the Seccomp Profile**
```
# Create the profiles directory if it doesn't exist
mkdir -p /var/lib/kubelet/seccomp/profiles
```

```
cat > /var/lib/kubelet/seccomp/profiles/block-debug.json << 'EOF'
{
  "defaultAction": "SCMP_ACT_ALLOW",    # Default action
  "syscalls": [
    {
      "names": [
        "ptrace",                       # Prevent Process debugging
        "process_vm_readv"              # Prevent Process debugging
      ],
      "action": "SCMP_ACT_ERRNO"        # Blocked syscalls
    }
  ]
}
EOF
```
Notes: Refer document https://kubernetes.io/docs/tutorials/security/seccomp/#download-profiles

**Step 2 — Check the Existing Pod & Edit the Pod**
```
kubectl get pod secure-app -n secure-runtime -o yaml

# Edit the file
vi /tmp/secure-app.yaml

```

Add the seccomp profile under securityContext at the pod level (not container level):
```
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: secure-runtime
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/block-debug.json   # relative to /var/lib/kubelet/seccomp/
  containers:
  - name: secure-app
    image: nginx
    ports:
    - containerPort: 80
```
Note: By default Seccomp profiles are stored in ``/var/lib/kubelet/seccomp/profiles``. Refer document https://kubernetes.io/docs/tutorials/security/seccomp/#create-a-pod-with-a-seccomp-profile-for-syscall-auditing on how to add a seccomp profile to Pod.

**Step 3 — Recreate the Pod**
```
# Delete existing pod
kubectl delete pod secure-app -n secure-runtime --force --grace-period=0

# Apply updated pod
kubectl apply -f /tmp/secure-app.yaml

# Verify pod is running
kubectl get pod secure-app -n secure-runtime
```

**Step 4 — Verify Pod is Functional**
```
# Check pod is Running
kubectl get pod secure-app -n secure-runtime

# Check nginx is serving content
kubectl exec -n secure-runtime secure-app -- curl -s localhost:80
```



