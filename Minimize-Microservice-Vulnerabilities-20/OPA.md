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

