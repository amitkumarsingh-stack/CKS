**Question 1:**

Enable audit logging for the Kubernetes API server to monitor security-relevant events. Please adhere to the following steps:

1. Create an audit policy that logs metadata for all requests.
2. Configure audit log rotation with the following specifications: a maximum size of ``100MB``, retain ``10`` backups, and maintain logs for a maximum of ``30`` days.
3. Set the audit log path to ``/var/log/kubernetes/audit.log``.
4. Mount the necessary directories to facilitate the API server's access to policy files and enable log writing.

Please ensure that the API server continues to function normally after implementing these changes.
In the event that the API server does not recover, a backup is stored at ``/root/kube-apiserver-backup.yaml``. To restore the API server, execute the following commands:

**Step 1: — Create the audit log directory if not present**
```
mkdir -p /var/log/kubernetes
```

**Step 2 — Create the audit policy file**
```
cat <<EOF > /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log metadata for all requests
- level: Metadata
EOF
```
Refer documentation: https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/#audit-policy
**Note:** Please note that you don't have to apply this policy. It has to be present on path ``/etc/kubernetes/audit-policy.yaml``

**Step 3 — Edit the kube-apiserver manifest**
```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add these flags under command:
```
spec:
  containers:
  - command:
    - kube-apiserver
    # ... existing flags ...
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-log-maxsize=100
    - --audit-log-maxbackup=10
    - --audit-log-maxage=30
```
**Note:** All these parameters can be found on the documentation

Add volumeMounts under the container:
```
volumeMounts:
    - mountPath: /etc/kubernetes
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes
      name: audit-log
```

Add volumes at the pod spec level:
```
- hostPath:
    path: /etc/kubernetes
    type: DirectoryOrCreate
  name: audit-policy
- hostPath:
    path: /var/log/kubernetes
    type: DirectoryOrCreate
  name: audit-logs
```

**Step 4: Verify API server is running**
```
kubectl get nodes
```

**Step 5: Check audit log is being created**
```
ls -la /var/log/kubernetes/audit.log
```

**Step 6: Generate some audit events**
```
kubectl get pods
kubectl get nodes
```