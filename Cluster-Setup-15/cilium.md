**Question 1:** Solve this question on: ``ssh cks7262``
In Namespace ``team-orange`` a ``Default-Allow`` strategy for all Namespace-internal traffic was chosen. There is an existing CiliumNetworkPolicy ``default-allow`` which assures this and which should not be altered. That policy also allows cluster internal DNS resolution.

Now it's time to deny and authenticate certain traffic. Create 3 CiliumNetworkPolicies in Namespace team-orange to implement the following requirements:

1. Create a ``Layer 3`` policy named ``p1`` to:
Deny outgoing traffic from Pods with label ``type=messenger`` to Pods behind Service ``database``

2. Create a ``Layer 4`` policy named ``p2`` to:
Deny outgoing ``ICMP traffic from Deployment transmitter`` to Pods behind Service ``database``


3. Create a ``Layer 3`` policy named ``p3`` to:
Enable ``Mutual Authentication`` for outgoing traffic from Pods with label ``type=database`` to Pods with label ``type=messenger``

💡 All Pods in the Namespace run plain Nginx images with open port 80. This allows simple connectivity tests like: ``k -n team-orange exec POD_NAME -- curl database``

### Solution
**Step 1 — Switch context and SSH**
```
ssh cks7262
sudo -i
```

**Step 2 — Investigate existing setup**
```
# Check existing policy
kubectl get ciliumnetworkpolicy -n team-orange

# Check existing services and pods
kubectl get svc -n team-orange
kubectl get pods -n team-orange --show-labels
kubectl get deployment -n team-orange
```

Find the selector used by the ``database`` service:
```
kubectl get svc database -n team-orange -o yaml | grep -A 5 selector
# Example: app=database or type=database
```

**Policy 1 — p1 (Layer 3: Deny messenger → database)**

Layer 3 = based on pod identity/labels (no port/protocol specification).
```
cat > p1.yaml << 'EOF'
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: p1
  namespace: team-orange
spec:
  endpointSelector:
    matchLabels:
      type: messenger
  egressDeny:
  - toEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": team-orange
        type: database
EOF

kubectl apply -f p1.yaml
```
Note: Refer document: https://docs.cilium.io/en/stable/security/policy/layer3/#simple-egress-deny

**Policy 2 — p2 (Layer 4: Deny ICMP transmitter → database)**

Layer 4 = based on protocol/port. Here we deny ICMP specifically.

First get the transmitter pod labels:
```
kubectl get deployment transmitter -n team-orange \
  -o yaml | grep -A 5 "matchLabels"
```

```
cat > p2.yaml << 'EOF'
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: p2
  namespace: team-orange
spec:
  endpointSelector:
    matchLabels:
      app: transmitter       # label from the transmitter deployment
  egressDeny:
  - toEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": team-orange
        type: database
    icmps:
    - fields:
      - type: 8              # ICMP type 8 = Echo Request (ping)
        family: IPv4
EOF

kubectl apply -f p2.yaml
```
Note: Refer document: https://docs.cilium.io/en/stable/security/policy/layer4/#example-icmp-icmpv6

**Policy 3 — p3 (Layer 3: Mutual Authentication database → messenger)**

Mutual Authentication in Cilium uses the ``authentication`` field:
```
cat > p3.yaml << 'EOF'
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: p3
  namespace: team-orange
spec:
  endpointSelector:
    matchLabels:
      type: database         # applies TO database pods
  egress:
  - toEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": team-orange
        type: messenger      # going TO messenger pods
    authentication:
      mode: "required"       # enforce mutual authentication
EOF

kubectl apply -f p3.yaml
```
Note: Refer document: https://docs.cilium.io/en/stable/network/servicemesh/mutual-authentication/mutual-authentication-example/

**Step 3 — Verify all policies are created**
```
kubectl get ciliumnetworkpolicy -n team-orange

# Expected:
# NAME            AGE
# default-allow   10m   ← existing, not touched
# p1              30s
# p2              25s
# p3              20s
```

**Step 4 — Verify policies are working**

Test p1 — messenger should NOT reach database:
```
# Get a messenger pod name
kubectl get pods -n team-orange -l type=messenger

# Try to curl database — should FAIL
kubectl exec -n team-orange <messenger-pod> -- curl database
# Expected: connection timeout or refused ✅
```

Test p2 — transmitter should NOT ping database:
```
kubectl get pods -n team-orange -l app=transmitter

# Ping should FAIL
kubectl exec -n team-orange <transmitter-pod> -- ping -c 1 database
# Expected: timeout ✅

# But HTTP should still WORK (only ICMP is denied)
kubectl exec -n team-orange <transmitter-pod> -- curl database
# Expected: nginx response ✅
```

Test p3 — mutual auth is required:
```
kubectl describe ciliumnetworkpolicy p3 -n team-orange | grep -i auth
```

⚠️ Critical exam tip: Always check if a default-allow policy exists before writing your policies. If it does, use egressDeny/ingressDeny instead of egress/ingress for deny rules — otherwise your deny policy will be overridden by the default-allow and won't work.