# install openbao via helm
```bash

helm repo add openbao https://openbao.github.io/openbao-helm
helm install openbao openbao/openbao -n openbao --create-namespace

```
# init the openbao-0
```bash
kubectl exec -ti openbao-0 -n openbao -- bao operator init -key-shares=1 -key-threshold=1

```
# Save the output into a file for reference later
```bash
Unseal Key 1: MBFSDepD9E6whREc6Dj+k3pMaKJ6cCnCUWcySJQymObb

Initial Root Token: s.zJNwZlRrqISjyBHFMiEca6GF
```
# Unseal the openbao pod
```bash
kubectl exec -ti openbao-0 -n openbao -- bao operator unseal -key=<KEY-1>
```
# Login using the Root Token
```bash
kubectl exec -ti openbao-0 -n openbao -- /bin/sh
```

```bash
bao login
```
# Portforward the openbao pod
```bash
kubectl port-forward openbao-0 -n openbao 8200:8200 -n openbao
```

# Authenicate into Kubernetes from inside the openbao pod

```bash
bao auth enable kubernetes
```
```bash
bao write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://172.16.16.100:6443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```
### Check the UI or check the auth list by running
```bash
bao auth list
```

# After Kubernetes authenication we can use the Key/Value Secrets Engine so our
```bash
bao secrets enable -path=secret -version=2 kv
```

## Write secrets to the KV Secrets Engine
```bash
# Write a secret at path "secret/my-app/config"
bao kv put -mount=secret my-app/config username=admin password="s3cr3t"
```
## Confirm
```bash
bao kv get -mount=secret my-app/config
```
## Crate a policy file named  my-app-policy.hcl
```bash
vi /tmp/my-app-policy.hcl
```
```bash
path "secret/data/my-app/config" {
  capabilities = ["read"]
}
```
### Apply the policy
```bash
bao policy write my-app-policy /tmp/my-app-policy.hcl
```
### Create an OpenBao Kubernetes Auth Role
##### Note: you have to create the namespace and the service account before running this command
```bash
kubectl create ns my-app
kubectl create serviceaccount my-app-service-account -n my-app
```

```bash
bao write auth/kubernetes/role/my-app-role \
    bound_service_account_names=my-app-service-account \
    bound_service_account_namespaces=my-app \
    policies=my-app-policy \
    ttl=1h \
    max_ttl=24h \
    period=1h
```
# Label the namespace with openbao-injection=enabled so the bao injector can inject data into the sidecar container which will be automatically created
```bash
kubectl label namespace my-app openbao-injection=enabled
```
# Now Time to deploy a nginx deployement and inject the secret automatically
```bash
kubectl apply -f deployment-sidecar-example.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "my-app-role"
        vault.hashicorp.com/agent-inject-secret-db-config: "secret/data/my-app/config"
        vault.hashicorp.com/agent-inject-output-path: "/app/secrets"

    spec:
      serviceAccountName: my-app-service-account
      containers:
        - name: my-app
          image: nginx
          volumeMounts:
            - name: secrets
              mountPath: /app/secrets
              # mountPath: /vault/secrets
              readOnly: true
      volumes:
        - name: secrets
          emptyDir: {}

```
