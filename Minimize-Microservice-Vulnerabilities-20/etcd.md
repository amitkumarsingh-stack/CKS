**Question 1:** Use context: ``kubectl config use-context workload-prod``
There is an existing Secret called ``database-access`` in Namespace ``team-green``.
1. Read the complete Secret content directly from ``ETCD`` (using etcdctl) and store it into ``/opt/course/11/etcd-secret-content``. 
2. Write the plain and decoded Secret's value of key pass into ``/opt/course/11/database-password``.

## Solution
The question specifically asks to read directly from ETCD — bypassing the API server entirely.

**Step 1 — Switch context and SSH to control plane**
```
kubectl config use-context workload-prod
ssh cluster1-controlplane1   # or whatever the control plane node is
sudo -i
```

**Step 3 — Query ETCD directly**
```
ETCDCTL_API=3 etcdctl \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  get /registry/secrets/team-green/database-access
```
Kubernetes stores secrets in ETCD under this path format:
```
/registry/secrets/<namespace>/<secret-name>
```

So for our secret:
```
/registry/secrets/team-green/database-access
```
You'll see raw output like:
```
/registry/secrets/team-green/database-access
k8s

v1Secret
...
database-access  team-green
...
pass    c2VjcmV0cGFzc3dvcmQ=    ← base64 encoded value
...
```

**Step 4 — Save complete ETCD content to file**
```
ETCDCTL_API=3 etcdctl \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  get /registry/secrets/team-green/database-access > /opt/course/11/etcd-secret-content
```

**Step 5 — Get the decoded value of key pass**
```
kubectl get secret database-access \
  -n team-green \
  -o jsonpath='{.data.pass}' \
  | base64 -d \
  > /opt/course/11/database-password

# Verify
cat /opt/course/11/database-password
# Should show plain text password e.g: secretpassword
```


