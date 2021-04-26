# efk manual install

## elasticsearch

* 编辑节点 host 机器，`/etc/security/limits.conf`
增加 `elasticsearch  -  nofile  65535`
如果是`ubuntu`，在`/etc/pam.d/su`中去掉注释
`# session    required   pam_limits.so`
* `/etc/sysctl.d/local.conf` 增加一行 `vm.max_map_count=262144` 

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install elastichsearch bitnami/elasticsearch \
--set global.kibanaEnabled=true \
--render-subchart-notes \
--set master.persistence.enabled=true \
--set master.persistence.storageClass=nfs-client \
--set master.persistence.size=8Gi \
--set data.replicas=2 \
--set data.persistence.enabled=true \
--set data.persistence.storageClass=nfs-client \
--set data.persistence.size=10Gi 
```

## fluentd

```shell
helm install fluentd bitnami/fluentd \

```

## kibana

```shell
helm install kibana bitnami/kibana \
--set elasticsearch.hosts={"elastichsearch-elasticsearch-master.svc.default"} \
--set elasticsearch.port=9200 \
--set persistence.enabled=true \
--set presistence.storageClass=nfs-client \
--set persistence.size=10Gi \
--set ingress.enabled=true \
--set ingress.hostname=kibana.vm-k3s-0.home

```