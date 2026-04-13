**Question 1:**
The namespace ``encrypted`` has two applications, ``alpha`` and ``beta``.
Since these applications handle critical communications, enforce ``strict mTLS`` using Istio in the ``encrypted`` namespace.

Create a PeerAuthentication named ``default`` in the ``encrypted`` namespace with **STRICT mTLS** mode.

Make sure that the workloads have the istio sidecar injected.

Note: istio and istioctl have already been installed for you.

## Solution

**Step 1 — Enable Istio sidecar injection for the namespace**
```
kubectl label namespace encrypted istio-injection=enabled
```
Refer Document on how to enable istio injection: https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/


**Step 2 — Create the PeerAuthentication manifest**
```
cat <<EOF > peerauth.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: encrypted
spec:
  mtls:
    mode: STRICT
EOF
```
Refer document: https://istio.io/latest/docs/reference/config/security/peer_authentication/

**Step 3 — Apply it**
```
kubectl apply -f peerauth.yaml
```

**Step 4 — Verify PeerAuthentication was created**
```
kubectl get peerauthentication -n encrypted
kubectl describe peerauthentication default -n encrypted
```

**Step 5: Proceed to restart the deployments with the following commands**
```
kubectl rollout restart deployment alpha -n encrypted
kubectl rollout restart deployment beta -n encrypted
```

**Step 6: Verify sidecar is injected (should show 2/2 READY)**
```
kubectl get pods -n encrypted
## Excepted

root@controlplane ~ ✖ k get po
NAME                     READY   STATUS    RESTARTS   AGE
alpha-84487b99cf-4kfk8   2/2     Running   0          78s   # It should show 2/2 ready
beta-588b848cb6-qbnpw    2/2     Running   0          68s   # It should show 2/2 ready
```
