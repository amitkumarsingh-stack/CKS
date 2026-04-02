**Question 1:** Solve this question on: ``ssh cks8930``
You're asked to update the cluster's KubeletConfiguration. Implement the following changes in the Kubeadm way that ensures new Nodes added to the cluster will receive the changes too.

Set ``containerLogMaxSize`` to ``5Mi``
Set ``containerLogMaxFiles`` to ``3``
Apply the changes for the Kubelet on ``cks8930``
Apply the changes for the Kubelet on ``cks8930-node1``.
Connect with ssh ``cks8930-node1`` from ``cks8930``


💡 Use ``sudo -i`` to become root which may be required for this question

**Solution**

#### Key Concept Before Starting
The question says "Kubeadm way" — this is very important. It means:

* ❌ Don't just edit /var/lib/kubelet/config.yaml directly on each node
* ✅ Update the ConfigMap in kube-system that kubeadm uses as the source of truth
* This ensures new nodes joining the cluster automatically get the same config

The kubeadm kubelet config is stored in:
```
ConfigMap: kubelet-config
Namespace: kube-system
```

**Step 1 — SSH and become root**
```
ssh cks8930
sudo -i
```

**Step 2 — Check the current kubelet ConfigMap**
```
kubectl get configmap kubelet-config -n kube-system -o yaml
```
You'll see the KubeletConfiguration inside the ``data`` field:
```
data:
  kubelet: |
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    cgroupDriver: systemd
    clusterDNS:
    - 10.96.0.10
    ...
    # containerLogMaxSize and containerLogMaxFiles may be missing
```

**Step 3 — Edit the kubelet ConfigMap (the kubeadm way)**
```
kubectl edit configmap kubelet-config -n kube-system
```
Find the ``kubelet:`` section inside ``data`` and add the two settings:
```
data:
  kubelet: |
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    cgroupDriver: systemd
    clusterDNS:
    - 10.96.0.10
    containerLogMaxSize: 5Mi      # ← add this
    containerLogMaxFiles: 3       # ← add this
    ...
```
Save and exit.

**Step 4 — Apply changes to cks8930 (control plane node)**
The kubeadm way to pull config from the ConfigMap down to the node:

```
# Pull the updated kubelet config from ConfigMap to local config file
kubeadm upgrade node phase kubelet-config
```
This updates ``/var/lib/kubelet/config.yaml`` on the node with the latest ConfigMap values.

Then restart kubelet:
```
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
```

**Step 5 — Verify on cks8930**
```
# Confirm the values are in the local kubelet config
cat /var/lib/kubelet/config.yaml | grep -i "containerLog"

# Expected output:
# containerLogMaxFiles: 3
# containerLogMaxSize: 5Mi
```

**Step 6 — Apply changes to cks8930-node1 (worker node)**
```
# From cks8930, SSH to the worker node
ssh cks8930-node1
sudo -i
```

Run the same kubeadm command to pull config from ConfigMap:
```
kubeadm upgrade node phase kubelet-config
```
Restart kubelet:
```
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
```

**Step 7 — Verify on cks8930-node1**
```
# Confirm values applied on worker node too
cat /var/lib/kubelet/config.yaml | grep -i "containerLog"

# Expected:
# containerLogMaxFiles: 3
# containerLogMaxSize: 5Mi
```

##Documentation
```
# KubeletConfiguration fields reference
https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/

# Kubeadm kubelet config management
https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-kubelet-config/
```