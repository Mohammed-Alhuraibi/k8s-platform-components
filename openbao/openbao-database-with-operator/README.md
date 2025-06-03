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
kubectl exec -ti openbao-0 -n openbao -- bao operator unseal <KEY-1>
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
# Enable Database usage

```bash
bao secrets enable database
```
# Apply the test Mysql pv-pvc file then the mysql-deployment
```bash
kubectl apply -f pvc-pv.yaml
kubectl apply -f mysql-deployment.yaml
```

# in the mysql database pod create user for openbao and grant him permissions as follow
```bash
mysql -h 127.0.0.1 -u root -p <DEFINED_IN_DEPOLYMENT_YAML_FILE>
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 13
Server version: 8.0.42 MySQL Community Server - GPL
```
```bash
mysql> CREATE USER IF NOT EXISTS 'openbaouser'@'%' IDENTIFIED BY 'openbaopass';
Query OK, 0 rows affected (0.04 sec)

mysql> GRANT SELECT, CREATE USER, ALTER, DROP ON *.* TO 'openbaouser'@'%';
Query OK, 0 rows affected (0.01 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.01 sec)
```
# Check user creation
```bash
mysql> SELECT user, host FROM mysql.user WHERE user = 'openbaouser';
+-------------+------+
| user        | host |
+-------------+------+
| openbaouser | %    |
+-------------+------+
1 row in set (0.01 sec)

mysql> SHOW GRANTS FOR 'openbaouser'@'%';
+--------------------------------------------------------------------+
| Grants for openbaouser@%                                           |
+--------------------------------------------------------------------+
| GRANT SELECT, DROP, ALTER, CREATE USER ON *.* TO `openbaouser`@`%` |
+--------------------------------------------------------------------+
1 row in set (0.01 sec)

```

# Now Inside the bao configure the mysql connection with new user and password that created above
```bash
bao write database/config/my-mysql-database \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(mysql.default.svc.cluster.local:3306)/" \
    allowed_roles="my-role" \
    username="openbaouser" \
    password="openbaopass"

```

# Now create the role that will be responsible for using the `my-mysql-database` connection an create new user per request
```bash
$ bao write database/roles/my-role \
    db_name=my-mysql-database \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}'; \
                       GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"

```
# Time to install the OpenBao (Vault) Secrets Operator
```bash
$ helm repo add hashicorp https://helm.releases.hashicorp.com
$ helm search repo hashicorp/vault-secrets-operator
NAME                                  CHART VERSION   APP VERSION DESCRIPTION
hashicorp/vault-secrets-operator      0.10.0          0.10.0      Official Vault Secrets Operator Chart
```
```bash
$ helm install vault-secrets-operator hashicorp/vault-secrets-operator \
    --version 0.10.0 \
    --namespace vault-secrets-operator \
    --create-namespace

```
# Enable secret kv
```bash

bao secrets enable -path=secret kv-v2


```
# Write the creds into bao secrets
```bash
bao kv put secret/db-creds \
    username="my_app_user" \
    password="S3cr3tP@ssw0rd"
```
# Create a policy that allows reading that KV path db-cred-policy.hcl
```bash
# db-cred-policy.hcl

# Allow read access to the static DB‐credential KV secret
path "secret/data/db-creds" {
  capabilities = ["read"]
}

# (Optional) If you also need list permissions, e.g. to enumerate child keys:
# path "secret/metadata/db-creds" {
#   capabilities = ["read", "list"]
# }

```
# Write the policy
```bash

bao policy write db-cred-policy db-cred-policy.hcl


```
# Create a Kubernetes auth role
#### Here vault-secrets-operator-controller-manager  is the serviceAccount your operator uses (check your operator’s Deployment or Helm values) and db-cred-policy is a Vault policy granting access to the DB secrets (you must create that policy separately)
```bash

bao write auth/kubernetes/role/vso-role \
    bound_service_account_names=vault-secrets-operator-controller-manager \
    bound_service_account_namespaces=vault-secrets-operator \
    policies=db-cred-policy \
    ttl=2h
```

# Time to configure the operator create Vault connection: This VaultConnection tells the operator where to find OpenBao
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: openbao-connection
  namespace: vault-secrets-operator
spec:
  address: "http://openbao.openbao.svc.cluster.local:8200"

```
# Here vaultConnectionRef refers to the above VaultConnection resource
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: openbao-auth
  namespace: vault-secrets-operator
spec:
  vaultConnectionRef: openbao-connection
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: vso-role
    serviceAccount: vault-secrets-operator-controller-manager

```
# Vault Static Secret to get the db-creds from openbao and store them in a secret
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: mydb-credentials
  namespace: vault-secrets-operator
spec:
  vaultAuthRef: openbao-auth
  mount: secret # KV v1 mount path (typically 'secret')
  type: kv-v2 # Important: specify kv-v1
  path: db-creds # Full path is 'secret/db-creds'
  destination:
    create: true
    name: mydb-root-secret # This will be the name of the K8s Secret created

```
