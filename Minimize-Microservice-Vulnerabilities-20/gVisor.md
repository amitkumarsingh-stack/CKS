**Question 1:** Use context: ``kubectl config use-context workload-prod``
Team purple wants to run some of their workloads more secure. Worker node ``cluster1-node2`` has container engine containerd already installed and it's configured to support the ``runsc/gvisor`` runtime.

1. Create a RuntimeClass named gvisor with handler runsc.
1. Create a Pod that uses the ``RuntimeClass``. The Pod should be in Namespace ``team-purple``, named ``gvisor-test`` and of image ``nginx:1.19.2``. Make sure the Pod runs on ``cluster1-node2``.
3. Write the dmesg output of the successfully started Pod into ``/opt/course/10/gvisor-test-dmesg``.

## Solution

``gVisor`` adds an extra security layer by running containers in a sandboxed environment with its own kernel implementation.

**Step 1 — Switch context**
```
kubectl config use-context workload-prod
```

**Step 2 — Verify gVisor is available on cluster1-node2**
```
ssh cluster1-node2
sudo -i

# Check runsc is installed
runsc --version

# Check containerd is configured for runsc
cat /etc/containerd/config.toml | grep -A 5 "runsc"
# Should show runsc as a runtime handler

exit
```

**Step 3 — Create the RuntimeClass**
```
cat > gvisor-runtimeclass.yaml << 'EOF'
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor              # name used in Pod spec
handler: runsc              # maps to the actual runtime on the node
EOF

kubectl apply -f gvisor-runtimeclass.yaml

# Verify
kubectl get runtimeclass
# Expected:
# NAME     HANDLER   AGE
# gvisor   runsc     5s
```
Refer document: https://kubernetes.io/docs/concepts/containers/runtime-class/

**Step 4 — Create the Pod using RuntimeClass**
```
cat > gvisor-test-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: gvisor-test
  namespace: team-purple
spec:
  runtimeClassName: gvisor        # references the RuntimeClass
  nodeSelector:
    kubernetes.io/hostname: cluster1-node2   # ensure pod runs on node2
  containers:
  - name: gvisor-test
    image: nginx:1.19.2
EOF

kubectl apply -f gvisor-test-pod.yaml
```

OR

```
cat > gvisor-test-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: gvisor-test
  namespace: team-purple
spec:
  runtimeClassName: gvisor        # references the RuntimeClass
  nodeName: cluster1-node2
  containers:
  - name: gvisor-test
    image: nginx:1.19.2
EOF

kubectl apply -f gvisor-test-pod.yaml
```

**Step 5 — Verify Pod is running on correct node**
```
# Wait for pod to be running
kubectl get pod gvisor-test -n team-purple -w

# Confirm it's on cluster1-node2
kubectl get pod gvisor-test -n team-purple -o wide
# NODE column should show cluster1-node2

# Confirm RuntimeClass is being used
kubectl describe pod gvisor-test -n team-purple | grep -i runtime
# Expected: Runtime Class Name: gvisor
```

**Step 6 — Get dmesg output from the Pod**
```
# Run dmesg inside the pod
kubectl exec gvisor-test -n team-purple -- dmesg
```

You'll see output like:
```
[    0.000000] Starting gVisor...
[    0.000000] ** This is not a real kernel **
```
This confirms gVisor is working — the output is from gVisor's own kernel implementation, not the host.
```
# Save the output to required path
kubectl exec gvisor-test -n team-purple -- dmesg \
  > /opt/course/10/gvisor-test-dmesg

# Verify the file has content
cat /opt/course/10/gvisor-test-dmesg
```

⚠️ Two most common mistakes on this question:

1. Confusing name and handler in RuntimeClass — name: gvisor is what you reference in the Pod, handler: runsc is the actual binary on the node
2. Forgetting the nodeSelector — gVisor is only installed on cluster1-node2, so without nodeSelector the pod might land on another node and fail to start
