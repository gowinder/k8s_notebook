# helm install gitea and drone

## gitea

### value file

#### 官方`chart`

```shell
helm repo add gitea-charts https://dl.gitea.io/charts/
helm install --values gitea_values.yaml \
 gitea gitea-charts/gitea
```

```yaml
replicaCount: 1

clusterDomain: cluster.local

image:
  repository: gitea/gitea
  tag: 1.13.1
  pullPolicy: Always

imagePullSecrets: []

service:
  http:
    type: ClusterIP
    port: 3000
    clusterIP: None
    #loadBalancerIP:
    #nodePort:
    annotations:
  ssh:
    type: ClusterIP
    port: 33322
    clusterIP: None
    #loadBalancerIP:
    #nodePort:
    #externalTrafficPolicy:
    externalIPs:
      - 192.168.1.110
    annotations:

ingress:
  enabled: false
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  hosts:
    - git.alpine-vm-k3s-master
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - git.example.com

resources:
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  limits:
    cpu: 1000m
    memory: 1024Mi
  requests:
    cpu: 500m
    memory: 512Mi

nodeSelector: {}

tolerations: []

affinity: {}

statefulset:
  env: []
    # - name: VARIABLE
    #   value: my-value
  terminationGracePeriodSeconds: 60

persistence:
  enabled: true
  storageClass: nfs-client
  # existingClaim: 
  size: 20Gi
  accessModes:
    - ReadWriteMany

gitea:
  metrics:
    enabled: true
  admin:
    username: gitea_admin
    password: xxxxxx
    email: "gowinder@hotmail.com"

  ldap:
    enabled: false
    #name: 
    #securityProtocol: 
    #host: 
    #port: 
    #userSearchBase: 
    #userFilter: 
    #adminFilter: 
    #emailAttribute: 
    #bindDn: 
    #bindPassword: 
    #usernameAttribute: 

  config:
    database:
      DB_TYPE: mysql
      HOST: mysql:3306
      NAME: gitea
      USER: root
      PASSWD: Gx7txKfkjS
      SCHEMA: gitea    
  #  APP_NAME: "Gitea: Git with a cup of tea"
  #  RUN_MODE: dev   
  #   
    server:
      SSH_PORT: 33322
      SSH_DOMAIN: gitea-ssh.vm-k8s-0.home
      DOMAIN: gitea-http.vm-k8s-0.home
  #
  #  security:
  #    PASSWORD_COMPLEXITY: spec

  podAnnotations: {}

  database:
    builtIn:
      postgresql:
        enabled: false
      mysql:
        enabled: false
      mariadb:
        enabled: false

  cache:
    builtIn:
      enabled: true

memcached:
  service:
    port: 11211

# postgresql:
#   global:
#     postgresql:
#       postgresqlDatabase: gitea
#       postgresqlUsername: gitea
#       postgresqlPassword: gitea
#       servicePort: 5432
#   persistence:
#     size: 10Gi

# mysql:
#   root:
#     password: gitea
#   db:
#     user: gitea
#     password: gitea
#     name: gitea
#   service:
#     port: 3306
#   persistence:
#     size: 10Gi

# mariadb:
#   auth:
#     database: gitea
#     username: gitea
#     password: gitea
#     rootPassword: gitea
#   primary:
#     service:
#       port: 3306
#     persistence:
#       size: 10Gi

```

##### ingressroute

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: gitea-http
spec:
  entryPoints:
    - web
    - websecure
  routes:
    - match: Host(`gitea-http.vm-k8s-0.home`)
      kind: Rule
      services:
        - name: gitea-http
          namespace: default
          port: 3000
  tls:
    passthrough: true
