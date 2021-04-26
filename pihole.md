# pi-hole

## add repo

```shell
helm repo add mojo2600 https://mojo2600.github.io/pihole-kubernetes/
helm repo update
```

## launch

```shell
helm install pihole mojo2600/pihole \
--set persistentVolumeClaim.enabled=true \
--set DNS1="202.103.24.68" \
--set DNS2="202.103.44.150" \
--set doh.enabled=false \
--set ingress.enabled=true \
--set ingress.hosts="{pihole.vm-k8s-0.home}" 



```

### values.yaml

```yaml
DNS1: 202.103.24.68
DNS2: 202.103.44.150
persistentVolumeClaim:
  enabled: true
ingress:
  enabled: true
  hosts:
    - "pihole.vm-k8s-0.home" 
serviceTCP:
  loadBalancerIP: '192.168.1.110'
  type: LoadBalancer
serviceUDP:
  loadBalancerIP: '192.168.1.110'
  type: LoadBalancer
resources:
  limits:
    cpu: 1000m
    memory: 1024Mi
  requests:
    cpu: 500m
    memory: 512Mi
# If using in the real world, set up admin.existingSecret instead.
adminPassword: xxxxxx

```

`helm install pihole mojo2600/pihole -f pihole.yaml`

### 过滤地址 
在`group management`->`adlists`中增加
`https://anti-ad.net/domains.txt`

然后在 `Tools`->`Update Gravity`中增加
