**Question 1**: Use context: ``kubectl config use-context infra-prod``
You're asked to evaluate specific settings of cluster2 against the CIS Benchmark recommendations. Use the tool kube-bench which is already installed on the nodes.
Connect using ``ssh cluster2-controlplane1`` and ``ssh cluster2-node1``.
On the master node ensure (correct if necessary) that the CIS recommendations are set for:

1. The ``--profiling`` argument of the kube-controller-manager
2. The ownership of directory ``/var/lib/etcd``

On the worker node ensure (correct if necessary) that the CIS recommendations are set for:

3. The permissions of the kubelet configuration ``/var/lib/kubelet/config.yaml``
4. The ``--client-ca-file`` argument of the kubelet

**Step 1**: Switch context
```
kubectl config use-context infra-prod
```

**Step 2**: Get nodes
```
kubectl get no
```
You should see ``cluster2-controlplane1`` and ``cluster2-node1``

**Step 3**: Master Node Fixes
**Sub-Step: 1**
```
ssh cluster2-controlplane1
sudo -i
```

**Sub-Step 2**: Run kube-bench on master node
```
kube-bench run --targets master
```
This outputs all CIS benchmark checks. Rather than reading everything, filter for what you need:

```
# Check profiling finding
kube-bench run --targets master | grep -A 3 "profiling"

# Check etcd ownership finding
kube-bench run --targets master | grep -A 3 "etcd"
```

Fix 1 — ``--profiling`` argument on kube-controller-manager
CIS Benchmark says: ``--profiling=false`` must be set on the controller manager.
Check current state:
```
cat /etc/kubernetes/manifests/kube-controller-manager.yaml | grep profiling
```
If it's missing or set to true, edit the static pod manifest:
```
vi /etc/kubernetes/manifests/kube-controller-manager.yaml
```

Find the command section and add/update:
```
spec:
  containers:
  - command:
    - kube-controller-manager
    - --profiling=false       # ← add this line
    - --other-existing-args
    ...
```
Save and exit. Kubelet will automatically restart the controller manager static pod.
Verify:
```
# Wait a few seconds for restart
kubectl get pod -n kube-system | grep controller-manager

# Confirm the arg is present
kubectl describe pod kube-controller-manager-cluster2-controlplane1 \
  -n kube-system | grep profiling
```

Fix 2 — Ownership of ``/var/lib/etcd``
CIS Benchmark says: ``/var/lib/etcd`` must be owned by ``etcd:etcd``.
Check current ownership:
```
ls -la /var/lib/ | grep etcd
```
If ownership is wrong (e.g., owned by ``root:root``):
```
chown etcd:etcd /var/lib/etcd
```

Verify:
```
ls -la /var/lib/ | grep etcd
# Expected: drwx------ 3 etcd etcd ... /var/lib/etcd
```

**Worker Node Fixes**
```
ssh cluster2-node1
sudo -i
```

Run kube-bench on worker node
```
kube-bench run --targets node

# Filter for specific findings
kube-bench run --targets node | grep -A 3 "config.yaml\|permissions"
kube-bench run --targets node | grep -A 3 "client-ca-file"
```

Permissions of ``/var/lib/kubelet/config``.yaml
CIS Benchmark says: File permissions must be 600 or more restrictive.
Check current permissions:
```
ls -la /var/lib/kubelet/config.yaml
```
If permissions are too open (e.g., 644 or 777):
```
chmod 600 /var/lib/kubelet/config.yaml
```

Verify:
```
ls -la /var/lib/kubelet/config.yaml
# Expected: -rw------- 1 root root ... config.yaml
```

``--client-ca-file`` argument on kubelet
CIS Benchmark says: Kubelet must have ``--client-ca-file`` set to authenticate API server requests.
Check current kubelet config:

```
# Check if set as a flag in the service
ps aux | grep kubelet | grep client-ca-file

# Check the kubelet config file
cat /var/lib/kubelet/config.yaml | grep clientCA
```

If missing, edit the kubelet config file:
```
vi /var/lib/kubelet/config.yaml
```

Add or update:
```
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
clientCAFile: /etc/kubernetes/pki/ca.crt    # ← add this
...
```
The CA file is almost always at /etc/kubernetes/pki/ca.crt — verify it exists first:
```
ls /etc/kubernetes/pki/ca.crt
```

Restart kubelet after the change:
```
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
```

Verify
```
ps aux | grep kubelet | grep client-ca-file
# OR
cat /var/lib/kubelet/config.yaml | grep clientCA
```

