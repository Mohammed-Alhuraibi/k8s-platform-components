# Add the bitnmai chart using helm
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

# install mariadb on the mariadb-prod namespace with overwritten values.yaml for production
```bash
helm install mariadb bitnami/mariadb \
  --namespace mariadb-prod \
  --create-namespace \
  -f values-prod.yaml
```

# If you modified the chart upgrade the helm realase by
```bash

helm upgrade --install mariadb bitnami/mariadb \
  --namespace mariadb-prod \
  --create-namespace \
  -f values-prod.yaml
```
