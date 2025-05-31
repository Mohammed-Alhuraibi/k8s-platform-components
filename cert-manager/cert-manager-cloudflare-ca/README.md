# Add the helm repo of cert-manager
```bash
helm repo add jetstack https://charts.jetstack.io --force-update
```

# install cert-manager using helm
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \
  --set crds.enabled=true \
  --set prometheus.enabled=true \  # Example: disabling prometheus using a Helm parameter
```

# For cloudflare install its crds, rbac, and manifest(controller, which will process Certificate Requests created by cert-manager.)
```bash
kubectl apply -f crds
kubectl apply -f rbac
kubectl apply -f manifests
```

# check pod origin-ca-issuer
```bash
kubectl get -n origin-ca-issuer pod
```

# Usage example create the secret for storing the cloudflare api token
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cfapi-token
  namespace: default
type: Opaque
data:
  key: <cloudflare-api-token>
```
```bash
kubectl apply -f cfapi-token.yaml
```

# Then create an OriginIssuer referencing the secret created above.
```yaml
apiVersion: cert-manager.k8s.cloudflare.com/v1
kind: OriginIssuer
metadata:
  name: prod-issuer
  namespace: default
spec:
  requestType: OriginECC
  auth:
    tokenRef:
      name: cfapi-token
      key: key
```
```bash
kubectl apply -f issuer.yaml
```

# The status conditions of the OriginIssuer resource will be updated once the Origin CA Issuer is ready.
```bash
kubectl get originissuer.cert-manager.k8s.cloudflare.com prod-issuer -o json | jq .status.conditions

[
  {
    "lastTransitionTime": "2020-10-07T00:05:00Z",
    "message": "OriginIssuer verified an ready to sign certificates",
    "reason": "Verified",
    "status": "True",
    "type": "Ready"
  }
]

```
# Creating our first certificate
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  # The secret name where cert-manager should store the signed certificate
  secretName: example-com-tls
  dnsNames:
    - example.com
  # Duration of the certificate
  duration: 168h
  # Renew a day before the certificate expiration
  renewBefore: 24h
  # Reference the Origin CA Issuer you created above, which must be in the same namespace.
  issuerRef:
    group: cert-manager.k8s.cloudflare.com
    kind: OriginIssuer
    name: prod-issuer
```
```bash
kubectl apply -f certificate.yaml
```

# IMPORTED THE BELOW GATEWAY EXAMPLE ISN'T TESTED YET

# GATEWAY
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  namespace: default
  annotations:
    cert-manager.io/issuer: prod-issuer
    cert-manager.io/issuer-kind: OriginIssuer
    cert-manager.io/issuer-group: cert-manager.k8s.cloudflare.com
spec:
  gatewayClassName: your-gatewayclass-name # Replace with your GatewayClass
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: example.com
      tls:
        mode: Terminate
        certificateRefs:
          - name: example-tls
            kind: Secret
      allowedRoutes:
        namespaces:
          from: Same

```
```bash
kubectl apply -f gateway.yaml
```

# HTTRout
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-route
  namespace: default
spec:
  parentRefs:
    - name: example-gateway
      kind: Gateway
  hostnames:
    - example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: examplesvc
          port: 80
```


# Self Signed Certificate EXAMPLE

## Generate certificate key and self-signed certificate
```bash
 openssl genrsa -out ca.key 4096
 openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \
   -subj "/CN=my-ca"
```
```bash

kubectl create secret tls my-ca-secret --cert=ca.crt --key=ca.key -n cert-manager --dry-run=client  -o yaml > ca-secret.yaml

```
## Apply the secret
```bash
kubectl apply -f ca-secret.yaml
```

## Apply the clusterissuer
```bash
kubectl apply -f clusterissuer-cert-manger.yaml
```
## Now time to generate a certificate
```bash
k apply -f certificate-self-signed.yaml
```
