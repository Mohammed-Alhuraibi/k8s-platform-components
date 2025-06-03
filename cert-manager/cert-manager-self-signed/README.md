
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
  --set prometheus.enabled=true   # Example: disabling prometheus using a Helm parameter
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
kubectl apply -f clusterissuer-self-signed.yaml
```
## Now time to generate a certificate
```bash
k apply -f certificate-self-signed.yaml
```
