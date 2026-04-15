**Question 1:** A deployment named ``data-processor`` in the ``immutable-apps`` namespace is currently at risk of file system attacks due to the use of a writable root filesystem.
The deployment has the following volumes mounted for writable storage:

* /tmp
* /var/log
* /var/cache/nginx
* /var/run

Your task is to:

* Configure the containers to utilize a read-only root filesystem
* Ensure that the application retains the capability to write to the mounted volumes
* Confirm that the nginx web server remains operational

Please modify the deployment as necessary and test the application's functionality.

## Solution
Key Concept Before Starting
```
readOnlyRootFilesystem: true
        │
        ├── / (root)          → READ ONLY ❌ cannot write
        ├── /etc              → READ ONLY ❌
        ├── /usr              → READ ONLY ❌
        │
        └── Mounted volumes   → STILL WRITABLE ✅
            ├── /tmp          (emptyDir mount)
            ├── /var/log      (emptyDir mount)
            ├── /var/cache/nginx  (emptyDir mount)
            └── /var/run      (emptyDir mount)
```

**Step 1 — Investigate current deployment**
```
# Check current deployment
kubectl get deployment data-processor -n immutable-apps -o yaml
```

**Step 2 — Export the deployment**
```
kubectl get deployment data-processor \
  -n immutable-apps \
  -o yaml > /tmp/data-processor.yaml

# Backup
cp /tmp/data-processor.yaml /tmp/data-processor-backup.yaml
```

There are 2 methods you can solve this issue
1. By exporting and then fixing the issue - But it's had as you can mess the indentation
2. You can patch the deployment - Bit easy to do

We will go with 2nd method
```
vi patch.yaml
{
    "spec": {
        "template": {
            "spec": {
                "containers": [
                    {
                        "name": "processor-container",
                        "securityContext": {
                            "readOnlyRootFilesystem": true
                        }
                    }
                ]
            }
        }
    }
}
```

```
kubectl patch deployment data-processor -n immutable-apps --patch-file=./patch.yaml
```

Another way of paching te deployment (most easy)
Refer document : https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/#use-a-strategic-merge-patch-to-update-a-deployment

```
vi patch-file.yaml


spec:
  template:
    spec:
      containers:
      - name: processor-container
        securityContext:
          readOnlyRootFilesystem: true
```
