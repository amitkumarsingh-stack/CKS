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
-------------------------------------

**Question 3**: The ``web-apps`` namespace contains a frontend application that should only be accessible from specific sources.
Create a NetworkPolicy that:

1. Allows ingress traffic to pods with label app: frontend ONLY from:
    * Pods in the same namespace with label app: backend
    * Any pod in the monitoring namespace

2. Blocks all other ingress traffic to the frontend pods
3. Does not affect egress traffic

The policy should be named frontend-access and should apply to the ``web-app``s namespace.
Verify that the policy correctly allows and blocks traffic as specified.

## Solution
Key Concept Before Starting
```
Who can reach frontend pods (app: frontend)?

  ✅ Pods with label app: backend  (same namespace: web-apps)
  ✅ ANY pod from monitoring namespace
  ❌ Everything else (other namespaces, internet, other pods)

Egress from frontend pods → completely unaffected
```

**Step 1 — Investigate the existing setup**
```
# Check namespace exists
kubectl get namespace web-apps
kubectl get namespace monitoring

# Check pods and their labels
kubectl get pods -n web-apps --show-labels

# Verify frontend pods exist
kubectl get pods -n web-apps -l app=frontend

# Check monitoring namespace has the right label
# (needed for namespaceSelector)
kubectl get namespace monitoring --show-labels
```

**Step 2 — Label the monitoring namespace if needed**
```
# Check if monitoring namespace has a label
kubectl get namespace monitoring -o yaml | grep labels -A 5

# If no useful label exists, add one
kubectl label namespace monitoring \
  kubernetes.io/metadata.name=monitoring

# Verify
kubectl get namespace monitoring --show-labels
```

**Step 3 — Create the NetworkPolicy**
```
cat > frontend-access.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-access
  namespace: web-apps
spec:
  podSelector:
    matchLabels:
      app: frontend              # applies TO frontend pods only

  policyTypes:
  - Ingress                      # only affects ingress, NOT egress

  ingress:

  # Rule 1: Allow from backend pods in SAME namespace
  - from:
    - podSelector:
        matchLabels:
          app: backend           # pods with app=backend label
                                 # implicitly same namespace (web-apps)

  # Rule 2: Allow from ANY pod in monitoring namespace
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring   # monitoring namespace
EOF

kubectl apply -f frontend-access.yaml
```

**Step 5 — Test the policy**
Test 1 — Backend pod SHOULD reach frontend:
```
# Get a backend pod
kubectl get pods -n web-apps -l app=backend

# Should SUCCEED
kubectl exec -n web-apps <backend-pod> -- \
  curl -m 5 http://<frontend-pod-ip>:80

# Expected: HTML response ✅
```

Test 2 — Monitoring namespace pod SHOULD reach frontend:
```
# Get a pod in monitoring namespace
kubectl get pods -n monitoring

# Should SUCCEED
kubectl exec -n monitoring <any-pod> -- \
  curl -m 5 http://<frontend-pod-ip>:80

# Expected: HTML response ✅
```

Test 3 — Other namespace pods should be BLOCKED:
```
# Try from default namespace
kubectl run test-blocked \
  --image=curlimages/curl \
  --rm -it \
  -n default \
  -- curl -m 5 http://<frontend-pod-ip>:80

# Expected: Connection timed out ✅
```
------------------------------------
**Question 4**
The ``external-services`` namespace contains applications that need controlled access to external APIs and services.
Create a NetworkPolicy that:
1. Allows egress traffic ONLY to specific external services:
  * DNS servers (UDP port 53)
  * HTTPS services (TCP port 443)
  * A specific API endpoint at api.company.com (TCP port 8443) - assume this resolves to IP range 192.168.100.0/24
2. Blocks all other egress traffic from the namespace
3. Applies to all pods in the ``external-services`` namespace

The policy should be named ``restrict-egress`` and should use CIDR blocks for the allowed destinations.

## Solution
**Step 1 — Create the NetworkPolicy manifest**
```
cat <<EOF > restrict-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: external-services
spec:
  podSelector: {}          # applies to ALL pods in the namespace
  policyTypes:
  - Egress
  egress:
  # Allow DNS (UDP port 53)
  - ports:
    - protocol: UDP
      port: 53

  # Allow HTTPS (TCP port 443)
  - ports:
    - protocol: TCP
      port: 443

  # Allow api.company.com (TCP port 8443) via CIDR
  - to:
    - ipBlock:
        cidr: 192.168.100.0/24
    ports:
    - protocol: TCP
      port: 8443
EOF
```

