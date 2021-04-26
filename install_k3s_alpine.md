# install k3s in alpine

## note

* 不要用`linux container`，少了很多东西，使用`vm`

## 准备 alpine

### 关闭swap

```shell
swapoff -a
```

然后编辑 `/etc/fstab`，屏掉`swap`分区

### 打开cgroup

`echo "cgroup /sys/fs/cgroup cgroup defaults 0 0" >> /etc/fstab`

编辑 `cgconfig.conf`

```shell
cat > /etc/cgconfig.conf <<EOF
mount {
  cpuacct = /cgroup/cpuacct;
  memory = /cgroup/memory;
  devices = /cgroup/devices;
  freezer = /cgroup/freezer;
  net_cls = /cgroup/net_cls;
  blkio = /cgroup/blkio;
  cpuset = /cgroup/cpuset;
  cpu = /cgroup/cpu;
}
EOF
```

编辑 `/etc/update-extlinux.conf`，找到`default_kernel_opts`,在内容后加上:
`default_kernel_opts="...  cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory"`
或者使用命令替换 `sed -i 's/default_kernel_opts="pax_nouderef quiet rootfstype=ext4"/default_kernel_opts="pax_nouderef quiet rootfstype=ext4 cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory"/g' /etc/update-extlinux.conf`

运行：

```shell
update-extlinux
reboot
```

### 安装 cni-plugins

编辑 `/etc/apk/repository`, 增加一行`@testing http://mirrors.tuna.tsinghua.edu.cn/alpine/edge/testing`
注意打开 `testing`源及`edge`源
然后运行 `apk update && add cni-plugins@testing`

### 安装 iptables

`apk add iptables`

### 准备 `K3S_NODE_NAME`

第台主机都需要在安装时定义 `K3S_NODE_NAME`,而且不能相同

## 安装 k3s

### 最新方法，使用k3sup

```shell
# master, no traefik but with serverlb
sudo ./k3sup install --local --k3s-extra-args '--no-deploy traefik'
# node
./k3sup join --ip 192.168.1.111 --server-ip 192.168.1.110 --user go --print-command
./k3sup join --ip 192.168.1.112 --server-ip 192.168.1.110 --user go --print-command
```

traefik 见文档 : `install_traefik_2.0.md`

---

### 下载脚本

```shell
curl -sfL https://get.k3s.io > install_k3s.sh
chmod +x install_k3s.sh
./install_k3s.sh
```
或者直接使用：
`curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--cluster-cidr=10.244.0.0/16 --kube-apiserver-arg=feature-gates=RemoveSelfLink=false" sh -`
如果报 `cgroup mount`错误，`busy`等，检查上面`cgroup`是在`default_kernel_opts`否设置正确
如果不行，还可以在`/etc/rc.conf` 中设置
 `rc_cggroup_memory_use_hierarchy="yes"`

注意，这一行，如果 后面nfs无法分配空间，就需要这个，但是听说1.21后这个功能就移除了
`--kube-apiserver-arg=feature-gates=RemoveSelfLink=false`

### 拿到`token`

`cat /var/lib/rancher/k3s/server/node-token`

### 设置`KUBECONFIG`

`export KUBECONFIG=/etc/rancher/k3s/k3s.yaml`,最好加入到`~/.zshrc`中去

### 安装其它节点

将其中 `K3S_URL`, `K3S_TOKEN`换成`server`的地址和上面得到的`token`
`curl -sfL https://get.k3s.io  agent | INSTALL_K3S_EXEC="agent --server https://192.168.1.131:6443 --token $K3S_TOKEN" sh -`

设置成`worker`: `kubectl label node ${node} node-role.kubernetes.io/worker=worker`

#### 其它节点安装问题

##### `invalid token format`

token不正确，可以将`K3S_TOKEN`,`K3S_URL`,`K3S_NODE_NAME` 写入到`~/.zshrc`等中去，然后再操作

##### `Failed to connect to proxy" error="x509: certificate signed by unknown authority`

查看 `/var/log/k3s-agent.log`日志，出现这个错误
说明配置错了，运行`curl -sfL https://get.k3s.io  agent` 没有加上`agent`参数，变成了`master`节点，所以会报错。

### 卸载 k3s

主节点
`/usr/local/bin/k3s-uninstall.sh`

node:
`/usr/local/bin/k3s-agent-uninstall.sh`

## 安装 rancher

### 安装 helm

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

```

### 配置chart

```shell
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```

### 启动 rancher

#### 增加namespace

```shell
kubectl create namespace cattle-system
```

#### 安装 `cert-manager`

我们使用自动生成证书，其它的要参考文档：`https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/#5-install-cert-manager`

