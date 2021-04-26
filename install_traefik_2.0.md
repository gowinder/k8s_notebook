# install traefik 2.0

## 参见：
`https://github.com/gowinder/k3s-traefik-v2-kubernetes-crd`

clone后顺着来

## expose app using ingresroute

注意：
* `namespace` 填写在 `services` 下面， `IngressRoute`中的`metadata`只用填写这个`ingressroute`的名称就可以了

### rancher

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: rancher
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`rancher.vm-k8s-0.home`)
      kind: Rule
      services:
        - name: rancher
          namespace: cattle-system
          port: 80
  tls:
    passthrough: true
```

## ingress tcp

* modify 005-deployment.yaml
  * contianers.args 中增加 - --entrypoints.mysql.address=:3306
  * ports中增加 
  ```yaml
  - name: mysql
              containerPort: 3306
              hostPort: 13306
              protocol: TCP
  ```
* 执行 `k apply -f 005-deployment.yaml`
* 编辑mysql.yaml 的 ingressroutetcp文件

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

`k apply -f mysql.yaml`
