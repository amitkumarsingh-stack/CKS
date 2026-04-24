**Question 1**: Use context: ``kubectl config use-context infra-prod``
The Open Policy Agent and Gatekeeper have been installed to, among other things, enforce blacklisting of certain image registries. Alter the existing constraint and/or template to also blacklist images from ``very-bad-registry.com``.

Test it by creating a single Pod using image ``very-bad-registry.com/image`` in Namespace ``default``, it shouldn't work.

You can also verify your changes by looking at the existing Deployment untrusted in Namespace default, it uses an image from the new untrusted source. The OPA constraint should throw violation messages for this one.

**Step 1 — Switch context**
```
kubectl config use-context infra-prod
```

**Step 2 — View the existing constraint & constrainttemplates**
```
kubectl get constraint
# You should see constraint. Verify it
```

```
kubectl get constrainttemplate
# You should see the constrainttemplate. Edit and modify it.
```

This is how it looks
```
package k8strustedimages

violation[{"msg": msg}] {
  image := input.review.object.spec.containers[_].image
  not startswith(image, "docker.io/")
  not startswith(image, "gcr.io/")
  msg := sprintf("Image not from trusted registry: %v", [image])
}
# Add a line for the registery which needs to be blocked
```

**Step 3 — Test by creating a Pod from the bad registry**
```
kubectl run test-bad \
  --image=very-bad-registry.com/image \
  -n default
```

Expected output:
```
Error from server ([pod-trusted-images] blacklisted registry: 
very-bad-registry.com/image): admission webhook 
"validation.gatekeeper.sh" denied the request
```
---------
**Question 2:**

OPA Gatekeeper is installed to enforce that all pods in the ``gatekeeper-demo`` namespace have resource limits defined. A ConstraintTemplate named ``k8srequiredresources`` is already created that validates CPU and memory limits. Create a Constraint that enforces this policy.

Tasks:
* Ensure Gatekeeper pods are ready in the ``gatekeeper-system`` namespace
* Verify the existing ConstraintTemplate ``k8srequiredresources`` is ready
* Create a Constraint named ``must-have-resources`` that targets the ``gatekeeper-demo`` namespace
* Test the policy by attempting to create a pod without resource limits

Constraint Requirements:
* Name: ``must-have-resources``
* Kind: ``K8sRequiredResources``
* Target namespace: ``gatekeeper-demo``
* Parameters: ``limits: true``

## Solution

**Step 1 — Verify Gatekeeper pods are running**
```
kubectl get pods -n gatekeeper-system
# All pods should be Running and Ready
```

**Step 2 — Verify the ConstraintTemplate exists and is ready**
```
kubectl get constrainttemplate k8srequiredresources
kubectl describe constrainttemplate k8srequiredresources
```

**Step 3 — Create the Constraint**
```
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: must-have-resources
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - gatekeeper-demo
  parameters:
    limits: true
EOF
```
Refer document: https://kubernetes.io/blog/2019/08/06/opa-gatekeeper-policy-and-governance-for-kubernetes/


**Step 4 — Test policy — create pod WITHOUT resource limits (should FAIL)**
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-no-limits
  namespace: gatekeeper-demo
spec:
  containers:
  - name: test
    image: nginx:alpine
EOF
# Expected: admission webhook denied — resource limits required
```

Note: Why apiVersion: ``constraints.gatekeeper.sh/v1beta1``? The Constraint's apiVersion is always ``constraints.gatekeeper.sh/v1beta1`` and the ``kind`` must exactly match the ``ConstraintTemplate`` name in PascalCase — ``k8srequiredresources`` →`` K8sRequiredResources``.

ConstraintTemplate vs Constraint: The ConstraintTemplate defines the policy logic (Rego code), while the Constraint defines where and how to apply that policy. Think of ConstraintTemplate as a class and Constraint as an instance of that class.
