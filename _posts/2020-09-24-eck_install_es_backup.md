# 使用ECK 安装Elasticsearch集群 #

本文介绍使用ECK安装Elasticsearch集群、Kibana以及Elasticsearch数据备份与备份清理等内容。

## 元信息

### 版本 ###

`6.8.8`

### 节点 ###

| 节点角色               | 数量 | CPU （request/limit） | 内存(request/limit) | Xms/Xmx | 存储  |
| ---------------------- | ---- | --------------------- | ------------------- | ------- | ----- |
| master eligible Node   | 3    | 0.5/2                 | 2Gi/2Gi             | 1g/1g   | 20Gi 高效云盘  |
| data node              | 3    | 4/4                   | 16Gi/16Gi           | 14g/14g | 100Gi 高效云盘 |
| cooridnating only node | 1    | 0.5/2                 | 2Gi/2Gi             | 1g/1g   | 20Gi 高效云盘  |

## Elasticsearch集群安装

### s3 secret

集群使用s3插件将snapshot备份至阿里云OSS中，其用到的access_key、secret_key存储在下面secret中。

ECK请使用1.2.1版本，低版本initContainer	elastic-internal-init-keystore可能会报错。

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  s3.client.default.access_key: ${access_key}
  s3.client.default.secret_key: ${secret_key}
kind: Secret
metadata:
  name: ali-credentials
  namespace: elasticsearch
type: Opaque
EOF
```

### Elasticsearch

Elasticsearch集群部署在其专用机器上，存储storageClass是cloud-efficiency-disk，为阿里云高效云盘。CSI插件安装、使用方式，以及storageClass的创建方式请参考[阿里云CSI插件安装](https://wangxiang5050.github.io/csi/2020/07/08/alibabaa-cloud-csi-driver/)

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-app
  namespace: elasticsearch
spec:
  http:
    service:
      metadata:
        creationTimestamp: null
    tls:
      selfSignedCertificate:
        disabled: true
  image: wangxiang5050/elasticsearch:6.8.8-s3
  nodeSets:
  - config:
      cluster.remote.connect: false
      node.data: false
      node.ingest: false
      node.master: true
      node.ml: false
      xpack.ml.enabled: true
      xpack.security.enabled: false
    count: 3
    name: master
    podTemplate:
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: dedicated
                  operator: In
                  values:
                  - elasticsearch
        containers:
        - env:
          - name: ES_JAVA_OPTS
            value: -Xms1g -Xmx1g
          name: elasticsearch
          resources:
            limits:
              cpu: 2
              memory: 2Gi
            requests:
              cpu: 0.5
              memory: 2Gi
        securityContext:
          fsGroup: 1000
          runAsGroup: 1000
          runAsUser: 1000
        tolerations:
        - effect: NoSchedule
          key: dedicated
          operator: Equal
          value: elasticsearch
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: cloud-efficiency-disk
  - config:
      cluster.remote.connect: false
      node.data: true
      node.ingest: false
      node.master: false
      node.ml: false
      xpack.security.enabled: false
    count: 3
    name: data
    podTemplate:
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: dedicated
                  operator: In
                  values:
                  - elasticsearch
        containers:
        - env:
          - name: ES_JAVA_OPTS
            value: -Xms14g -Xmx14g
          name: elasticsearch
          resources:
            limits:
              cpu: 4
              memory: 16Gi
            requests:
              cpu: 4
              memory: 16Gi
        securityContext:
          fsGroup: 1000
          runAsGroup: 1000
          runAsUser: 1000
        tolerations:
        - effect: NoSchedule
          key: dedicated
          operator: Equal
          value: elasticsearch
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 100Gi
        storageClassName: cloud-efficiency-disk
  - config:
      cluster.remote.connect: false
      node.data: false
      node.ingest: false
      node.master: false
      node.ml: false
      xpack.security.enabled: false
    count: 1
    name: cooridnating
    podTemplate:
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: dedicated
                  operator: In
                  values:
                  - elasticsearch
        containers:
        - env:
          - name: ES_JAVA_OPTS
            value: -Xms1g -Xmx1g
          name: elasticsearch
          resources:
            limits:
              cpu: 2
              memory: 2Gi
            requests:
              cpu: 0.5
              memory: 2Gi
        securityContext:
          fsGroup: 1000
          runAsGroup: 1000
          runAsUser: 1000
        tolerations:
        - effect: NoSchedule
          key: dedicated
          operator: Equal
          value: elasticsearch
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 20Gi
        storageClassName: cloud-efficiency-disk
  secureSettings:
  - secretName: ali-credentials
  version: 6.8.8
EOF
```

