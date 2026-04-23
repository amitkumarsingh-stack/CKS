**Question 1:**

The administrator has partially upgraded cluster1.
Complete the upgrade process by updating the worker node to the latest installed version available among the nodes.

## Solution

**Step 1: Check current version across all the nodes**
```
kubectl get nodes
```
Excepted
```
NAME                    STATUS   ROLES           AGE     VERSION
cluster1-controlplane   Ready    control-plane   35m     v1.35.0
node02                  Ready    worker          2m24s   v1.34.0
```

**Step 2 — SSH into the worker node**
```
ssh node02
```

**Step 3 — Update the APT repository to v1.35**
```
vim /etc/apt/sources.list.d/kubernetes.list
```
Refer documentation https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/

**Note:** The legacy package repositories (apt.kubernetes.io and yum.kubernetes.io) have been deprecated and frozen starting from September 13, 2023. Using the new package repositories hosted at pkgs.k8s.io is strongly recommended and required in order to install Kubernetes versions released after September 13, 2023.

For new package repositories refer https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/#how-to-migrate-deb

Change the version:
```
# FROM
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /

# TO
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /
```

**Step 4 — Add the new release key**
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | \
  sudo gpg --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
**Note:** Both the steps are mentioned in the documentation

**Step 5 — Update and confirm version available**
```
sudo apt-get update
apt-cache madison kubeadm | grep 1.35
```
Excepted
```
node02 ~ ➜  apt-cache madison kubeadm | grep 1.35
   kubeadm | 1.35.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
   kubeadm | 1.35.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
   kubeadm | 1.35.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
   kubeadm | 1.35.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
   kubeadm | 1.35.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.35/deb  Packages
```

**Step 6 — Drain node02 (from controlplane)
```
# Run from controlplane
kubectl drain node02 \
  --ignore-daemonsets \
  --delete-emptydir-data
```
**Note:** Once the node is drain exit from there are ssh to node02

**Step 7 — Install kubeadm v1.35.0 (on node02)**
```
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.35.0-1.1
sudo apt-mark hold kubeadm
```
**Note:** Target version is ``1.35.0-1.1`` — always match exactly what the controlplane is running. The -1.1 suffix is the Debian package revision for that Kubernetes release.

**Step 8 — Upgrade the node**
```
sudo kubeadm upgrade node
```

**Step 9 - Upgrade kubelet and kubectl**
```
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
sudo apt-mark hold kubelet kubectl
```

**Step 10 — Restart kubelet**
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**Step 11 — Uncordon node02 (from controlplane)**
```
# Run from controlplane
kubectl uncordon node02
```

**Step 12 — Verify**
```
k get nodes
```

Excepted
```
NAME                    STATUS   ROLES           AGE   VERSION
cluster1-controlplane   Ready    control-plane   59m   v1.35.0
node02                  Ready    worker          26m   v1.35.0
```
Now the node is upgarded to the version of Master Node.
