**Question 1:**
The ``secure-web`` namespace contains a web application deployment ``secure-app`` exposed by a service of the same name.
Create an ingress resource named secure-ingress with the following security requirements:
1. Route traffic for host ``secure-app.company.com`` to the backend service on path ``/``
2. Enable TLS using the existing secret ``web-tls`` in the secure-web namespace
3. Configure the ingress to:
   * Force SSL redirect (HTTP to HTTPS)
   * Use the ``nginx`` ingress class
   * Add the annotation ``nginx.ingress.kubernetes.io/ssl-passthrough: "false"``
4. Ensure the ingress only accepts HTTPS traffic

Verify the ingress is working with TLS termination by testing through the ingress-nginx controller's NodePort.

Note: The ingress-nginx controller uses NodePorts for external access. Use the correct HTTPS NodePort assigned to the ingress-nginx service.

## Solution

**Step 1: Create the ingress resource with TLS configuration and security annotations:**
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: secure-web
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "false"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure-app.company.com
    secretName: web-tls
  rules:
  - host: secure-app.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-app
            port:
              number: 80
```
Refer documents
1. https://kubernetes.io/docs/concepts/services-networking/ingress/#tls
2. For annotations: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/

**Step 2: Verfication**
Ensure that ingress pods are healthy. Check the ingress status and TLS configuration using the following commands:
```
kubectl get ingress secure-ingress -n secure-web
kubectl describe ingress secure-ingress -n secure-web
```

**Step 3: Test Ingress**
First, find the NodePort assigned to the ingress controller for HTTPS:
```
kubectl get svc ingress-nginx-controller -n ingress-nginx
```
Excepted
```
# The output will show the actual NodePorts assigned, for example:
# NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
# ingress-nginx-controller   NodePort   10.96.123.456   <none>        80:30080/TCP,443:30443/TCP   5m
```

HTTPS Test: This should succeed.
```
curl -k -H "Host: secure-app.company.com" https://localhost:30443/
```
Expected output:
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

HTTP Test: This should redirect to HTTPS (308 redirect).
```
curl -k -H "Host: secure-app.company.com" http://localhost:30080/
```
Expected: Should show a 308 redirect to HTTPS.