## Kibana安装 ##

```
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-app
  namespace: elasticsearch
spec:
  version: 6.8.8
  count: 1
  elasticsearchRef:
    name: elasticsearch-app
    namespace: elasticsearch
  podTemplate:
    spec:
      containers:
      - name: kibana
        image: kibana:6.8.8
  http:
    tls:
      selfSignedCertificate:
        disabled: true
EOF
```

## 数据备份 ##

### 创建repository

kibana Dev Tools页面执行如下命令，完成repository创建。

```
PUT _snapshot/my_s3_repository
{
  "type": "s3",
  "settings": {
    "bucket": "backup",
    "endpoint" : "oss-cn-hangzhou.aliyuncs.com",
    "client": "default"
  }
}
```

### 定时备份

使用以下CronJob每天对Elasticsearch中数据进行全量备份。

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: elasticsearch-app-backup
  namespace: elasticsearch
spec:
  schedule: "@daily"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: snapshotter
            image: centos:7
            command:
            - /bin/bash
            args:
            - -cex
            - 'curl -s -i -k -XPUT "http://elasticsearch-app-es-http:9200/_snapshot/my_s3_repository/%3Ces-app-%7Bnow%2Fd%7D%3E" | tee /dev/stderr | grep "200 OK"'
          restartPolicy: OnFailure
EOF
```

#### 备份测试

```
kubectl  delete -n elasticsearch job test-backup
kubectl create job test-backup --namespace elasticsearch --from=cronjob/elasticsearch-app-backup
```

### snapshot清理

清理名字中时间戳大于三天的snapshot。

#### values.yaml

```yaml
# Default values for elasticsearch-curator.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

cronjob:
  # At 01:00 every day
  schedule: "0 1 * * *"
  annotations: {}
  labels: {}
  concurrencyPolicy: ""
  failedJobsHistoryLimit: ""
  successfulJobsHistoryLimit: ""
  jobRestartPolicy: Never
  startingDeadlineSeconds: ""

pod:
  annotations: {}
  labels: {}

rbac:
  # Specifies whether RBAC should be enabled
  enabled: false

serviceAccount:
  # Specifies whether a ServiceAccount should be created
  create: true
  # The name of the ServiceAccount to use.
  # If not set and create is true, a name is generated using the fullname template
  name:


psp:
  # Specifies whether a podsecuritypolicy should be created
  create: false

image:
  repository: untergeek/curator
  tag: 5.7.6
  pullPolicy: IfNotPresent

hooks:
  install: false
  upgrade: false

# run curator in dry-run mode
dryrun: false

command: ["/curator/curator"]
env: {}

configMaps:
  # Delete indices older than 3 days
  action_file_yml: |-
    ---
    actions:
      1:
        action: delete_snapshots
        description: "Delete selected snapshots from 'repository'"
        options:
          repository: my_s3_repository
          retry_interval: 120
          retry_count: 3
        filters:
        - filtertype: pattern
          kind: prefix
          value: es-app
        - filtertype: age
          source: name
          direction: older
          timestring: '%Y.%m.%d'
          unit: days
          unit_count: 3
          field:
          stats_result:
          epoch:
          exclude: False
  # Having config_yaml WILL override the other config
  config_yml: |-
    ---
    client:
      hosts:
        - elasticsearch-app-es-http 
      port: 9200
resources: 
  limits:
   cpu: 100m
   memory: 128Mi
  requests:
   cpu: 100m
   memory: 128Mi

priorityClassName: ""
extraInitContainers: {}
securityContext:
  runAsUser: 16  # run as cron user instead of root
```

#### 安装命令

``` shell
helm install --name es-app-snapshot-cleaner --namespace elasticsearch  -f values.yaml stable/elasticsearch-curator
```

#### 测试snapshot删除

```shell
kubectl  delete -n elasticsearch job test-es-app-backup-clean
kubectl create job test-es-app-backup-clean --namespace elasticsearch --from=cronjob/es-app-snapshot-cleaner-elasticsearch-curator
```

## 集群删除 ##

```
kubectl delete elastic elasticsearch-app
```