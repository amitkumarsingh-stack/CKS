**Question 1:**
In the namespace ``code``, create a TLS secret ``code-secret`` with the following certificate and key provided:
* cert: ``/root/custom-cert.crt``
* key: ``/root/custom-key.key``

Attach that secret as a volume named ``secret-volume`` in the deployment ``code-server`` and mount it to the container at path ``/etc/code/tls``.

## Solution

**Step 1 — Create the TLS secret**
```
kubectl create secret tls code-secret \
  --cert=/root/custom-cert.crt \
  --key=/root/custom-key.key \
  -n code
```
Refer document: https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_secret_tls/#examples


**Step 2 — Verify the secret was created**
```
kubectl get secret code-secret -n code
kubectl describe secret code-secret -n code
```

**Step 3 — Patch the deployment to add the volume and volumeMount**
```
kubectl edit deployment code-server -n code
```
Add the volume under ``spec.template.spec.volumes``:
```
volumes:
- name: secret-volume
  secret:
    secretName: code-secret
```
Add the volumeMount under ``spec.template.spec.containers[0].volumeMounts``:
```
volumeMounts:
- name: secret-volume
  mountPath: /etc/code/tls
  readOnly: true
```

Complete file
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: code-server
  namespace: code
spec:
  replicas: 1
  selector:
    matchLabels:
      app: code-server
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: code-server
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: code-secret
      containers:
      - name: busybox-container
        image: busybox
        command:
        - sleep
        - "3600"
        volumeMounts:
        - name: secret-volume
          mountPath: /etc/code/tls
          readOnly: true
```

**Step 5 — Verify pods are running with the mount**
```
kubectl get pods -n code
kubectl describe pod -n code -l app=code-server | grep -A5 "Mounts\|Volumes"
```

**Step 6 — Confirm the TLS files are mounted inside the container**
```
kubectl exec -n code deployment/code-server -- ls /etc/code/tls
# Should show: tls.crt  tls.key
```

