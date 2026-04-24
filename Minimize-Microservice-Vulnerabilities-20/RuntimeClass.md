**Question 1:**

Create a RuntimeClass and configure a deployment to use it for workload isolation.
Tasks:
1. Create a RuntimeClass named ``secured`` using the ``runc`` handler
2. Create a deployment named ``isolated-app`` in the ``runtime-demo`` namespace that uses this RuntimeClass

RuntimeClass Requirements:
* Name: ``secured``
* Handler: ``runc``

Deployment Requirements:
* Deployment name: ``isolated-app``
* Namespace: ``runtime-demo``
* Replicas: 1
* Container name: ``app``
* Image: ``nginx:alpine``
* Container port: ``80``
* RuntimeClass: ``secured``

# Solution

**Step 1 — Create the RuntimeClass**
```
cat <<EOF | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: secured
handler: runc
EOF
```
Refer document https://kubernetes.io/docs/concepts/containers/runtime-class/#2-create-the-corresponding-runtimeclass-resources

**Step 2 — Create the deployment**
```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: isolated-app
  namespace: runtime-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: isolated-app
  template:
    metadata:
      labels:
        app: isolated-app
    spec:
      runtimeClassName: secured
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF
```
Refer document: https://kubernetes.io/docs/concepts/containers/runtime-class/#usage

**Step 3 — Verify the deployment and pod**
```
kubectl get deployment isolated-app -n runtime-demo
kubectl get pods -n runtime-demo
kubectl rollout status deployment/isolated-app -n runtime-demo
```
