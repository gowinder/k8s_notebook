# helm install postpresql

## install 

```shell
helm install postgresql bitnami/postgresql \
--set postgresUser=go,postgresPassword=xxxxxx,postgresDatabase=postgres \
--set metrics.enabled=true \
--set postgresqlDatabase=prometheusdb \
--set persistence.enabled=true \
--set persistence.storageClass=nfs-client \
--set persistence.size=20Gi 

```

## local dns in cluster

`postgresql.cattle-logging.svc.cluster.local`

## get password

```shell
kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode

export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgresql-password}" | base64 --)
```

## test connect

```shell
kubectl run postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:11.10.0-debian-10-r79 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql -U postgres -d prometheusdb -p 5432
```

## outside cluster connect

```shell
kubectl port-forward --namespace cattle-logging svc/postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d prometheusdb -p 5432

```

## 权限

注意，挂载文件夹应该属于 1001:1001,参见 ：`https://github.com/bitnami/bitnami-docker-postgresql#configuration-file`