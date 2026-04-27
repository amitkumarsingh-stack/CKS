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
--------------------------
**Question 2:**
A security audit identified that containers in the ``secure-runtime`` namespace could be vulnerable to process debugging attacks. There is already a pod named ``secure-app running`` in this namespace.

Your tasks:

1. Create a custom seccomp profile at ``/var/lib/kubelet/seccomp/profiles/block-debug.json`` with:
* defaultAction: ``SCMP_ACT_ALLOW`` (allow all syscalls by default)
* Block specific syscalls: ``ptrace and process_vm_readv``
* Use ``SCMP_ACT_ERRNO`` action for the blocked syscalls
2. Recreate the ``secure-app`` pod to use this custom seccomp profile:
* seccompProfile ``type: Localhost``
* localhostProfile: ``profiles/block-debug.json``
3. Ensure the pod remains functional and can still serve nginx content

Note: Since you cannot modify a running pod's security context, you must delete and recreate the pod.

**Step 1: Create the custom seccomp profile**
```
sudo mkdir -p /var/lib/kubelet/seccomp/profiles

sudo tee /var/lib/kubelet/seccomp/profiles/block-debug.json <<EOF
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "names": ["ptrace", "process_vm_readv"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
EOF
```
Refer document: https://kubernetes.io/docs/tutorials/security/seccomp/#download-profiles

**Step 2: Recreate the secure-app pod with the seccomp profile**
```
kubectl get pod secure-app -n secure-runtime -o yaml > secure-app.yaml
```

**Step 3: Edit (or create) the pod manifest and add the seccomp profile under securityContext:**
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
      localhostProfile: profiles/block-debug.json
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

**Step 4: Verify the pod is functional**
```
kubectl get pods -n secure-runtime
```

