**Question 1:**
A deployment named ``log-aggregator`` in the ``secure-logging`` namespace is currently vulnerable to runtime tampering because its container uses a writable root filesystem. This could allow an attacker to modify application binaries or configuration files if the container is compromised.

The application requires write access only to the following specific directories for its normal operation:
* ``/var/log/application`` (for application logs)
* ``/tmp`` (for temporary processing files)
* ``/var/cache`` (for cached data)
* ``/var/run`` (for nginx runtime files like PID)

Your objective is to:

1. Harden the deployment by configuring its container to use a read-only root filesystem
2. Preserve application functionality by ensuring it can still write to the required directories listed above
3. Verify the solution by confirming the pod runs successfully and the logging service remains operational

Make the necessary changes to the deployment and validate that the application works correctly under the new security constraints.

**Step 1 — Check the existing deployment**
```
kubectl get deployment log-aggregator -n secure-logging -o yaml
```

**Step 2 — Patch the deployment**
```
spec:
  template:
    spec:
      containers:
      - name: log-container
        securityContext:
          readOnlyRootFilesystem: true

kubectl patch deployment log-aggregator \
  -n secure-logging \
  --type=strategic \
  --patch-file=patch.yaml
```


