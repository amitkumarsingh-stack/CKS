**Question 1:** Use context: ``kubectl config use-context workload-prod``
Some containers need to run more secure and restricted. There is an existing AppArmor profile located at ``/opt/course/9/profile`` for this.

Install the AppArmor profile on Node ``cluster1-node1``. Connect using ``ssh cluster1-node1``.
Add label ``security=apparmor`` to the Node
Create a Deployment named apparmor in Namespace default with:

* One replica of image nginx:1.19.2
* NodeSelector for security=apparmor
* Single container named c1 with the AppArmor profile enabled

The Pod might not run properly with the profile enabled. Write the logs of the Pod into ``/opt/course/9/logs`` so another team can work on getting the application running.

### Solution
**Step 1 — Switch context**
```
kubectl config use-context workload-prod
```

**Step 2 — Read the profile to get its name**
```
cat /opt/course/9/profile
```

You'll see something like:
```
#include <tunables/global>

profile very-secure flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
```
⚠️ The profile name is very-secure (defined inside the file) — NOT the filename profile. You'll need this exact name in the Deployment spec.

**Step 3 — Copy the profile to the node**
```
# From the jumpbox/exam machine
scp /opt/course/9/profile cluster1-node1:/tmp/profile
```

**Step 4 — SSH to node and install the profile**
```
ssh cluster1-node1
sudo -i

# Copy to apparmor profiles directory
cp /tmp/profile /etc/apparmor.d/very-secure

# Load the profile
apparmor_parser /etc/apparmor.d/very-secure

# Verify it's loaded
aa-status | grep very-secure
# Expected: very-secure (enforce)

exit  # leave node
```
Refer document: https://kubernetes.io/docs/tutorials/security/apparmor/

**Step 5 — Add label to the Node**
```
kubectl label node cluster1-node1 security=apparmor

# Verify
kubectl get node cluster1-node1 --show-labels | grep security
# Expected: security=apparmor
```

**Step 6 — Create the Deployment**
```
cat > apparmor-deploy.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apparmor
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: apparmor
  template:
    metadata:
      labels:
        app: apparmor
    spec:
      nodeSelector:
        security: apparmor          # ensures pod lands on labelled node
      securityContext:
        appArmorProfile:
          type: Localhost
          localhostProfile: very-secure   # exact profile name from file
      containers:
      - name: c1                    # must be named c1
        image: nginx:1.19.2
EOF

kubectl apply -f apparmor-deploy.yaml
```

**Step 7 — Check the Pod status**
```
kubectl get pods -n default | grep apparmor

# Pod may show Error or CrashLoopBackOff
# This is EXPECTED — the profile denies all file writes
# nginx needs to write files to start, so it will fail
```

**Step 8 — Verify AppArmor profile is applied on the container**
```
# Get the pod name
kubectl get pods -n default | grep apparmor

# SSH to node and verify with crictl
ssh cluster1-node1

# Find the pod
crictl pods | grep apparmor

# Find the container (including stopped)
crictl ps -a | grep <pod-id>

# Inspect to confirm profile
crictl inspect <container-id> | grep -i apparmor
# Expected: "apparmor_profile": "localhost/very-secure"

exit
```
Note: You can either use step 7 or 8 to verify

**Step 9 — Write Pod logs to required path**
```
# Get the pod name
POD=$(kubectl get pods -n default -l app=apparmor \
  -o jsonpath='{.items[0].metadata.name}')

echo $POD  # confirm pod name

# Write logs to required path
kubectl logs $POD -n default > /opt/course/9/logs

# Verify logs file has content
cat /opt/course/9/logs
```

You'll see errors like:
```
/docker-entrypoint.sh: 13: cannot create /dev/null: Permission denied
mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
nginx: [emerg] mkdir() failed (13: Permission denied)
```
This confirms the AppArmor profile is working correctly — it's denying all file writes which nginx needs to start.

⚠️ Three most common mistakes on this question:
1. Using the filename instead of the profile name in the Deployment spec
2. Forgetting to run apparmor_parser after copying the file to the node
3. Forgetting the nodeSelector — without it the pod might land on a node without the profile and fail to schedule or use wrong profile
----------

**Question 2:**
A deployment named frontend in the ``apparmor-demo`` namespace requires additional security isolation. An AppArmor profile has been created at ``/etc/apparmor.d/containers/restricted-frontend`` with the following security restrictions:
1. Prevents the container from writing to ``/etc/``, ``/bin/``, ``/sbin/``, ``/usr/bin/``, and ``/usr/sbin/`` directories
2. Allows network access only on ``TCP`` and ``UDP`` protocols
3. Blocks raw socket access
4. Allows write access only to ``/tmp/`` directory
5. Prevents capability escalation

Your tasks:
1. Load the AppArmor profile using apparmor_parser
2. Configure the deployment to use the AppArmor profile using the securityContext field (not annotations)

Configuration Details:
* Container name: ``web``
* AppArmor profile type: ``Localhost``
* ``LocalhostProfile: restricted-frontend`` (just the profile name, not localhost/restricted-frontend)

Use the modern securityContext approach instead of deprecated annotations.

## Solution

**Step 1 — Verify the AppArmor profile exists**
```
cat /etc/apparmor.d/containers/restricted-frontend
```

**Step 2 — Load the AppArmor profile**
```
apparmor_parser -q /etc/apparmor.d/containers/restricted-frontend
```

**Step 3 — Verify profile is loaded**
```
aa-status | grep restricted-frontend
```

**Step 4 — Check the existing deployment**
```
kubectl get deployment frontend -n apparmor-demo -o yaml
```

**Step 5 — Patch the deployment with AppArmor securityContext**
```
kubectl patch deployment frontend -n apparmor-demo \
  --type=strategic -p='
spec:
  template:
    spec:
      containers:
      - name: web
        securityContext:
          appArmorProfile:
            type: Localhost
            localhostProfile: restricted-frontend
```

**Step 6 — Test the restrictions work**
```
# Writing to /etc/ should be BLOCKED
kubectl exec -n apparmor-demo deployment/frontend -- \
  touch /etc/test.txt
# Expected: Permission denied
```