```shell
# Install the CustomResourceDefinition resources separately
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.crds.yaml

# **Important:**
# If you are running Kubernetes v1.15 or below, you
# will need to add the `--validate=false` flag to your
# kubectl apply command, or else you will receive a
# validation error relating to the
# x-kubernetes-preserve-unknown-fields field in
# cert-manager’s CustomResourceDefinition resources.
# This is a benign error and occurs due to the way kubectl
# performs resource validation.

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.0.4
```

如果 `install` 报错： `Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version?timeout=32s": dial tcp [::1]:8080: connect: connection refused`，说明没有配置 `KUBECONFIG`,按照上面设置。

测试一下: `kubectl get pods --namespace cert-manager`

### 安装 rancher

将 $HOSTNAME 换成load-balance的地址，此次是内网，那就直接是master地址
```shell

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.vm-k8s-0.home
```

#### 如果报错

如果报错： `Error: chart requires kubeVersion: < 1.20.0-0 which is incompatible with Kubernetes v1.20.0+k3s2`

是`Chart.yaml`里面规定了`KubeVersion`，格式和`k3s`不匹配，解决方法：
```shell
helm pull rancher-stable/rancher --untar
vi rancher/Chart.yaml
```
屏掉`#kubeVersion: < 1.20.0-0`
再次运行 

```shell
helm install rancher rancher \
  --namespace cattle-system \
  --set hostname=$HOST
```

其中第二个 `rancher` 为下载解压后的目录

#### 等待安装完成

`kubectl -n cattle-system rollout status deploy/rancher`
等待3个pods全部启动，检查下：
`kubectl -n cattle-system get deploy rancher`
应该得到这样结果：

```shell
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
rancher   3/3     3            3           13m
```

#### 端口转发

默认是没有转发的，我们访问不了rancher的ui
查看端口：
`kubectl get services -n cattle-system` 默认应该是 80,443/TCP

##### 使用`port-forward`：
`kubectl port-forward svc/rancher 7443:443 --address 0.0.0.0 -n cattle-system`
注意`--address 0.0.0.0`,如果不指定就是`localhost`了.

##### 使用`extenalIP`:
`kubectl patch svc -n cattle-system rancher -p '{"spec":{"externalIPs":["192.168.1.110"]}}'`
将 `192.168.1.110` 换成任何一个可以访问的节点ip


### nfs 存储

#### 注意

如果无法分配volume，看看nfs-client的pod日志，是否有`class "nfs-client": unexpected error getting claim reference: selfLink was empty, can't make reference`
那就需要`kube-apiserver`增加启动参数，在`/etc/init.d/k3s`中增加一行`--kube-apiserver-arg=feature-gates=RemoveSelfLink=false`，注意最好 不加引号

#### 过程

新建一个 pv-nfs.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 200G
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.1.151
    # Exported path of your NFS server
    path: "/volume1/docker/rancher"
```

或者：

```shell
helm install nfs-client stable/nfs-client-provisioner \
--set nfs.server=192.168.1.151 \
--set nfs.path=/volume1/docker/rancher \
--set nfs.mountOptions="{nolock,proto=tcp,rw,sec=sys}"
```

r720

```shell
helm install nfs-client stable/nfs-client-provisioner \
--set nfs.server=192.168.1.101 \
--set nfs.path=/tank/docker/rancher \
--set nfs.mountOptions="{nolock,proto=tcp,rw,sec=sys}"
```

```shell
helm upgrade nfs-client stable/nfs-client-provisioner \
--set nfs.server=192.168.1.101 \
--set nfs.path=/tank/docker/rancher \
--set nfs.mountOptions="{nolock,proto=tcp,rw,sec=sys}"
```

#### test nfs claim

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "nfs-client"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

`kubectl create -f testpvc.yaml`

#### 如果没有分配成功

`kubectl describe pvc test-claim`

如果出现`permission denied`, 在`nfs server`上，将要共享的目录直接`chomd 0777 -R /path`

#### 如果容器没有权限 

如mysql报错`chown: changing ownership of '/var/lib/mysql/': Operation not permitted`
是nfs服务的参数要调整，改成：`rw,async,no_wdelay,crossmnt,insecure,no_root_squash,insecure_locks,sec=sys`

```shell
helm upgrade nfs-client stable/nfs-client-provisioner \
--set nfs.server=192.168.1.101 \
--set nfs.path=/tank/docker/rancher \
--set nfs.mountOptions="{rw,async,no_wdelay,crossmnt,insecure,no_root_squash,insecure_locks,sec=sys}"
```

### rancher cli

#### 生成token
点击右上用户图标，选择api,生成一个新的，注意`scope`要选择`none`
然后拿到 `Bearer Token`

#### login

`./rancher login https://alpine-vm-k3s-master/v3 -t token-cv5fm:65pl4xrz7bhvznqmw7djk2pbmnckmh8v686qmlp9r6brpgstphc6nq
`

### traefik 2.0

ref: `https://medium.com/@fache.loic/k3s-traefik-2-9b4646393a1c`

