**Question 1:**

Configure resource limits for a pod to ensure predictable performance and prevent resource starvation. Create a pod named ``limited-pod`` in the ``resource-demo`` namespace with the following specifications:
Pod Requirements:
* Pod name: ``limited-pod``
* Namespace: ``resource-demo``
* Label: app: ``limited-pod``
* Container name: ``app``
* Image: ``nginx:alpine``

Resource Configuration:
CPU request: ``100m``
CPU limit: ``100m``
Memory request: ``64Mi``
Memory limit: ``64Mi``

Tasks:
1. Create the pod with both resource requests and limits defined
2. Ensure the pod achieves "Guaranteed" QoS class by setting equal requests and limits
3. Verify the pod is running successfully

**Step 1 — Create the pod**
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
  namespace: resource-demo
  labels:
    app: limited-pod
spec:
  containers:
  - name: app
    image: nginx:alpine
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
EOF
```

**Step 2 — Verify pod is running**
```
kubectl get pod limited-pod -n resource-demo
kubectl describe pod limited-pod -n resource-demo | \
  grep -A6 "Limits\|Requests"
```

