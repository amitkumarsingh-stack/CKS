**Question 1:** Solve this question on: ``ssh cks3477``
You're asked to investigate a possible permission escape using the pre-defined context. The context authenticates as user restricted which has only limited permissions and shouldn't be able to read Secret values.

1. Switch to the restricted context with:

   ``k config use-context restricted@infra-prod``

2. Try to find the password-key values of the Secrets ``secret1``, ``secret2`` and ``secret3`` in Namespace restricted using context ``restricted@infra-prod``

3. Write the decoded plaintext values into files ``/opt/course/12/secret1``, ``/opt/course/12/secret2`` and ``/opt/course/12/secret3`` on ``cks3477``

Switch back to the default context with:

   ``k config use-context kubernetes-admin@kubernetes``

## Solution

**Key Concept Before Starting**
The question says the restricted user shouldn't be able to read Secrets — but it's asking you to try anyway and find if there's a permission escape. This means:

* The user may not be able to do ``kubectl get secret -o yaml`` directly
* But there may be other ways to access secret values through misconfigured RBAC
* Common escape routes: listing pods, exec into pods, reading env vars, reading mounted volumes

**Step 1 — SSH to node and switch to restricted context**
```
ssh cks3477
sudo -i

# Switch to restricted context
kubectl config use-context restricted@infra-prod

# Verify current context
kubectl config current-context
# Expected: restricted@infra-prod
```

**Step 2 — Investigate what the restricted user CAN do**
```
# Check what permissions this user has
kubectl auth can-i --list -n restricted
```
Output will show something like:
```
Resources          Verbs
secrets            []         ← cannot get/list secrets directly
pods               [get list] ← but CAN list pods
pods/exec          [create]   ← and CAN exec into pods!
```
This is the permission escape — the user can't read secrets directly but CAN exec into pods that have secrets mounted!

For more details refer YouTube video: https://www.youtube.com/watch?v=fcLR38yUUTA&list=PLyJzBek6WsDpeFd8BUHx4OCtuv_A5zQW-&index=13