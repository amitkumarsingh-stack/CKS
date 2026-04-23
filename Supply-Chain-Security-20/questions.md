**Question 1**: Solve this question on: ``ssh cks3477``
There are four Kubernetes server binaries located at ``/opt/course/6/binaries`` on ``cks3477``. You're provided with the following verified sha512 values for these:

kube-apiserver
``f417c0555bc0167355589dd1afe23be9bf909bf98312b1025f12015d1b58a1c62c9908c0067a7764fa35efdac7016a9efa8711a44425dd6692906a7c283f032c``

kube-controller-manager
``60100cc725e91fe1a949e1b2d0474237844b5862556e25c2c655a33b0a8225855ec5ee22fa4927e6c46a60d43a7c4403a27268f96fbb726307d1608b44f38a60``

kube-proxy
``52f9d8ad045f8eee1d689619ef8ceef2d86d50c75a6a332653240d7ba5b2a114aca056d9e513984ade24358c9662714973c1960c62a5cb37dd375631c8a614c6``

kubelet
``4be40f2440619e990897cf956c32800dc96c2c983bf64519854a3309fa5aa21827991559f9c44595098e27e6f2ee4d64a3fdec6baba8a177881f20e3ec61e26c``

Delete those binaries that don't match the sha512 values above.

**Step 1 — Switch context and SSH**
```
ssh cks3477
sudo -i
```

**Step 2 — Navigate to the binaries directory**
```
cd /opt/course/6/binaries
ls -la
# Should show: kube-apiserver, kube-controller-manager, kube-proxy, kubelet
```

**Step 3 — Generate sha512 hashes for all binaries**
There are 2 ways you can solve this
```
sha512sum kube-apiserver
```
But the problem with this approach is that you will have to verify all the characters which is time consuming

The other aproach is
```
echo "f417c0555bc0167355589dd1afe23be9bf909bf98312b1025f12015d1b58a1c62c9908c0067a7764fa35efdac7016a9efa8711a44425dd6692906a7c283f032c kube-apiserver" sha512sum --check
# excepted value is "kube-apiserver: OK"
```

Repate the steps for all the files and delete the ones which don't match.

------------------------------

**Question 2**:
A vulnerable deployment has been identified in the ``security-scanning`` namespace. Your task is to utilize the pre-configured KubeLinter configuration located at ``/root/kube-linter-config.yaml`` to identify and rectify all security issues in this deployment.

Tasks:
* Scan the vulnerable deployment using the provided KubeLinter configuration.
* Identify all security violations present in the deployment.
* Address and resolve the security issues identified in the deployment.
* Verify that the revised deployment successfully passes all security checks.

The vulnerable deployment can be found at ``/tmp/init/manifests/vulnerable-deployment.yaml``, and KubeLinter is pre-installed with the configuration file already set up.

## Solution

**Step 1 — View the KubeLinter config**
```
cat /root/kube-linter-config.yaml
```

**Step 2 — Run KubeLinter scan**
```
kubelinter lint /tmp/.init/manifests/vulnerable-deployment.yaml --config /root/kube-linter-config.yaml
```
Excepeted Result
```
/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) The container "app" is using an invalid container image, "nginx:latest". Please use images that are not blocked by the `BlockList` criteria : [".*:(latest)$" "^[^:]*$" "(.*/[^:]+)$"] (check: latest-tag, remediation: Use a container image with a specific tag other than latest.)

/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) container "app" does not have a read-only root file system (check: no-read-only-root-fs, remediation: Set readOnlyRootFilesystem to true in the container securityContext.)

/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) container "app" is Privileged hence allows privilege escalation. (check: privilege-escalation-container, remediation: Ensure containers do not allow privilege escalation by setting allowPrivilegeEscalation=false, privileged=false and removing CAP_SYS_ADMIN capability. See https://kubernetes.io/docs/tasks/configure-pod-container/security-context/ for more details.)

/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) container "app" is privileged (check: privileged-container, remediation: Do not run your container as privileged unless it is required.)

/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) container "app" is not set to runAsNonRoot (check: run-as-non-root, remediation: Set runAsUser to a non-zero number and runAsNonRoot to true in your pod or container securityContext. Refer to https://kubernetes.io/docs/tasks/configure-pod-container/security-context/ for details.)

/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) container "app" has cpu request 0 (check: unset-cpu-requirements, remediation: Set CPU requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)

/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) container "app" has cpu limit 0 (check: unset-cpu-requirements, remediation: Set CPU requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)

/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) container "app" has memory request 0 (check: unset-memory-requirements, remediation: Set memory requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)

/tmp/.init/manifests/vulnerable-deployment.yaml: (object: security-scanning/insecure-app apps/v1, Kind=Deployment) container "app" has memory limit 0 (check: unset-memory-requirements, remediation: Set memory requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)
```

**Step 3 — Fix all common violations**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: insecure-app
  namespace: security-scanning
spec:
  replicas: 1
  selector:
    matchLabels:
      app: insecure-app
  template:
    metadata:
      labels: {}
    spec:
      containers:
      - name: app
        image: nginx:1.30                                             # Remove 'latest' and tag the version
        securityContext:
          privileged: false                                           # Disable privilege
          readOnlyRootFilesystem: true                                # readOnlyRootFilesystem to 'true'
          allowPrivilegeEscalation: false                             # allowPrivilegeEscalation: false
          runAsNonRoot: true                                          # No 'root' user
          runAsUser: 1000                                             # specific UID
          capabilities:
            drop:
            - ALL                                                       # Drop all Linux capabilities
        resources:                                                    # Resource Quota
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
```