# Create a user which has the privileges to create delete .. etc users in the mariadb databse so this user will be used by the openbao role to genrate temporary users used by applications so they can access the mariadb database.
## inside the mariadb
```bash
CREATE USER 'openbaouser'@'%' IDENTIFIED BY 'openbaopass';
GRANT CREATE USER, DROP USER, SELECT, LOCK TABLES, EVENT, TRIGGER ON *.* TO 'openbaouser'@'%';
FLUSH PRIVILEGES;
```

## check user exists
```bash
SELECT User FROM mysql.user WHERE User = 'openbaouser';
```


## inside the openba
```bash
bao secrets enable database

```
## Database config
```bash
bao write database/config/my-mariadb-database \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mariadb-primary.mariadb-prod.svc.cluster.local:3306)/" \
    allowed_roles="app1-role" \
    username="openbaouser" \
    password="openbaopass" \
    tls_ca=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

```
## Plociy
```bash
# /tmp/app1-policy.hcl
path "auth/kubernetes/login" {
  capabilities = ["create", "read"]
}

path "database/creds/app1-role" {
  capabilities = ["read"]
}
```
```bash
bao policy write app1-policy /tmp/app1-policy.hcl
```
## Role for database
```bash
bao write database/roles/app1-role \
    db_name=my-mariadb-database \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"

```

## Role for kubernetes auth
```bash
# This creates the auth/kubernetes role in Vault itself:
bao write auth/kubernetes/role/app1-role \
  bound_service_account_names="app1-sa" \
  bound_service_account_namespaces="app1" \
  policies="app1-policy" \
  ttl="1h"

```

# Cofigure the openbao to use the creds of the user openbao created above.
## Create the namespace app1 and its service account and secret
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: app1
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app1-sa
  namespace: app1
---
apiVersion: v1
kind: Secret
metadata:
  name: app1-sa
  namespace: app1
  annotations:
    # Tells K8s to populate this Secret with the 'app1-sa' ServiceAccount’s token
    kubernetes.io/service-account.name: app1-sa
type: kubernetes.io/service-account-token

```

```bash
k apply -f app1-ns-sa-secret.yaml
```

## copy the openbao-ca-bundle.crt to the app1 namespace
```bash

# Export the CA bundle from the openbao namespace
kubectl get secret openbao-ca-bundle -n openbao -o yaml > openbao-ca-bundle.yaml

# Modify for the target namespace and clean up metadata
sed -i \
  -e 's/namespace: openbao/namespace: app1/' \
  -e '/^\s*resourceVersion:/d' \
  -e '/^\s*uid:/d' \
  -e '/^\s*creationTimestamp:/d' \
  -e '/^\s*selfLink:/d' \
  -e '/^\s*managedFields:/,+5d' \
  -e '/^status:/,$d' \
  openbao-ca-bundle.yaml

# Apply to the target namespace
kubectl apply -f openbao-ca-bundle.yaml
```

# using dynamic secrets pull dynaimc user creds from openbao

```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: app1-vault-connection
  namespace: app1
spec:
  address: https://openbao.openbao.svc.cluster.local:8200
  # caCertSecretRef: openbao-ca-bundle
  caCertSecretRef: openbao-ca-bundle
  skipTLSVerify: true
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: app1-vault-auth
  namespace: app1
spec:
  vaultConnectionRef: app1-vault-connection # <- Reference to VaultConnection CR
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: app1-role
    serviceAccount: app1-sa
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultDynamicSecret
metadata:
  name: mydb-dynamic-secret
  namespace: app1
spec:
  # Reference to your VaultAuth object (e.g. openbao-auth) that holds
  # the kube-auth/k8s-auth configuration for Vault
  vaultAuthRef: app1-vault-auth

  mount: database

  path: creds/app1-role



  # Where to write the resulting username/password in Kubernetes
  destination:
    # Will create or update a Secret named "mydb-app-credentials"
    create: true
    name: mydb-app-credentials

```

```bash
kubectl apply -f vault-connection-auth-dynamicsecret.yaml
```
