# Vault Kubernetes Authentication Setup for MariaDB

This comprehensive guide walks you through setting up HashiCorp Vault's Kubernetes authentication backend to enable your MariaDB pod in the `mariadb-prod` namespace to securely retrieve static secrets from Vault.

## Prerequisites

Before starting, ensure you have:
- OpenBao and Vault Secrets Operator installed and running in your Kubernetes cluster
- Administrative access to both Kubernetes and Vault/OpenBao
- The `bao` CLI tool available

## Architecture Overview

This setup creates a secure authentication flow where:
1. MariaDB pod authenticates to Vault using its Kubernetes ServiceAccount
2. Vault validates the ServiceAccount token with the Kubernetes API
3. Upon successful authentication, MariaDB gains access to retrieve its secrets

---

## Step 1: Configure Kubernetes Authentication Backend

### Enable Kubernetes Auth Method

```bash
bao auth enable kubernetes
```

### Configure Kubernetes Authentication

Set up Vault to communicate with your Kubernetes cluster:

```bash
bao write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://172.16.16.100:6443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### Configure Token Reviewer ServiceAccount

**Important**: The `token_reviewer_jwt` parameter uses the token of the ServiceAccount running in the same namespace as OpenBao. In this setup, we're using the `default` ServiceAccount in the `openbao` namespace. We need to create a token for this ServiceAccount using a Secret, then grant it Token Reviewer permissions via ClusterRoleBinding so it can review tokens coming from other ServiceAccounts.

The token reviewer requires proper permissions to validate ServiceAccount tokens. Create the necessary resources:

```yaml
# openbao-default-sa-clusterolebinding.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: default
  namespace: openbao
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openbao-default-token-reviewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: default
  namespace: openbao
```

Apply the configuration:
```bash
kubectl apply -f openbao-default-sa-clusterolebinding.yaml
```

---

## Step 2: Create Vault Policy and Role

### Define Access Policy

Create a policy that grants read access to MariaDB secrets:

```hcl
# mariadb-read.hcl
path "secret/data/mariadb/prod" {
  capabilities = ["read"]
}
```

Apply the policy:
```bash
bao policy write mariadb-read mariadb-read.hcl
```

### Create Kubernetes Authentication Role

Link the Kubernetes ServiceAccount to the Vault policy:

```bash
bao write auth/kubernetes/role/mariadb-role \
  bound_service_account_names="mariadb-sa" \
  bound_service_account_namespaces="mariadb-prod" \
  policies="mariadb-read" \
  ttl="24h"
```

---

## Step 3: Store MariaDB Credentials in Vault

### Write Secrets to Vault

Store your MariaDB credentials in Vault's KV v2 engine:

```bash
bao secrets enable -path=secret -version=2 kv


bao kv put secret/mariadb/prod \
  mariadb-root-password="root_password" \
  mariadb-password="mariadb_password" \
  mariadb-replication-password="replication_password"
```

### Verify Secret Storage

Confirm the secrets were stored correctly:
```bash
bao kv get secret/mariadb/prod
```

---

## Step 4: Create MariaDB ServiceAccount

Create the ServiceAccount that will be used by your MariaDB pods:

```yaml
# mariadb-sa-secret.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mariadb-sa
  namespace: mariadb-prod
---
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-sa
  namespace: mariadb-prod
  annotations:
    kubernetes.io/service-account.name: "mariadb-sa"
type: kubernetes.io/service-account-token
```

Apply the ServiceAccount configuration:
```bash
kubectl apply -f mariadb-sa-secret.yaml
```

---

## Step 5: Configure Vault Secrets Operator

### Prepare CA Bundle Secret

The Vault connection requires the CA certificate for secure communication. Copy it to the target namespace:

```bash
# Export the CA bundle from the openbao namespace
kubectl get secret openbao-ca-bundle -n openbao -o yaml > openbao-ca-bundle.yaml

# Modify for the target namespace and clean up metadata
sed -i \
  -e 's/namespace: openbao/namespace: mariadb-prod/' \
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

### Create Vault Resources

Configure the Vault connection, authentication, and static secret resources:

```yaml
# vault-connection-auth-staticsecret.yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: mariadb-vault-connection
  namespace: mariadb-prod
spec:
  address: https://openbao.openbao.svc.cluster.local:8200
  caCertSecretRef: openbao-ca-bundle
  skipTLSVerify: false
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: mariadb-vault-auth
  namespace: mariadb-prod
spec:
  vaultConnectionRef: mariadb-vault-connection
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: mariadb-role
    serviceAccount: mariadb-sa
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: mariadb-secret-staticsecret
  namespace: mariadb-prod
spec:
  vaultAuthRef: mariadb-vault-auth
  mount: secret
  path: mariadb/prod
  type: kv-v2
  destination:
    create: true
    name: mariadb-k8s-secret
```

Apply the Vault configuration:
```bash
kubectl apply -f vault-connection-auth-staticsecret.yaml
```

---

## Verification

After completing the setup, verify that everything is working correctly:

### Check Secret Creation

Confirm that the Kubernetes secret was created successfully:
```bash
kubectl get secret mariadb-k8s-secret -n mariadb-prod
kubectl describe secret mariadb-k8s-secret -n mariadb-prod
```

### Verify Secret Content

Check that the secret contains the expected keys:
```bash
kubectl get secret mariadb-k8s-secret -n mariadb-prod -o yaml
```

### Check Vault Secrets Operator Status

Monitor the status of your Vault resources:
```bash
kubectl get vaultstaticsecret -n mariadb-prod
kubectl describe vaultstaticsecret mariadb-secret-staticsecret -n mariadb-prod
```

---

## Troubleshooting

### Common Issues

1. **Authentication failures**: Verify that the ServiceAccount has the correct permissions and exists in the expected namespace
2. **Secret not created**: Check the Vault Secrets Operator logs for detailed error messages
3. **CA certificate issues**: Ensure the CA bundle secret exists in both source and target namespaces
4. **Network connectivity**: Verify that pods in the `mariadb-prod` namespace can reach the Vault service

### Debug Commands

```bash
# Check Vault Secrets Operator logs
kubectl logs -l app.kubernetes.io/name=vault-secrets-operator -n vault-secrets-operator-system

# Verify ServiceAccount token
kubectl get secret mariadb-sa -n mariadb-prod -o jsonpath='{.data.token}' | base64 -d

# Test Vault connectivity from within the cluster
kubectl run debug-pod --image=alpine/curl --rm -it -- sh
# From within the pod:
curl -k https://openbao.openbao.svc.cluster.local:8200/v1/sys/health
```

---

## Security Considerations

- **Least Privilege**: The policy grants only read access to the specific secret path needed
- **Token TTL**: Authentication tokens expire after 24 hours, requiring periodic renewal
- **TLS Security**: All communication with Vault uses TLS encryption
- **Namespace Isolation**: ServiceAccounts are bound to specific namespaces for security

## Next Steps

Once this setup is complete, you can:
1. Configure your MariaDB deployment to use the `mariadb-k8s-secret`
2. Set up similar authentication patterns for other applications
3. Implement secret rotation using Vault's dynamic secrets capabilities
4. Monitor secret access through Vault's audit logs
