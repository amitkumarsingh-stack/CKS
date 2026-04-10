**Question 1:**

A pod definition file at ``/root/CKS/data-processor.yaml`` has excessive Linux capabilities that pose a security risk.

Secure the pod by:

1. Dropping ALL Linux capabilities by default
2. Adding back only the ``NET_BIND_SERVICE`` capability
3. Ensuring the pod can still bind to port ``8080``
4. Generating a kubesec scan report after fixing the pod

Once done, generate the report again and save it to /root/CKS/capabilities-report.txt

**Step 1 — View the Original File**
```
cat /root/CKS/data-processor.yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
    name: data-processor
  name: data-processor-1
spec:
  containers:
  - image: nginxinc/nginx-unprivileged:alpine
    name: data-processor
    ports:
    - containerPort: 8080
    securityContext:
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
        - NET_ADMIN        # ❌ must remove
        - SYS_ADMIN        # ❌ must remove
        - NET_RAW          # ❌ must remove
        - SYS_PTRACE       # ❌ must remove
```

**Step 2 — Fix the YAML (remove dangerous capabilities)**
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: data-processor
  name: data-processor-1
spec:
  containers:
  - image: nginxinc/nginx-unprivileged:alpine
    name: data-processor
    ports:
    - containerPort: 8080
    securityContext:
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
```

**Step 2 — Recreate the Pod**
```
kubectl delete pod data-processor-1 --force --grace-period=0

kubectl apply -f /root/CKS/data-processor.yaml

kubectl get pod data-processor-1
```

**Step 3 — Generate Kubesec Report**
```
kubesec scan /root/CKS/data-processor.yaml > /root/CKS/capabilities-report.txt

cat /root/CKS/capabilities-report.txt
```
