---
layout: post
title:  "install gerrit on kubernetes"
date:   2020-07-22 15:30:00 +0800
categories: gerrit
tags:
    - gerrit
    - kubernetes
---

## 在K8S上安装gerrit，并向gogs同步

### 安装gerrit

yaml如下，kubectl apply -f后，开始安装。

P.S. 环境变量CANONICAL_WEB_URL必须以http://或https://开头。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: gerrit
  name: gerrit
  namespace: gerrit
spec:
  selector:
    matchLabels:
      run: gerrit
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: gerrit
    spec:
      containers:
      - env:
        - name: CANONICAL_WEB_URL
          value: http://gerrit.domainname.in
        image: harbor.domainname.in/domainname/gerrit:3.2.2
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 600
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 5
        name: gerrit
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /
            port: 8080
            scheme: HTTP
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 5
        volumeMounts:
        - mountPath: /var/gerrit/etc
          name: etc
        - mountPath: /var/gerrit/git
          name: git
        - mountPath: /var/gerrit/db
          name: db
        - mountPath: /var/gerrit/index
          name: index
        - mountPath: /var/gerrit/cache
          name: cache
        - mountPath: /var/gerrit/.ssh
          name: ssh
      dnsPolicy: ClusterFirst
      volumes:
      - name: etc
        persistentVolumeClaim:
          claimName: gerrit
      - name: git
        persistentVolumeClaim:
          claimName: gerrit-git
      - name: db
        persistentVolumeClaim:
          claimName: gerrit-db
      - name: index
        persistentVolumeClaim:
          claimName: gerrit-index
      - name: cache
        persistentVolumeClaim:
          claimName: gerrit-cache
      - name: ssh
        persistentVolumeClaim:
          claimName: gerrit-ssh
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: gerrit
  name: gerrit
  namespace: gerrit
spec:
  ports:
  - name: ssh
    port: 29418
    protocol: TCP
    targetPort: 29418
  - name: web
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: gerrit
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gerrit
  namespace: gerrit
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: package-dedicated
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gerrit-cache
  namespace: gerrit
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: package-dedicated
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gerrit-db
  namespace: gerrit
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: package-dedicated
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gerrit-git
  namespace: gerrit
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: package-dedicated
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gerrit-index
  namespace: gerrit
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: package-dedicated
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gerrit-ssh
  namespace: gerrit
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: package-dedicated
```

```
[root@node1 ~]# kubectl get -n gerrit pods
NAME                      READY   STATUS    RESTARTS   AGE
gerrit-6dbcbcc4bd-t79cg   1/1     Running   0          142m
```

安装成功！

### 代码同步

#### gerrit中新建仓库

仓库名为：gerrit-test

#### gogs中新建仓库

仓库名：wangxiang/gerrit-test

#### 生成ssh公钥、私钥

```shell
bash-4.4$ ssh-keygen -m PEM -t rsa -C "wangxiang@gmail.com"
```

#### 将上步中生成的公钥拷贝至gogs用户ssh秘钥中

```shell
bash-4.4$ cat ~/.ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCU7umh8m8lBBmAt7vKwOGDwzNs/DVmJiBOzRZYqCOxrKQSL6U2df7BzvjLX7QRtHkOwzzVsX5HNym8U6/DXVWNmtslCXoAePloX3QifZ1YIxtuDWmSAhbzwGa9dpgoUloDQ7z0yj6gR/LJLu/re7uwNBx90OZODaqBiJyWQUZ2cVrsEGUZkyiboIlQiMU0aVJvSwXkbywfgD9e7ai5yjvd8saPs8vP9zDhwotkpseL/250Ua5bSVXZ1x/l/9gMaQH8yQ99b9yRLTBW21HBT4aZNboL4xO+npMa7ZWn6S531wvgOYa3VopGSCLGMou4+h2VkpItrZ3F7e3IN35OhnFUqzy5zjCrkqT4gtGsiAzy2syxO5s3Tnf9auyrXmAGsIpUzogma0w9ieb0zlomxvi46L4Ru/TGulihqT1UjHeJlG4iQy5cBMBo4NsmVeyltvlmcYNwTeMMYtNXEty9y6PNlie4I5iCkYWDac9FElIRTpJ8zzIlTYcPh5ut2Nr8Pa8= wangxiang@gmail.com
```

#### 配置~/.ssh/config

由于git.domainname.in ssh端口为1058，需要配置~/.ssh/config

```
Host git.domainname.in
  IdentityFile ~/.ssh/id_rsa
  PreferredAuthentications publickey
  Port 1058