```

##### drone

直接使用 rancher app drone, 编辑yaml

```yaml
---
  defaultImage: true
  images: 
    server: 
      repository: "ranchercharts/drone-drone"
      tag: "1.2"
    agent: 
      repository: "ranchercharts/drone-agent"
      tag: "1.2"
    dind: 
      repository: "ranchercharts/library-docker"
      tag: "18.06.1-ce-dind"
  server: 
    host: "drone.vm-k8s-0.home"
    adminUser: "gitea_admin"
    protocol: "https"
    env: 
      DRONE_LOGS_DEBUG: "true"
      DRONE_DATABASE_DRIVER: "mysql"
      DRONE_DATABASE_DATASOURCE: "drone:password@tcp(mysql:3306)/drone?parseTime=true"
      DRONE_GITEA_CLIENT_ID: "e667ed13-e61b-4b4a-8ea5-a2ec8f673e96"
      DRONE_GITEA_CLIENT_SECRET: "5LTDp_UCYQ4FD59MVPWisqPDa_ghnOteUIp1AWohdnQ="
      DRONE_GITEA_SKIP_VERIFY: "true"
  sourceControl: 
    provider: "gitea"
    secret: "gitea"
    github: 
      clientID: "1"
      clientSecretValue: "1"
    gitlab: 
      clientID: ""
      server: ""
      clientSecretValue: ""
    gitea: 
      server: "https://gitea-http.vm-k8s-0.home"
    gogs: 
      server: ""
    bitbucketCloud: 
      clientID: ""
      clientSecretValue: ""
    bitbucketServer: 
      server: ""
      username: ""
  persistence: 
    enabled: true
    size: "20Gi"
    storageClass: "nfs-client"
    existingClaim: ""
  ingress: 
    enabled: false
    hosts: 
      - "xip.io"
  service: 
    type: "ClusterIP"
    nodePort: ""
  metrics: 
    prometheus: 
      enabled: true
```

###### 说明

- 如果是内网定义的域名，指定了自己的dns服务器，需要修改k8s所在的node 主机上的`/etc/resolve.conf`,参见 `install_k3s_using_k3sup.md`
- `DRONE_DATABASE_DRIVER` 数据 库名
- `DRONE_DATABASE_DATASOURCE` 数据库连接串，要自己改
- `DRONE_GITEA_CLIENT_ID` `DRONE_GITEA_CLIENT_SECRET` 需要在gitea中设置一个新的app,架设地址填 `http://drone.vm-k8s-0.home/login`
- `gitea.server` 结尾要加上 `?skip_verify=true`不然无法登录 ，会报： `We're sorry but Drone does not work properly without JavaScript enabled. Please enable it to continue.`



#### 非官方 

from `https://github.com/jfelten/gitea-helm-chart`
add repo `helm repo add keyporttech https://keyporttech.github.io/helm-charts/`



```shell
helm install --values gitea_values_kp.yaml \
 gitea --namespace gitea keyporttech/gitea
```

