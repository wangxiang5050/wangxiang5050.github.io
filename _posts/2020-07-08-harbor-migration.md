---
layout: post
title:  "Harbor迁移"
date:   2020-07-08 17:42:00 +0800
categories: harbor
tags:
    - harbor
---

# Harbor迁移 #

Harbor因为几个月前埋下的坑，迁移起来异常艰难。这就是安装时的随意，导致维护的艰辛吧。  

## 问题描述 ##

Harbor的主要问题是使用的镜像版本是goharbor/harbor-core:dev，而这个版本是在持续开发的，最近更新时间是10小时前。所以，当harbor的pod在新节点启动时，拉取了最新的镜像，而这个镜像又运行了最新的脚本，导致数据库表结构发生变化，最终结果就是启动异常。 

## 解决方案 ##

将镜像还原至备份版本后，依然无法启动，报错如下：

```
[/core/main.go:106]: failed to initialize database: file does not exist
```

报错原因：

```
[root@node1 maintain]# kubectl exec -it harbor-slb-harbor-database-0 bash -n harbor-slb
root [ / ]# psql -U postgres -d registry
psql (9.6.14)
Type "help" for help.

registry=# select * from schema_migrations;
 version | dirty | data_version 
---------+-------+--------------
      30 | f     |           30
(1 row)

registry=# 
```

可以看到，由于跑过新版镜像，所以数据库中schema_migrations.version值变成了30。在用老镜像启动harbor-slb-harbor-core时，会在/harbor/migrations/postgresql目录下找0030_开头的SQL文件执行。很显然，旧版本不可能有新版本的SQL文件，所以报错了。知道问题的原因就好解决了，将version字段修改为10，并重启harbor-slb-harbor-core的pod后，Harbor启动成功。  
但另一个问题随之而来，虽然启动成功后的Harbor可以在页面登录，但仅能拉取镜像，不能推送镜像，报错如下：

``` 
2020-03-06T06:46:38Z [WARNING] [/core/middlewares/countquota/handler.go:52]: Error occurred when to handle request in count quota handler: failed to compute the resources for quota, error: error occurred when to check Manifest existence pq: column t0.repo does not exist
```  

由错误日志column t0.repo does not exist，想到可能是t0表repo字段不存在导致的，查阅SQL后，发现artifact表字段发生了变化，使用SQL增加缺少字段后，问题得到解决。

```
alter table artifact add column repo varchar(255);
alter table artifact add column tag varchar(255);
alter table artifact add column kind varchar(255);
alter table artifact add column creation_time timestamp;
```

## 总结 ##

由于使用了粗暴的修改表结构的方式解决了问题，可能存在种种未知隐患，最好使用稳定版本搭建一套新的Harbor，并将数据迁移过去。

