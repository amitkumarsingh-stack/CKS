**Question 1**: A pod named ``secure-app`` in the ``security-context`` namespace is currently running with excessive privileges. The container is capable of escalating privileges and executing as the root user, which presents a significant security risk.
Please modify the pod configuration to achieve the following objectives:

Prevent privilege escalation
1. Ensure the container operates as a non-root user
2. Set the user ID to ``101`` and the group ID to ``101``
3. Utilize the ``nginxinc/nginx-unprivileged:alpine image``
4. Configure the container to listen on port ``8080``

Ensure that the pod remains capable of serving web content on port ``8080``. You may need to delete and recreate the pod with the appropriate security context.

## Solution

**Step 1 — Switch context and investigate current pod**
```
# Check current pod configuration
kubectl get pod secure-app -n security-context -o yaml
```
You will see the current insecure configuration:
```
spec:
  containers:
  - name: secure-app
    image: nginx:latest          # ← runs as root
    securityContext:
      allowPrivilegeEscalation: true   # ← insecure
      runAsNonRoot: false              # ← insecure
      runAsUser: 0                     # ← root user
```

**Step 2 — Export current pod definition**
```
# Export to file so we can modify it
kubectl get pod secure-app \
  -n security-context \
  -o yaml > /tmp/secure-app.yaml

# Make a backup
cp /tmp/secure-app.yaml /tmp/secure-app-backup.yaml
```

**Step 3 — Edit the pod definition**
```
vi /tmp/secure-app.yaml
```

Remove these auto-generated fields that will cause issues on recreation:
```
# REMOVE these fields:
resourceVersion: "..."
uid: "..."
creationTimestamp: "..."
status: {}
```

Make the following changes
```
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  namespace: security-context
spec:
  securityContext:               # pod-level security context
    runAsNonRoot: true           # fix 2: non-root user
    runAsUser: 101               # fix 3: user ID 101
    runAsGroup: 101              # fix 3: group ID 101
  containers:
  - name: secure-app
    image: nginxinc/nginx-unprivileged:alpine   # fix 4: unprivileged image
    ports:
    - containerPort: 8080        # fix 5: port 8080
    securityContext:             # container-level security context
      allowPrivilegeEscalation: false   # fix 1: prevent escalation
      runAsNonRoot: true                # fix 2: enforce non-root
      runAsUser: 101                    # fix 3: user ID
      runAsGroup: 101                   # fix 3: group ID
```

**Step 4 — Delete the existing pod and recreate**
```
# Delete the current insecure pod
kubectl delete pod secure-app -n security-context

# Wait for deletion
kubectl get pods -n security-context -w
# Wait until pod is fully gone

# Recreate with secure configuration
kubectl apply -f /tmp/secure-app.yaml
```

**Step 5 — Verify pod is running**
```
# Check pod status
kubectl get pod secure-app -n security-context

# Expected:
# NAME         READY   STATUS    RESTARTS   AGE
# secure-app   1/1     Running   0          10s
```

**Step 6 — Verify the container is running as correct user**
```
# Check which user the process is running as
kubectl exec secure-app -n security-context -- id

# Expected:
# uid=101 gid=101 groups=101

# Double check it's NOT root
kubectl exec secure-app -n security-context -- whoami
# Expected: nginx (or similar non-root user, NOT root)
```

**Step 8 — Verify web content is served on port 8080**
```
# Test web server is responding on 8080
kubectl exec secure-app -n security-context -- \
  wget -qO- http://localhost:8080

# Expected: nginx HTML response ✅

# Or using curl
kubectl exec secure-app -n security-context -- \
  curl -s http://localhost:8080
```
-------------------------------------------