```

#### 将gogs加入known_hosts

ssh命令后出现类似日志即为成功，可以看到~/.ssh下多出了known_hosts文件。

```shell
bash-4.4$ ssh git@git.domainname.in
PTY allocation request failed on channel 0
Hi there, You've successfully authenticated, but Gogs does not provide shell access.
If this is unexpected, please log in with password and setup Gogs under another user.
Connection to git.domainname.in closed.
```

#### 配置gerrit replication插件

~/etc/目录下新建replication.config（[参考](https://gerrit.googlesource.com/plugins/replication/+/master/src/main/resources/Documentation/config.md)）。

```
vi replication.config
[remote "host-one"]
  url = git@git.domainname.in:wangxiang/${name}.git
  push = +refs/heads/*:refs/heads/*
  push = +refs/tags/*:refs/tags/*
  push = +refs/changes/*:refs/changes/*::q  
```

#### 重新加载replication插件

```shell
[root@node1 ~]# ssh -p 29418 admin@gerrit.domainname.in gerrit plugin reload replication
```

#### 手动开启同步

```shell
[root@node1 ~]# ssh -p 29418 admin@gerrit.domainname.in replication start --all
```

#### 查看同步日志

在gerrit容器中

```shell
bash-4.4$ tail -f replication_log 
[2020-07-17 09:39:55,314] scheduling replication All-Projects:..all.. => git@git.domainname.in:wangxiang/All-Projects.git
[2020-07-17 09:39:55,324] scheduled All-Projects:..all.. => [a995c7f6] push git@git.domainname.in:wangxiang/All-Projects.git to run after 15s
[2020-07-17 09:39:55,328] scheduling replication All-Users:..all.. => git@git.domainname.in:wangxiang/All-Users.git
[2020-07-17 09:39:55,328] scheduled All-Users:..all.. => [e98bbfd7] push git@git.domainname.in:wangxiang/All-Users.git to run after 15s
[2020-07-17 09:39:55,349] scheduling replication gerrit-test:..all.. => git@git.domainname.in:wangxiang/gerrit-test.git
[2020-07-17 09:39:55,371] scheduled gerrit-test:..all.. => [29a9d746] push git@git.domainname.in:wangxiang/gerrit-test.git to run after 15s
[2020-07-17 09:40:10,326] Replication to git@git.domainname.in:wangxiang/All-Projects.git started... [CONTEXT pushOneId="a995c7f6" ]
[2020-07-17 09:40:11,252] Created remote repository: git@git.domainname.in:wangxiang/All-Projects.git [CONTEXT pushOneId="a995c7f6" ]
[2020-07-17 09:40:11,253] Missing repository created; retry replication to git@git.domainname.in:wangxiang/All-Projects.git [CONTEXT pushOneId="a995c7f6" ]
[2020-07-17 09:40:11,257] Replication to git@git.domainname.in:wangxiang/All-Users.git started... [CONTEXT pushOneId="e98bbfd7" ]
[2020-07-17 09:40:11,843] Created remote repository: git@git.domainname.in:wangxiang/All-Users.git [CONTEXT pushOneId="e98bbfd7" ]
[2020-07-17 09:40:11,844] Missing repository created; retry replication to git@git.domainname.in:wangxiang/All-Users.git [CONTEXT pushOneId="e98bbfd7" ]
[2020-07-17 09:40:11,846] Replication to git@git.domainname.in:wangxiang/gerrit-test.git started... [CONTEXT pushOneId="29a9d746" ]
[2020-07-17 09:40:12,157] Replication to git@git.domainname.in:wangxiang/gerrit-test.git completed in 309ms, 16497ms delay, 0 retries [CONTEXT pushOneId="29a9d746" ]
```
### HTTP认证

修改gerrit.config中auth.type为http后开启HTTP认证登录模式。

```
[auth]
        type = http
```

开启后，需要配合nginx或httpd使用，本文使用nginx。

nginx的作用是完成http基础认证，认证通过后，会将用户名传给gerrit，并新建用户。相关yaml如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: gerrit-nginx
  name: gerrit-nginx
  namespace: gerrit
spec:
  selector:
    matchLabels:
      run: gerrit-nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: gerrit-nginx
    spec:
      containers:
      - image: harbor.domainname.in/domainname/gerrit-nginx
        imagePullPolicy: Always
        name: gerrit-nginx
        volumeMounts:
        - mountPath: /app/appdist
          name: htpasswd
      volumes:
      - name: htpasswd
        persistentVolumeClaim:
          claimName: gerrit-nginx
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: gerrit-nginx
  name: gerrit-nginx
  namespace: gerrit
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: gerrit-nginx
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gerrit
  namespace: gerrit
spec:
  rules:
  - host: gerrit.domainname.in
    http:
      paths:
      - backend:
          serviceName: gerrit-nginx
          servicePort: 80
        path: /
```

#### nginx配置

```
server {
    listen       80;

    allow   all; 
    deny    all; 
    auth_basic "Welcomme to Gerrit Code Review Site!"; 
    auth_basic_user_file /app/appdist/htpasswd.conf; 

    location / {  
        proxy_pass  http://gerrit:8080; 
    }  
    
}
```

#### 新增用户

```shell
htpasswd htpasswd.conf wangxiang-test2
```

### 邮件发送

之前使用263邮箱配置邮件发送，配置正确后，一直报如下错误：

```
com.google.gerrit.exceptions.EmailException: Mail Error: Server smtp.263.net rejected from address wangxiang@gmail.com
```

换了其它用户也无法发送，几经折腾后，使用我自己的QQ邮箱发送成功了。怀疑是263对我们的公网IP做了限制。

gerrit.config中邮箱相关配置如下：

```
[sendemail]
        enbale = true
        smtpServer = smtp.qq.com
        smtpUser = 45xxxx62@qq.com
        smtpPass = xxx
        connectTimeout = 30sec
        from = wangxiang<45xxxx62@qq.com>
        smtpServerPort = 465
        smtpEncryption = SSL
```

### 维护命令

##### 进入gerrit容器

```shell
[root@node1 ~]# kubectl exec -it -n gerrit $(kubectl get -n gerrit  pods -l run=gerrit  -o jsonpath="{.items[0].metadata.name}") /bin/bash
```

##### gerrit.config目录

```shell
bash-4.4$ pwd
/var/gerrit/etc
bash-4.4$ ls -rhl gerrit.config
```

##### 删除用户邮箱

```shell
[root@node1 ~]# ssh -p 29418 admin@gerrit.domainname.in gerrit set-account 1000002 --delete-email wangxiang@gmail.com
```

##### 将用户加入Administrators组

```shell
[root@node1 ~]# ssh -p 29418 admin@gerrit.domainname.in gerrit set-members Administrators --add wangxiang
```

##### 查看Administrators组成员

```shell
[root@node1 ~]# ssh -p 29418 admin@gerrit.domainname.in gerrit ls-members Administrators
id	username	full name	email
1000000	admin	admin	wangxiang@gmail.com
```

##### help

```shell
[root@node1 ~]# ssh -p 29418 admin@gerrit.domainname.in gerrit --help
gerrit [COMMAND] [ARG ...] [--] [--help (-h)]

 --          : end of options (default: false)
 --help (-h) : display this help text (default: true)

Available commands of gerrit are:

   apropos              Search in Gerrit documentation
   ban-commit           Ban a commit from a project's repository
   close-connection     Close the specified SSH connection
   create-account       Create a new batch/role account
   create-branch        Create a new branch
   create-group         Create a new account group
   create-project       Create a new project and associated Git repository
   flush-caches         Flush some/all server caches from memory
   gc                   Run Git garbage collection
   index                
   logging              
   ls-groups            List groups visible to the caller
   ls-members           List the members of a given group
   ls-projects          List projects visible to the caller
   ls-user-refs         List refs visible to a specific user
   plugin               
   query                Query the change database
   receive-pack         Standard Git server side command for client side git push
   reload-config        Reloads the Gerrit configuration
   rename-group         Rename an account group
   review               Apply reviews to one or more patch sets
   sequence             
   set-account          Change an account's settings
   set-head             Change HEAD reference for a project
   set-members          Modify members of specific group or number of groups
   set-project          Change a project's settings
   set-project-parent   Change the project permissions are inherited from
   set-reviewers        Add or remove reviewers on a change
   set-topic            Set the topic for one or more changes
   show-caches          Display current cache statistics
   show-connections     Display active client SSH connections
   show-queue           Display the background work queues
   stream-events        Monitor events occurring in real time
   test-submit          
   version              Display gerrit version

See 'gerrit COMMAND --help' for more information.
```

