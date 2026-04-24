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
------------------------------------------------------------
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
------------------------------------------------------------
**Question4**: A pod in the ``crypto-monitor`` namespace is suspected of running crypto-mining software.
Create a Falco rule that detects the execution of known mining processes like ``xmrig``,`` minerd``, or ``cpuminer``.
The rule should be added to ``/etc/falco/falco_rules.local.yaml`` with below specs:

1. Trigger when any of these processes are executed: ``xmrig``, ``minerd``, ``cpuminer``
Set the priority to ``CRITICAL``
2. Output the format: ``MINING_ALERT: %evt.time,%container.name,%proc.name``
3. Tag the events with ``[container, crypto_mining, mitre_execution]``

Make sure the rule persists across Falco updates by adding it to the local rules file.

## Solution
**Step 1: Check the status of Falco Service**
```
systemctl status falco
```

**Step 2 — Check existing rules for reference (good habit)**
```
cat /etc/falco/falco_rules.yaml | grep proc.name -A 10 -B 10
```
Here you can just copy the matching rule to file ``/etc//falco/falco_rules.local.yaml``

Always make sure that you write the new rules to ``/etc//falco/falco_rules.local.yaml``

**Step 3: Write the rule to local rules files**
```
- rule: Detect Crypto Mining Process
  desc: Detects execution of known crypto mining processes inside containers
  condition: >
    container.id != host
    and evt.type = execve
    and proc.name in (xmrig, minerd, cpuminer)
  output: >
    MINING_ALERT: %evt.time,%container.name,%proc.name
  priority: CRITICAL
  tags: [container, crypto_mining, mitre_execution]
```
Refer document https://falco.org/docs/concepts/rules/basic-elements/ for writing the rule

Rule Field Breakdown
```
condition: >
  container.id != host          # only trigger inside containers
                                # (not on host processes)
  and evt.type = execve         # when a process is EXECUTED
                                # (not just running)
  and proc.name in              # process name matches any of:
    (xmrig, minerd, cpuminer)   # the known mining tools

output: >
  MINING_ALERT:                 # literal prefix string
  %evt.time,                    # timestamp with nanoseconds
  %container.name,              # name of the container
  %proc.name                    # name of the executed process

priority: CRITICAL              # highest severity level

tags:
  - container                   # relates to container activity
  - crypto_mining               # custom tag for this threat type
  - mitre_execution             # MITRE ATT&CK: Execution tactic
```

**Step 4- Restart Falco to load the new rules**
```
systemctl restart falco

# Verify it started without errors
systemctl status falco

# Check the rule loaded in logs
journalctl -u falco | tail -20 | grep -i "crypto\|mining\|error"

OR

falco -U | tail -20 | grep -i "crypto\|mining\|error"
```

**Step 5 — Test the rule is working**
```
# Get a pod in crypto-monitor namespace
kubectl get pods -n crypto-monitor

# Exec into a pod and simulate running a mining process
kubectl exec -n crypto-monitor <pod-name> -- \
  sh -c "echo simulating; ls /tmp/xmrig 2>/dev/null || true"
```

Then check Falco logs for alerts
```
falco -U | tail -20 | grep "MINING_ALERT"
```
------------------------------------------------------------
**Question 5:**

A Pod is behaving improperly and poses a security threat to the system.
A Pod belonging to the application analytics is abnormal. It is reading sensitive credential data by accessing the file ``/etc/shadow``.
First, identify the misbehaving Pod that is accessing ``/etc/shadow``.
Next, scale the Deployment of the misbehaving Pod down to zero replicas.

Notes:
* Do not modify anything else in that Deployment besides scaling replicas.
* Do not modify any other Deployments.
* Do not delete any Deployments.

## Solution

**Step 1: Write a custom Falco Rule**
You can look at /etc/falco/falco_rules.yaml for template examples of rule structure.
```
# Your custom rules!
- list: shadow-file
  items: [/etc/shadow]

- rule: shadow-read
  desc: shadow-read
  condition: >
    fd.name in (shadow-file)
  output: >
    Sensitive file accessed (container_id=%container.id)
  priority: NOTICE
  tags: [file]
```

**Step 2: Run Falco for 30 seconds to capture events**
```
falco -M 30 -r /etc/falco/falco_rules.local.yaml > shadow.log
```

**Step 3: Check the output for container IDs**
```
cat shadow.log
```
Sample output:
```
11:15:20.180930443: Notice Sensitive file accessed (container_id=a1b2c3d4e5f6)
11:15:30.184015793: Notice Sensitive file accessed (container_id=a1b2c3d4e5f6)
```

**Step 4: Find the Pod from the container ID**
```
crictl ps | grep a1b2c3d4e5f6
```

**Step 5: Identify the Deployment**
```
kubectl get pod,deploy -n privilege-monitor | grep collector
```

**Step 6: Scale the Deployment to zero**
```
kubectl scale deployment collector -n privilege-monitor --replicas 0
```