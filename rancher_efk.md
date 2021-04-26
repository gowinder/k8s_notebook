# efk in rancher library

## FAQ

### elasticsearch max file descriptors [4096] for elasticsearch process is too low, increase to at least

`k edit pod elasticsearch-master-0 -n efk`
upgrade efk, 在`edit as yaml`中的 `elasticsearch`增加一个`initContainers`

```yaml
    extraInitContainers: |
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          runAsUser: 0
          privileged: true
```

在 `/etc/init.d/k3s`中增加一行
`rc_ulimit="-n 666666"` 如果是子节点，在`/etc/init.d/k3s-agent`中，改好了运行

* `master`: `rc-service k3s restart`
* `worker`: `rc-service k3s-agent restart`

完整 yaml

`replicas` 是数量，一个`node`一个数量

```yaml
  defaultImage: true
  elasticsearch: 
    image: "ranchercharts/elasticsearch-elasticsearch"
    imageTag: "7.3.0"
    replicas: "3"
    antiAffinity: "hard"
    persistence: 
      enabled: true
      storageClass: "nfs-client"
      size: "30Gi"
    # sysctlInitContainer:
    #   enabled: false
    # extraInitContainers: |
    #   - name: increase-fd-ulimit
    #     image: "ranchercharts/elasticsearch-elasticsearch:7.3.0"
    #     command: ["sh", "-c", "#!/bin/bash\n ulimit -n 65536 && ulimit -n && sysctl -w vm.max_map_count=262144"]
    #     securityContext:
    #       runAsUser: 0
    # #       privileged: true
    # ingress:
    #   enabled: true
    #   path: /
    #   hosts:
    #     - elastic.vm-k8s-0      
  kibana: 
    image: "ranchercharts/kibana-kibana"
    imageTag: "7.3.0"
    language: "en"
    service: 
      type: "ClusterIP"
    enableProxy: false
    ingress:
      enabled: false
      path: /
      hosts:
        - kibana.vm-k8s-0.home

  filebeat: 
    image: "ranchercharts/beats-filebeat"
    imageTag: "7.3.0"
    enabled: false
  metricbeat: 
    image: "ranchercharts/beats-metricbeat"
    imageTag: "7.3.0"
    kube-state-metrics: 
      image: 
        repository: "ranchercharts/coreos-kube-state-metrics"
        tag: "v1.7.2"
    enabled: true
```



### 删除PVC

```shell
k patch -n efk pvc efk/elasticsearch-master-elasticsearch-master-0 -p '{"metad
ata":{"finalizers":null}}'
k delete -n efk pvc elasticsearch-master-elasticsearch-master-0
k patch -n efk pvc elasticsearch-master-elasticsearch-master-1 -p '{"metad
ata":{"finalizers":null}}'
k delete -n efk pvc elasticsearch-master-elasticsearch-master-1
k patch -n efk pvc elasticsearch-master-elasticsearch-master-2 -p '{"metad
ata":{"finalizers":null}}'
k delete -n efk pvc elasticsearch-master-elasticsearch-master-2
```

## rancher 将日志发向`efk`

参考文章： `https://support.rancher.com/hc/en-us/articles/360041579232-Setting-up-logging-on-K8S-example#configure-log-forwarding-0-1`

注意里面`elasticsearch`的地址就可以填 `http://elasticsearch-master-headless.efk:9200`,或者将域名换成内网ip, 域名可以通过命令`kubectl get svc --all-namespaces`得到

## 安装 kibana k8s metribeat dashboard

```shell
kubectl get pods -n efk
kubectl exec -it pod/efk-metricbeat-xxxx -c metricbeat -n efk -- bash
metricbeat setup --dashboards -v
```

如果报错
， 
```shell
Exiting: error connecting to Kibana: fail to get the Kibana version: HTTP GET request to http://localhost:5601/api/status fails: fail to execute the HTTP GET request: Get http://localhost:5601/api/status: dial tcp [::1]:5601: connect: connection refused. Response:
```

编辑

```shell
k get configmap
k edit cm/efk-metricbeat-config
```

 在窗口中找到 ` metricbeat.yml: |`， 加上
 
 ```yaml
 setup.kibana:
    host: "efk-kibana:5601"
```

然后删除pod，会自动重建