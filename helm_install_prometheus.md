# helm install prometheus

## ref

`https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack`

## install repo

```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## show value

`helm show values prometheus-community/kube-prometheus-stack`

ref: `https://github.com/prometheus-community/helm-charts`



## install

```shell

helm install prometheus-stack prometheus-community/kube-prometheus-stack \
--namespace monitor \
--set prometheusOperator.admissionWebhooks.certManager.enabled=true \
--set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
--set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
--set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=nfs-client \
--set grafana.ingress.enabled=true \
--set grafana.ingress.hosts="{grafana.vm-k8s-0.home}"


```