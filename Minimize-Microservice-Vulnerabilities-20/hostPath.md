**Question 1:**

Create a pod named ``volume-app`` in the ``volume-security`` namespace that uses a hostPath volume with proper security measures to restrict volume access and prevent potential host system compromise.
Security Requirements:
* Mount the hostPath volume as read-only to prevent modifications to the host filesystem
* Use a specific directory ``/var/log/app`` on the host
* Set the volume mount to ``read-only`` mode
* Ensure the pod runs as ``non-root`` user (UID 1000)

Pod Specifications:
* Pod name: ``volume-app``
* Namespace: ``volume-security``
* Label: ``app: volume-app``
* Container name: ``logger``
* Image: ``busybox``
* Command: ``['sh', '-c', 'tail -f /dev/null']``
* Volume mount path: ``/app/logs``

**Step 1 — Create the pod**
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: volume-app
  namespace: volume-security
  labels:
    app: volume-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log/app
      type: DirectoryOrCreate
  containers:
  - name: logger
    image: busybox
    command: ['sh', '-c', 'tail -f /dev/null']
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
    volumeMounts:
    - name: host-logs
      mountPath: /app/logs
      readOnly: true
EOF
```

**Step 2 — Verify pod is running**
```
kubectl get pod volume-app -n volume-security
```

**Step 3 — Verify read-only mount**
```
# Try to write to the mounted volume — should FAIL
kubectl exec -n volume-security volume-app -- \
  touch /app/logs/test.txt
# Expected: Read-only file system error ✅
```

