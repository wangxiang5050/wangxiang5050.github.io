---
layout: post
title:  "阿里云CSI插件安装"
date:   2020-07-08 17:50:00 +0800
categories: CSI
tags:
    - ali
    - CSI
    - kubernetes
---
## Disk CSI_Plugin安装过程 ##

### Step 1: Create CSI disk-plugin ###

环境变量的ACCESS_KEY_ID, ACCESS_KEY_SECRET使用的是阿里云子账户的的AccessKey ID, Access Key Secret。由于不晓得具体使用什么权限，目前该用户是AdministratorAccess权限。  

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1beta1
kind: CSIDriver
metadata:
  name: diskplugin.csi.alibabacloud.com
spec:
  attachRequired: false
---
kind: DaemonSet
apiVersion: apps/v1beta2
metadata:
  name: csi-diskplugin
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: csi-diskplugin
  template:
    metadata:
      labels:
        app: csi-diskplugin
    spec:
      tolerations:
      - operator: "Exists"
      priorityClassName: system-node-critical
      serviceAccount: admin-user
      hostNetwork: true
      hostPID: true
      containers:
        - name: driver-registrar
          image: registry.cn-hangzhou.aliyuncs.com/acs/csi-node-driver-registrar:v1.1.0
          imagePullPolicy: Always
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "rm -rf /registration/diskplugin.csi.alibabacloud.com /registration/diskplugin.csi.alibabacloud.com-reg.sock"]
          args:
            - "--v=5"
            - "--csi-address=/var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com/csi.sock"
            - "--kubelet-registration-path=/var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com/csi.sock"
          env:
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: pods-mount-dir
              mountPath: /var/lib/kubelet/
            - name: registration-dir
              mountPath: /registration
        - name: csi-diskplugin
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          image: registry.cn-hangzhou.aliyuncs.com/acs/csi-plugin:v1.14.8.32-c77e277b-aliyun
          imagePullPolicy: "Always"
          args:
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--v=5"
            - "--driver=diskplugin.csi.alibabacloud.com"
          env:
            - name: CSI_ENDPOINT
              value: unix://var/lib/kubelet/plugins/diskplugin.csi.alibabacloud.com/csi.sock
            - name: ACCESS_KEY_ID
              value: ""
            - name: ACCESS_KEY_SECRET
              value: ""
            - name: MAX_VOLUMES_PERNODE
              value: "15"
            - name: DISK_TAGED_BY_PLUGIN
              value: "true"
          volumeMounts:
            - name: pods-mount-dir
              mountPath: /var/lib/kubelet
              mountPropagation: "Bidirectional"
            - mountPath: /dev
              mountPropagation: "HostToContainer"
              name: host-dev
            - mountPath: /var/log/
              name: host-log
            - name: etc
              mountPath: /host/etc
      volumes:
        - name: registration-dir
          hostPath:
            path: /var/lib/kubelet/plugins_registry
            type: DirectoryOrCreate
        - name: pods-mount-dir
          hostPath:
            path: /var/lib/kubelet
            type: Directory
        - name: host-dev
          hostPath:
            path: /dev
        - name: host-log
          hostPath:
            path: /var/log/
        - name: etc
          hostPath:
            path: /etc
  updateStrategy:
    type: RollingUpdate
EOF
```

### Step 2: Create CSI disk-provisioner ###

```shell
cat <<EOF | kubectl apply -f -
kind: Service
apiVersion: v1
metadata:
  name: csi-disk-provisioner
  namespace: kube-system
  labels:
    app: csi-disk-provisioner
spec:
  selector:
    app: csi-disk-provisioner
  ports:
    - name: dummy
      port: 12345

---
kind: StatefulSet
apiVersion: apps/v1beta1
metadata:
  name: csi-disk-provisioner
  namespace: kube-system
