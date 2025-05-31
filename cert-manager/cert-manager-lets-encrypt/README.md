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

# Create the secret for cloudflare API token
```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token=<YOUR_CLOUDFLARE_API_TOKEN>
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: cloudflare-api-token
  namespace: cert-manager
data:
  api-token: <YOUR_CLOUDFLARE_API_TOKEN_IN_BASE64>
```

# ClusterIssuer as Lets Encrypt Issuer with DNS challenge for cloudflare
```bash
kubectl apply -f clusterissuer-lets-encrypt.yaml
```
```yaml

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory # Use for production

    # server: https://acme-staging-v02.api.letsencrypt.org/directory # Use staging for testing

    email: <your-email@example.com>
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - dns01:
          cloudflare:
            email: <your-cloudflare-account-email>
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones:
            - <your-domain.com> # e.g., example.com

```