**Step 2 — Apply it**
```
kubectl apply -f restrict-egress.yaml
```
----------------------------------------
**Question 5**
Secure pods in the  ``node-security`` namespace by preventing access to the node metadata service ``(169.254.169.254)``.

Tasks:
* Create a NetworkPolicy named block-metadata-access that blocks all egress traffic to the metadata service IP
* Ensure the policy allows all other egress traffic
* Test that ``metadata access`` is blocked while maintaining DNS functionality

Note: A test deployment metadata-test-pod is already running in the namespace for validation.

## Solution

**Step 1 — Create the NetworkPolicy**
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-metadata-access
  namespace: node-security
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS (UDP + TCP port 53)
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # Allow all egress EXCEPT metadata IP
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
EOF
```

**Step 2 — Test metadata access is BLOCKED**
```
# Exec into the test pod
kubectl exec -n node-security deployment/metadata-test-pod -- \
  wget -qO- --timeout=5 http://169.254.169.254/

# Expected: connection timeout or refused — BLOCKED
```

**Step 3 — Test DNS still works**
```
kubectl exec -n node-security deployment/metadata-test-pod -- \
  nslookup kubernetes.default.svc.cluster.local

# Expected: successful DNS resolution — ALLOWED
```
-------------------
**Question 6**

Create a NetworkPolicy in the ``egress-control`` namespace that restricts egress traffic to only allow:

1. DNS queries (UDP/TCP port 53) to any destination
2. HTTPS traffic (TCP port 443) to public IP ranges (excluding private IPs)
3, Block all other egress traffic

NetworkPolicy Requirements:
* Name: ``restrict-egress``
* Namespace: ``egress-control``
* Apply to all pods in the namespace (empty podSelector)
* Use CIDR 0.0.0.0/0 with exceptions for private IP ranges:
  * 10.0.0.0/8
  * 172.16.0.0/12
  * 192.168.0.0/16

The policy should apply to all pods in the namespace. Use ipBlock with except to exclude private IPs.

**Step 1: Create Network Policy**
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: egress-control
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Rule 1 — Allow DNS (UDP + TCP port 53) to anywhere
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53

  # Rule 2 — Allow HTTPS (TCP port 443) to public IPs only
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
EOF
```

**Step 2: Test Network Policy**
```
# Run a test pod
kubectl run test --rm -it \
  --image=busybox \
  -n egress-control -- sh

# Inside pod:

# DNS — should WORK
nslookup google.com

# HTTPS to public IP — should WORK
wget -qO- --timeout=5 https://google.com

# HTTPS to private IP — should be BLOCKED
wget -qO- --timeout=5 https://192.168.1.1

# HTTP port 80 — should be BLOCKED
wget -qO- --timeout=5 http://google.com
```
-----------------------
**Question 7**

Secure the Kubernetes Dashboard deployment by implementing network segmentation:
Create a NetworkPolicy named ``restrict-dashboard-access`` in the ``kubernetes-dashboard`` namespace with the following specifications:
* Apply to all pods with label ``k8s-app: kubernetes-dashboard``
* Only allow ingress traffic from within the same namespace (kubernetes-dashboard)
* Only allow TCP traffic on port 8443
* Block all other ingress traffic

This will restrict dashboard access to only pods within the dashboard namespace.

**Step 1 — Create the NetworkPolicy**
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-dashboard-access
  namespace: kubernetes-dashboard
spec:
  podSelector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kubernetes-dashboard
    ports:
    - protocol: TCP
      port: 8443
EOF
```

**Step 2 — Test ingress is allowed from same namespace**
```
# Create a test pod in kubernetes-dashboard namespace
kubectl run test-allowed \
  --image=busybox \
  -n kubernetes-dashboard \
  -- sleep 3600

# Test connection to dashboard (should WORK)
kubectl exec -n kubernetes-dashboard test-allowed -- \
  wget -qO- --timeout=5 https://kubernetes-dashboard:8443
```