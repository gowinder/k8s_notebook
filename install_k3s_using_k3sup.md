# install k3s using k3s up

## install k3sup binary

```shell
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/

k3sup --help

```

## install k3s master

```shell
./k3sup install --local --k3s-extra-args '--no-deploy traefik'
```

note: `--k3s-extra-args '--no-deploy traefik'`, 目前K8s用的是1.7版本的traefik，需要自己部署2.0版本，就得加上这个参数，注意不要去掉`servicelb`了，这个是`loadbalance`用的

## install k3s worker

```shell
./k3sup join --ip 192.168.1.111 --server-ip 192.168.1.110 --user go --print-command
```

## 可选：修改host上的dns指向内网dns

`https://www.tecmint.com/set-permanent-dns-nameservers-in-ubuntu-debian/`

及 `https://askubuntu.com/questions/1128536/how-to-make-persistent-changes-to-etc-resolv-conf-ubuntu-18-10`
关键是要`sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf`,
然后删除pod让他更新 `kubectl delete pods -n kube-system -l k8s-app=kube-dns`
