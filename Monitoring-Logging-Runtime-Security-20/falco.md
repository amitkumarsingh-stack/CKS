## Falco CKS exam questions

**Question 1**: Falco is installed on `node01`. A misbehaving workload is spawning shells inside containers. Find all Falco events related to shell spawning, and save them to `/opt/cks/falco/shells.log` on the jumpbox in this exact format:
`timestamp,container-id,container-name,image`
Collect logs for at least 30 seconds.

#### Step1: SSH to node01 and check Falco is running
```
ssh node01  # Falco is installed on node01
systemctl status falco # This checks the status of Falco service
```

#### Step 2: Find relevant Falco events in syslog
```
cat /var/log/syslog | grep falco | grep -i "shell" # Falco logs are by default stored in /var/log/syslog
```

#### Step 3: Collect for 30+ seconds and format output
```
# Tail live for 30s and filter
timeout 35 journalctl -fu falco | grep -i "shell" > /tmp/raw-shells.log
```
Paste the logs in the required format to ``node`` on path ``/opt/cks/falco/shells.log``

------------------------------------------------------------

**Question2**: Falco on node01 is missing a rule to detect writes to /etc/passwd. Create a custom rule in the correct override file that triggers a WARNING whenever any process writes to /etc/passwd inside a container. Output must include: time, pod name, container name, user, and command. Restart Falco and confirm the rule loads without errors.

#### Step 1: SSH to node01
```
ssh node01
```

#### Step 2: Write the rule to the local override file
```
# falco_rules.local.yaml override the rule. Alway try to use this file for writing the rules instead of /etc/falco/falco_rules.yaml. Usually this file will be empty always

sudo vi /etc/falco/falco_rules.local.yaml
```
Add
```
- rule: Detect Write to /etc/passwd in Container
  desc: Detects any process writing to /etc/passwd inside a container
  condition: >
    container.id != host
    and open_write
    and fd.name = /etc/passwd
  output: >
    Write to /etc/passwd detected
    (time=%evt.time pod=%k8s.pod.name container=%container.name
    user=%user.name cmd=%proc.cmdline)
  priority: WARNING
  tags: [filesystem, cks]
```

#### Step 3: Restart Falco and check for errors
```
sudo systemctl restart falco
sudo systemctl status falco

# Confirm no rule errors
journalctl -u falco | tail -20 | grep -i error
```

You can also use below command to view Falco Logs
```
falco -U
```

#### Step 4: Verify the rule is loaded
```
journalctl -u falco | grep "Detect Write to /etc/passwd"

# Or trigger it manually:

kubectl exec -it <any-pod> -- sh -c "echo test >> /etc/passwd"

cat /var/log/syslog | grep "Write to /etc/passwd"
```

**Question3**: Using Falco on ``node01``, identify which Pod is accessing sensitive files under ``/proc`` on the host. Scale down that Deployment to 0 replicas.

#### Step 1: Find the offending events
Check where are the logs are been saved
```
ssh node01
cat /etc/falco/falco.yaml | grep -i output -A 3
```
Note: 
1. Check for ``syslog_output`` if it's ``enabled``
2. Check for ``file_output``. If it's enabled where is the file path where logs are written.

```
ssh node01
cat /var/log/syslog | grep falco | grep "/proc"
```
Note the ``container ID``, ``container_name`` or ``k8s.pod.name`` from the output.

Once you get the ``container ID``
```
crictl ps -d <CONTAINER_ID> # This will give you the ID of POD.
```
Once you get the POD ID
```
crictl pods -id <POD_ID> # This give you the name of POD and in which namespace it's running
```

#### Step 2: Map container to Pod and Deployment
```
exit  # back to jumpbox

# Find which namespace the pod is in
kubectl get pods -A | grep <pod-name-from-logs>

# Find its deployment
kubectl get deploy -n <namespace>
```
#### Step 3: Scale down to zero
```
kubectl scale deployment <deployment-name> -n <namespace> --replicas=0

# Verify
kubectl get pods -n <namespace>
```

