# install mysql

## using rancher app chart

注意资源，太低的话mysql会因为处理不过来连接导致被监控重启，建议内存1G+

yaml

```yaml
---
  defaultImage: true
  image: "ranchercharts/mysql"
  imageTag: "5.7.14" 
  busybox: 
    image: "ranchercharts/busybox"
    tag: "1.29.3"
  mysqlDatabase: "admin"
  mysqlUser: "admin"
  mysqlPassword: "xxxxxx"
  persistence: 
    enabled: true
    size: "20Gi"
    storageClass: "nfs-client"
    existingClaim: ""
  service: 
    port: "3306"
    type: "ClusterIP"
    nodePort: ""
  metrics: 
    enabled: true
    serviceMonitor: 
      enabled: true
  configurationFiles:
  mysql.cnf: |-
    [mysqld]
    skip-grant-tables
  resources:
    requests:
      memory: 2048Mi
      cpu: 2000m
```

**Note**

* `skip-grant-tables` 需要加到`[mysqld]`中去，不然root登录会被拒

### ingressroutetcp 开户外部访问

#### 修改 traefik entrypoints

见  `install_traefik_2.0.md`

#### 新建 `mysql.yaml`的 ingressroute文件

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  name: mysql
spec:
  entryPoints:
    - mysql
  routes:
    - match: HostSNI(`*`)
      kind: service
      services:
        - name: mysql
          namespace: default
          port: 3306
```

然后执行 `k apply -f mysql.yaml`