spec:
  serviceName: "csi-disk-provisioner"
  replicas: 1
  template:
    metadata:
      labels:
        app: csi-disk-provisioner
    spec:
      tolerations:
      - operator: "Exists"
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists
      priorityClassName: system-node-critical
      serviceAccount: admin-user
      hostNetwork: true
      containers:
        - name: csi-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/acs/csi-provisioner:v1.2.2-aliyun
          args:
            - "--provisioner=diskplugin.csi.alibabacloud.com"
            - "--csi-address=$(ADDRESS)"
            - "--feature-gates=Topology=True"
            - "--volume-name-prefix=pv-disk"
            - "--v=5"
          env:
            - name: ADDRESS
              value: /socketDir/csi.sock
          imagePullPolicy: "Always"
          volumeMounts:
            - name: disk-provisioner-dir
              mountPath: /socketDir
        - name: csi-diskplugin
          image: registry.cn-hangzhou.aliyuncs.com/acs/csi-plugin:v1.14.8.32-c77e277b-aliyun
          imagePullPolicy: "Always"
          args :
            - "--endpoint=$(CSI_ENDPOINT)"
            - "--v=5"
            - "--driver=diskplugin.csi.alibabacloud.com"
          env:
            - name: CSI_ENDPOINT
              value: unix://socketDir/csi.sock
            - name: ACCESS_KEY_ID
              value: ""
            - name: ACCESS_KEY_SECRET
              value: ""
            - name: MAX_VOLUMES_PERNODE
              value: "15"
          volumeMounts:
            - mountPath: /var/log/
              name: host-log
            - mountPath: /socketDir/
              name: disk-provisioner-dir
            - name: etc
              mountPath: /host/etc
      volumes:
        - name: disk-provisioner-dir
          emptyDir: {}
        - name: host-log
          hostPath:
            path: /var/log/
        - name: etc
          hostPath:
            path: /etc
  updateStrategy:
    type: RollingUpdate
EOF
```

### Step 3: Create StorageClass ###

使用下面SC，会创建一块SSD云盘，付费类型为按量付费，云盘容量在pvc spec.resources.requests.storage字段指定。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: csi-disk
provisioner: diskplugin.csi.alibabacloud.com
parameters:
    zoneId: cn-hangzhou-i
    regionId: cn-hangzhou
    fsType: ext4
    type: cloud_ssd
    readOnly: "false"
reclaimPolicy: Delete
```

### Step 4: Check Status of CSI plugin ###

```
[root@node1 alibaba-cloud-csi-driver]# kubectl get pods | grep csi
[root@node1 alibaba-cloud-csi-driver]# kubectl get pods -n kube-system | grep csi
csi-disk-provisioner-0                          2/2     Running     0          8d
csi-diskplugin-4qmcn                            2/2     Running     0          8d
csi-diskplugin-6cgch                            2/2     Running     0          8d
csi-diskplugin-76tsm                            2/2     Running     0          8d
csi-diskplugin-7rshs                            2/2     Running     0          8d
csi-diskplugin-8h7bt                            2/2     Running     0          8d
csi-diskplugin-b25rn                            2/2     Running     0          8d
csi-diskplugin-bc5bt                            2/2     Running     0          8d
csi-diskplugin-bmzdk                            2/2     Running     0          8d
csi-diskplugin-fjksw                            2/2     Running     0          8d
csi-diskplugin-jrpmp                            2/2     Running     0          8d
csi-diskplugin-jrxml                            2/2     Running     0          8d
csi-diskplugin-lqp6k                            2/2     Running     2          8d
csi-diskplugin-lr2q5                            2/2     Running     0          8d
csi-diskplugin-wff49                            2/2     Running     2          8d
csi-diskplugin-xsszs                            2/2     Running     0          8d
csi-diskplugin-xv7sr                            2/2     Running     0          8d
csi-diskplugin-zmjzx                            2/2     Running     0          8d
```

### Step 5: Create PVC & Deployment ###

在pod启动时，完成自动挂载。当Deployment实例个数变为0后，云盘变为待挂载状态。当PVC删除后，云盘也会自动删除。  
可以看到disk-pvc会购买并挂载一块容量为20GiB大小的磁盘。

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: disk-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: csi-disk
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-disk
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        volumeMounts:
          - name: disk-pvc
            mountPath: "/data"
      volumes:
        - name: disk-pvc
          persistentVolumeClaim:
            claimName: disk-pvc
```

## 参考文档 ##

alibaba-cloud-csi-driver github: https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver/blob/master/docs/disk.md  
parameters API文档地址: https://www.alibabacloud.com/help/zh/doc-detail/25513.htm?spm=a2c63.p38356.879954.17.5de7135eQF5ScP#doc-api-Ecs-CreateDisk  

