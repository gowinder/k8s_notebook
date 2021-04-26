# helm install memcached

## install

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install my-release bitnami/memcached \
--set persistence.enabled=true \
--set persistence.storageClass=nfs-client \
--set persistence.size=10Gi \
--set metrics.enabled=true

```