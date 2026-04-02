**Question 1**: Solve this question on: ssh ``cks7262``
There is a Deployment ``container-host-hacker`` in Namespace ``team-red`` which mounts ``/run/containerd`` as a hostPath volume on the Node where it's running. This means that the Pod can access various data about other containers running on the same Node.
To prevent this configure Namespace team-red to enforce the baseline Pod Security Standard. Once completed, delete the Pod of the Deployment mentioned above.
Check the ReplicaSet events and write the event/log lines containing the reason why the Pod isn't recreated into ``/opt/course/4/logs`` on cks7262.

**Answer**
To solve the question check the dosumnetation: https://kubernetes.io/docs/tutorials/security/ns-level-pss/

**Step 1:** Investigate the existing Deployment
```
# Check the deployment
kubectl get deployment container-host-hacker -n team-red

# Check the pod
kubectl get pods -n team-red

# Confirm the hostPath volume mount
kubectl get deployment container-host-hacker -n team-red -o yaml | grep -A 10 hostPath
```
You'll see something like:
```
volumes:
- name: containerd-sock
  hostPath:
    path: /run/containerd   # ← dangerous - exposes host container runtime
```

**Step 2**: Label the Namespace to enforce baseline PSS
```
kubectl label namespace team-red \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest
```

**Step 3**: Delete the Pod of the Deployment
```
# Get the pod name
kubectl get pods -n team-red

# Delete the pod (NOT the deployment)
kubectl delete pod <pod-name> -n team-red
```
**Note**: Delete the Pod, not the Deployment. The Deployment's ReplicaSet will try to recreate it — but it will fail because of the new PSS enforcement. That failure is exactly what the next step captures.

**Step 4**: Check ReplicaSet events for the failure reason
```
# Get the ReplicaSet name
kubectl get rs -n team-red

# Describe the ReplicaSet and check events
kubectl describe rs <replicaset-name> -n team-red
```

You'll see events at the bottom like:
```
Events:
  Warning  FailedCreate  ReplicaSet pods "container-host-hacker-xxxx" 
  is forbidden: violates PodSecurity "baseline:latest": 
  hostPath volumes not permitted
```

**Step 5**: Save the event/log lines to the required path
```
# Get the events and save to file
kubectl describe rs <replicaset-name> -n team-red \
  | grep -A 5 "FailedCreate" \
  > /opt/course/4/logs

# Verify
cat /opt/course/4/logs
```