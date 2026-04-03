# 1. Generation of Self Signed SSL/TLS certificate
```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=yourdomain.com/O=yourdomain.com"
```
# 2. create a Kubernetes secret for the certificate
```bash 
    kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```
# 3. Define Ingress rules
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - yourdomain.com
    secretName: tls-secret
  rules:
  - host: yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```
# 4. Apply the changes
```bash
    kubectl apply -f ingress.yaml
```
# 5. Test the Changes
```bash
    curl -v http://<load-balancer-IP>/your-app-endpoint -H "Host: yourdomain.com" -k
```
- The above request will provide a 308 redirect response
```bash
> GET / HTTP/1.1
> Host: my-self-signed-domain.com
> User-Agent: curl/7.88.1
> Accept: */*
> 
< HTTP/1.1 308 Permanent Redirect
```
- try using the ```https``` as the protocol
```bash
    curl -v https://<load-balancer-IP>/your-app-endpoint -H "Host: yourdomain.com" -k
```
- this time the request is served with ```200``` response
```bash
* ALPN: offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):


* using HTTP/2
* h2h3 [:method: GET]
* h2h3 [:path: /]
* h2h3 [:scheme: https]
* h2h3 [:authority: yourdomain.com]
* h2h3 [user-agent: curl/7.88.1]
* h2h3 [accept: */*]
* Using Stream ID: 1 (easy handle 0x11c812800)
> GET / HTTP/2
> Host: yourdomain.com
> user-agent: curl/7.88.1
> accept: */*
> 
< HTTP/2 200 
```