```yaml
images:
  gitea: gitea/gitea:1.12.4
  postgres: "postgres:11"
  memcached: "memcached:1.5.19-alpine"
  imagePullPolicy: Always
memcached:
  maxItemMemory: 64
  verbosity: v
  extendedOptions: modern
## ingress settings - Optional
ingress:
  enabled: true
  # sets .ini PROTOCOL value
  hosts:
    - "gitea.alpine-vm-k3s-master"

service:
  http:
    serviceType: ClusterIP
    port: 3000
    # nodePort: 30280
    # sometimes if is necesary to access through an external port i.e. http(s)://<dns-name>:<external-port>
    # externalPort: 8280
    # externalHost: git.example.com
    # some parts of gitea like the api use a redirect needs config if using non standard ports
    # httpAPIPort: 8280
    # if serviceType is LoadBalancer
    # loadBalancerIP: "192.168.0.100"
    # svc_annotations:
    #   service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    #   service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "a_subnet"
  ssh:
    serviceType: ClusterIP
    port: 22
    # nodePort: 30222
    # if serving on a different external port used for determining the ssh url in the gui
    # externalPort: 8022
    # externalHost: git.example.com
    # if serviceType is LoadBalancer
    # loadBalancerIP: "192.168.0.100"
    # svc_annotations:
    #   service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    #   service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "a_subnet"
## Configure resource requests and limits
## ref: http://kubernetes.io/docs/user-guide/compute-resources/
##
resources:
  gitea:
    requests:
      memory: 1000Mi
      cpu: 1000m
    limits:
      memory: 2Gi
      cpu: 2
  postgres:
    requests:
      memory: 200Mi
      cpu: 200m
    limits:
      memory: 2Gi
      cpu: 1
  memcached:
    requests:
      memory: 64Mi
      cpu: 50m
persistence:
  enabled: true
  # existingGiteaClaim: gitea-gitea
  # existingPostgresClaim: gitea-postgres
  giteaSize: 20Gi
#   postgresSize: 5Gi
  storageClass: nfs-client
  accessMode: ReadWriteMany
## addtional annotations for hte pvcs  uncommenting below will prevent helm from deleting the pvc when hte chart is deleted
  annotations:
    "helm.sh/resource-policy": keep

## if you want to mount a volume directly without using a storageClass or pvcs
#  directGiteaVolumeMount:
#    glusterfs:
#      endpoints: "192.168.1.1 192.168.1.2 192.168.1.3"
#      path: giteaData
#  directPostgresVolumeMount:
#    glusterfs:
#      endpoints: "192.168.1.1 192.168.1.2 192.168.1.3"
#      path: giteaPostgresData

# Connect to an external database
externalDB:
  dbUser: "mysql"
  dbPassword: "xxxxxxx"
  dbHost: "mysql" # or some external host
  dbPort: "3306"
  dbDatabase: "gitea"

# valid types: postgres, mysql, mssql, sqllite
# dbType: "postgres"
# useInPodPostgres: true
# inPodPostgres:
#   secret: postgresecrets
#   subPath: "postgresql-db"
#   dataMountPath: /var/lib/postgresql/data/pgdata
  # Create a database user
  # Default: postgres
  # postgresUser:
  # Default: random 10 character string
  # postgresPassword:

  # Inject postgresPassword via a volume mount instead of environment variable
#   usePasswordFile: false
  # Use Existing secret instead of creating one
  # It must have a postgres-password key containing the desired password
  # existingSecret: 'secret'

  # Create a database
  # Default: the postgres user
#   postgresDatabase: gitea
  # Specify initdb arguments, e.g. --data-checksums
  # ref: https://github.com/docker-library/docs/blob/master/postgres/content.md#postgres_initdb_args
  # ref: https://www.postgresql.org/docs/current/static/app-initdb.html
  # postgresInitdbArgs:

  ## Specify runtime config parameters as a dict, using camelCase, e.g.
  ## {"sharedBuffers": "500MB"}
  ## ref: https://www.postgresql.org/docs/current/static/runtime-config.html
  # postgresConfig:
# Node labels and tolerations for pod assignment
# ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector
# ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#taints-and-tolerations-beta-feature
nodeSelector: {}
tolerations: []
affinity: {}
## Annotations for the deployment and nodes.
deploymentAnnotations: {}
podAnnotations: {}
# In order to disable initial install screen you must have secretKey and disableInstaller=true
config:
  #  secretKey: define_your_secret
  disableInstaller: false
  offlineMode: false
  requireSignin: false
  disableRegistration: false
  openidSignin: true
  notifyMail: false
  mailer:
    enabled: false
    host: smtp.gmail.com
    port: 587
    tls: false
    from: ""
    user: ""
    passwd: ""
  metrics:
    enabled: true
    token: ""

# Here you can set extra settings from https://docs.gitea.io/en-us/config-cheat-sheet/
# The following is just an example
# extra_config: |-
#   [service]
#   ENABLE_USER_HEATMAP=false

```

##### change password

```shell
kubectl exec -n gitea -it gitea-gitea-5d67dcd798-n9vdb -c gitea -- bash
su git
gitea admin create-user --username go --admin --password xxxxxx --email gowinder@hotmail.com
```