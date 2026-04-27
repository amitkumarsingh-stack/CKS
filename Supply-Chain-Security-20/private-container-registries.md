**Question 1:**
A private container registry requires authentication for pulling images. Configure the necessary resources to allow pods in the ``private-registry-ns`` namespace to pull images from a private registry.

Tasks:
1. Create a Secret of type ``docker-registry`` named ``my-registry-key`` in the ``private-registry-ns`` namespace
2. Use the following credentials:
* Username: ``myuser``
* Password: ``mypassword``
* Email: ``myuser@example.com``
* Registry server: ``registry.example.com``

Create a Pod named ``private-app`` with label ``app: private-app`` that references the image pull secret
* Use the ``busybox`` image with command ``["sleep", "3600"]`` for testing purposes

**Step 1 — Create the docker-registry secret**
```
kubectl create secret docker-registry my-registry-key \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com \
  -n private-registry-ns
```
Refer document: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-by-providing-credentials-on-the-command-line

**Step 2 — Create the Pod**
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: private-app
  namespace: private-registry-ns
  labels:
    app: private-app
spec:
  imagePullSecrets:
  - name: my-registry-key
  containers:
  - name: private-app
    image: busybox
    command: ["sleep", "3600"]
EOF
```
Refer document: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret

**Step 3 — Verify pod is running**
```
kubectl get pod private-app -n private-registry-ns
kubectl describe pod private-app -n private-registry-ns | \
  grep -E "Image|Pull|Secret"
```