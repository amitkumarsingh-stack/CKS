**Question 1:** Use context: ``kubectl config use-context infra-prod``
There is a metadata service available at ``http://192.168.100.21:32000`` on which Nodes can reach sensitive data, like cloud credentials for initialisation. By default, all Pods in the cluster also have access to this endpoint. The DevSecOps team has asked you to restrict access to this metadata server.
In Namespace ``metadata-access``:

* Create a NetworkPolicy named ``metadata-deny`` which prevents egress to ``192.168.100.21`` for all Pods but still allows access to everything else
* Create a NetworkPolicy named ``metadata-allow`` which allows Pods having label ``role: metadata-accessor`` to access endpoint ``192.168.100.21``

There are existing Pods in the target Namespace with which you can test your policies, but don't change their labels.

## Solution
**Key Concept Before Starting**

This is a classic cloud metadata endpoint protection pattern. In cloud environments (AWS, GCP, Azure), the metadata service exposes sensitive data like IAM credentials. Unrestricted pod access to this endpoint is a serious security risk.

The challenge here is:

* Block 192.168.100.21 for ALL pods
* But allow it for pods with label role: metadata-accessor
* Keep everything else working normally

This requires two policies working together.

**Step 1 — Switch context and investigate**
```
kubectl config use-context infra-prod

# Check existing pods and their labels
kubectl get pods -n metadata-access --show-labels

# You'll see pods like:
# NAME              LABELS
# accessor-pod      role=metadata-accessor   ← should be allowed
# regular-pod       app=regular              ← should be blocked
```

**Step 2 — Understand the policy logic**
```
All Pods
   │
   ├── egress to 192.168.100.21  → BLOCKED  (metadata-deny)
   │
   └── EXCEPT pods with role=metadata-accessor
              │
              └── egress to 192.168.100.21  → ALLOWED  (metadata-allow)

All other egress (internet, DNS, other pods) → ALLOWED for everyone
```

**Policy 1 — metadata-deny**
Deny egress to 192.168.100.21 for ALL pods but allow everything else.

The trick here is to use two egress rules — one that allows all traffic EXCEPT to the metadata IP:
```
cat > metadata-deny.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: metadata-deny
  namespace: metadata-access
spec:
  podSelector: {}                # applies to ALL pods in namespace
  policyTypes:
  - Egress
  egress:
  - to:                          # allow egress to everything
    - ipBlock:
        cidr: 0.0.0.0/0          # all IPs
        except:
        - 192.168.100.21/32      # EXCEPT the metadata server
EOF

kubectl apply -f metadata-deny.yaml
```

**Policy 2 — metadata-allow**

Allow pods with label ``role: metadata-accessor`` to reach ``192.168.100.21``:
```
cat > metadata-allow.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: metadata-allow
  namespace: metadata-access
spec:
  podSelector:
    matchLabels:
      role: metadata-accessor    # only pods with this label
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.100.21/32  # allow access to metadata server
EOF

kubectl apply -f metadata-allow.yaml
```

**Step 3 — Verify both policies are created**
```
kubectl get networkpolicy -n metadata-access

# Expected:
# NAME              POD-SELECTOR              AGE
# metadata-deny     <none> (all pods)         10s
# metadata-allow    role=metadata-accessor    5s
```

**Step 4 — Test the policies**

Test 1 — Regular pod should NOT reach metadata server:
```
# Get a regular pod (without role=metadata-accessor label)
kubectl get pods -n metadata-access --show-labels | grep -v metadata-accessor

# Try to curl metadata server — should FAIL
kubectl exec -n metadata-access <regular-pod> -- \
  curl -m 5 http://192.168.100.21:32000

# Expected: curl: (28) Connection timed out ✅
```

Test 2 — Regular pod should still reach everything else:
```
# Other internet/cluster traffic should still work
kubectl exec -n metadata-access <regular-pod> -- \
  curl -m 5 http://google.com

# Expected: Response received ✅
```

Test 3 — Accessor pod SHOULD reach metadata server:
```
# Get the pod with role=metadata-accessor label
kubectl get pods -n metadata-access -l role=metadata-accessor

# Should SUCCEED
kubectl exec -n metadata-access <accessor-pod> -- \
  curl -m 5 http://192.168.100.21:32000

# Expected: Response from metadata server ✅
```
---------------------------------------

**Question 2**: A security team has identified that pods in the ``threat-prevention`` namespace are attempting to connect to known malicious IP ranges used for command and control servers.
Create a NetworkPolicy that:

1. Blocks ALL egress traffic to the following malicious CIDR ranges:
    * 192.168.100.0/24
    * 10.0.99.0/24

2. Allows DNS traffic (UDP and TCP port 53) to ensure basic network functionality
3. Allows all other egress traffic except the blocked malicious ranges
4. Applies to all pods in the threat-prevention namespace

The policy should be named ``block-malicious-egress`` and should not affect ingress traffic.

## Solution
Key Concept Before Starting
This question has 3 egress rules that must work together:
```
All Pods egress traffic
      │
      ├── To 192.168.100.0/24  → BLOCK ❌
      ├── To 10.0.99.0/24      → BLOCK ❌
      ├── To port 53 UDP/TCP   → ALLOW ✅ (DNS)
      └── Everything else      → ALLOW ✅
```

**Step 1 — Switch context and check namespace**
```
kubectl get pods -n threat-prevention --show-labels
```

Sample Policy
```
egress:
- to:                         # Rule 1: Allow DNS
  ports:
  - port 53 UDP
  - port 53 TCP

- to:                         # Rule 2: Allow everything except bad IPs
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 192.168.100.0/24
      - 10.0.99.0/24
```
Refer document : https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy-resource

**Step 3 — Create the NetworkPolicy**
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-malicious-egress
  namespace: threat-prevention
spec:
  podSelector: {}              # applies to ALL pods in namespace
  policyTypes:
  - Egress                     # only affects egress, NOT ingress
  egress:

  # Rule 1: Allow DNS (UDP + TCP port 53)
  # Without this pods can't resolve hostnames
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53

  # Rule 2: Allow all egress EXCEPT the two malicious CIDR ranges
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0        # all IPs
        except:
        - 192.168.100.0/24     # block malicious range 1
        - 10.0.99.0/24         # block malicious range 2
```

**Step 4 — Verify the policy was created**
```
kubectl get networkpolicy -n threat-prevention
```

**Step 5 — Test the policy**
Test 1 — Malicious IPs should be BLOCKED:
```
kubectl get pods -n threat-prevention

# Try to reach malicious IP — should FAIL
kubectl exec -n threat-prevention <pod-name> -- \
  curl -m 5 http://192.168.100.1
# Expected: curl: (28) Connection timed out ✅

kubectl exec -n threat-prevention <pod-name> -- \
  curl -m 5 http://10.0.99.1
# Expected: curl: (28) Connection timed out ✅
```

Test 2 — DNS should still work:
```
kubectl exec -n threat-prevention <pod-name> -- \
  nslookup kubernetes.default.svc.cluster.local
# Expected: Valid DNS response ✅
```

Test 3 — Other traffic should still work:
```
kubectl exec -n threat-prevention <pod-name> -- \
  curl -m 5 http://8.8.8.8
# Expected: Response received ✅

kubectl exec -n threat-prevention <pod-name> -- \
  curl -m 5 http://10.0.0.1
# Expected: Response or connection refused (not timeout) ✅
```