`https://www.infvie.com/ops-notes/kubernetes-ingress-traefik.html`

`https://www.qikqiak.com/post/expose-redis-by-traefik2/`

deploy 文件:

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - traefik.containo.us
    resources:
      - middlewares
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - ingressroutes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - ingressroutetcps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - traefik.containo.us
    resources:
      - tlsoptions
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system

---

apiVersion: v1
kind: ServiceAccount
metadata:
 namespace: kube-system
 name: traefik-ingress-controller

---

kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: traefik
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  selector:
    matchLabels:
      k8s-app: traefik-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      tolerations:
      - operator: "Exists"
      nodeSelector:
        kubernetes.io/hostname: vm-k8s-0
      containers:
      - image: traefik:v2.0
        name: traefik-ingress-lb
        ports:
        - name: web
          containerPort: 80
          hostPort: 80
        - name: websecure
          containerPort: 443
          hostPort: 443
        - name: admin
          containerPort: 8080
        args:
        - --entrypoints.web.Address=:80
        - --entrypoints.websecure.Address=:443
        - --api.insecure=true
        - --providers.kubernetescrd
        - --api
        - --api.dashboard=true
        - --accesslog

---

kind: Service
apiVersion: v1
metadata:
  name: traefik
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 8080
      name: admin

```

注意：
*  `entryPoints`是 `traefik_deployment.yaml`中定义的(此文件参见：`https://gist.github.com/lfache/fca258bfde73066fea66f3888eb45622/raw/139bfce35511e5b61a89549baa4c769336bc89b1/gistfile1.txt`)，一个端口相当于一个 `engtryPoints`，对于tcp的服务，一个端口开一个
* `namespace`一定要是 `kube-system`

使用IngressRoute方式

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard-route
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`traefik.vm-k8s-0.home`)
    kind: Rule
    services:
      - name: traefik
        port: 8080
```

使用IngressRoute方式暴露grafana, 注意`namespace`要填写`monitor`是`prometheus-stack`所在的`namespace`

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: prometheus-stack-grafana-route
  namespace: monitor
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`grafana.vm-k8s-0.home`)
    kind: Rule
    services:
      - name: prometheus-stack-grafana
        port: 80
```

kibana

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: efk-kibana-route
  namespace: efk
spec:
  entryPoints:
  - web
  routes:
  - match: Host(`kibana.vm-k8s-0.home`)
    kind: Rule
    services:
      - name: kibana-http
        port: 80
```

### traefik dashboard (despered)

编辑 `/var/lib/rancher/k3s/server/manifests/traefik.yaml`


```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: traefik
  namespace: kube-system
spec:
  chart: https://%{KUBERNETES_API}%/static/charts/traefik-1.81.0.tgz
  set:
    rbac.enabled: "true"
    ssl.enabled: "true"
    metrics.prometheus.enabled: "true"
    kubernetes.ingressEndpoint.useDefaultPublishedService: "true"
    image: "rancher/library-traefik"
    dashboard.enabled: "true" # <-- 增加
    dashboard.domain: "traefik.internal"  # <-- 增加域名

```

然后运行`kubectl apply -f /var/lib/rancher/k3s/server/manifests/traefik.yaml`

#### metallb

install `kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml`

make a new file `metallb.yaml`

```yaml
apiVersion: v1  
kind: ConfigMap  
metadata:  
  namespace: metallb-system  
  name: config  
data:  
  config: |  
    address-pools:  
    - name: address-pool-1  
      protocol: layer2  
      addresses:  
      - 192.168.1.115-192.168.1.119
```

`192.168.1.115-192.168.1.119` 这个IP段不能被占用

然后编辑一个新文件， traefik-dashboard.yaml

```yaml
apiVersion: traefik.containo.us/v1alpha1  
kind: IngressRoute  
metadata:  
  name: dashboardsecure  
  namespace: traefik  
spec:  
  entryPoints:  
    - websecure  
  routes:  
    - match: Host(`traefik.vm-k8s-0.home`)  
      kind: Rule  
      services:  
        - name: api@internal  
          kind: TraefikService  
      middlewares:  
      - name: traefik-sso@kubernetescrd  
  tls:  
    certResolver: cloudflare
```


### note

#### helm 设置数组

```shell
--set test={x,y,z}
--set "test={x,y,z}"

Result YAML:

test:
  - x
  - y
  - z

```