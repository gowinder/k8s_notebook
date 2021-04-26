# rancher 2.5 logging

## use helm

```shell
helm repo add banzaicloud-stable https://kubernetes-charts.banzaicloud.com
helm repo update
helm install logging banzaicloud-stable/logging-operator-logging -f logging.yaml
```

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: defaultlogging
spec:
  fluentd:
    bufferStorageVolume:
      pvc:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          volumeMode: Filesystem
  fluentbit: {}
  controlNamespace: default
```

## open login

Cluster Explore-> Charts -> Logging

Edit yaml

```yaml
additionalLoggingSources:
  aks:
    enabled: false
  eks:
    enabled: false
  gke:
    enabled: false
  k3s:
    container_engine: systemd
    enabled: true
  rke:
    enabled: false
    fluentbit:
      log_level: info
      mem_buffer_limit: 5MB
  rke2:
    enabled: false
affinity: {}
annotations: {}
createCustomResource: false
disablePvc: false # use pvc
fluentbit_tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/controlplane
    value: 'true'
  - effect: NoExecute
    key: node-role.kubernetes.io/etcd
    value: 'true'
fullnameOverride: ''
global:
  cattle:
    systemDefaultRegistry: ''
http:
  port: 8080
  service:
    annotations: {}
    clusterIP: None
    labels: {}
    type: ClusterIP
image:
  pullPolicy: IfNotPresent
  repository: rancher/banzaicloud-logging-operator
  tag: 3.8.2
imagePullSecrets: []
images:
  config_reloader:
    repository: rancher/jimmidyson-configmap-reload
    tag: v0.2.2
  fluentbit:
    repository: rancher/fluent-fluent-bit
    tag: 1.6.4
  fluentbit_debug:
    repository: rancher/fluent-fluent-bit
    tag: 1.6.4-debug
  fluentd:
    repository: rancher/banzaicloud-fluentd
    tag: v1.11.5-alpine-1
monitoring:
  serviceMonitor:
    enabled: false
nameOverride: ''
namespaceOverride: ''
nodeSelector:
  kubernetes.io/os: linux
podSecurityContext: {}
priorityClassName: {}
rbac:
  enabled: true
  psp:
    enabled: false
replicaCount: 1
resources: {}
securityContext: {}
tolerations:
  - effect: NoSchedule
    key: cattle.io/os
    operator: Equal
    value: linux

# add spec
sped:
  fluentd:
    disablePvc: false
    bufferStorageVolume:
      pvc:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          volumeMode: Filesystem
```

## change fluentd buffer size

in rancher, open "Explore CLuster" -> "Installed Apps" -> "rancher-logging" -> "Secret - rancher-logging-fluentd" ->
"Edit Config" 
edit `cluster.conf`, add `total_limit_size 3GB` in:

```xml
<buffer>
 @type file
 ...
```