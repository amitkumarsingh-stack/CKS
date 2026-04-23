**Question 1:**
The kubectl commands executed on ``cluster2-controlplane`` are encountering TLS certificate errors.
Identify the issue within the kubeconfig file and take the necessary steps to resolve it.

If you are unable to execute the kubectl commands successfully, please refer to the kubeconfig backup file located at ``/root/cert-test/config.backup``.

## Solution

**Step 1: First, confirm that kubectl commands are failing:**
```
k get no
```
Error
```
cluster2-controlplane ~ ➜  k get no
error: unable to read certificate-authority /etc/kubernetes/pki/ca.crt.moved for kubernetes due to open /etc/kubernetes/pki/ca.crt.moved: no such file or directory
```

**Step 2: Examine the Kubeconfig File
```
grep "certificate-authority" ~/.kube/config
```
You'll see the incorrect path:

certificate-authority: /etc/kubernetes/pki/ca.crt.moved

**Step 3: Correct the Certificate Path**
```
vi .kube/config

apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt.moved
    server: https://cluster2-controlplane:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
users:
- name: kubernetes-admin
  user:
```
Corrected file
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://cluster2-controlplane:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
users:
- name: kubernetes-admin
  user:
```
You can check that file ``/etc/kubernetes/pki/ca.crt.moved`` does not exists

**Step 4: Alternative Solution - Restore from Backup**
If you prefer, you can restore the original kubeconfig:
```
cp /root/cert-test/config.backup ~/.kube/config
```

**Step 5: Test Cluster Connectivity**
Verify that kubectl commands now work:
```
kubectl get nodes
```