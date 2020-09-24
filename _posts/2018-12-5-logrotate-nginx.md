---
layout: post
title:  " Logrotate管理nginx日志"
date:   2018-12-5 10:23:56 +0800
categories: nginx
tags:
    - nginx
    - logrotate
---

## Logrotate管理nginx日志

### 问题描述

nginx使用默认日志配置时，所有的日志都会吐到access.log和error.log两个文件中，会造成日志文件过于庞大，不方便管理和维护。而nginx并不附带日志分割、清理的功能。所以需要其他工具来辅助nginx完成日志管理任务。

### 解决方案

logrotate + crontab。

### logrotate简介

logrotate是一款管理系统日志文件工具。它功能十分强大，它可以通过配置完成日志文件自动分割、压缩、删除的工作。处理日志文件的频率可以是每天、每周或每月。也可以设置日志文件大小阈值，当日志文件超过阈值时，处理文件。

#### logrotate配置文件

默认路径/etc/logrotate.d/nginx

```
/app/nginx/logs/access.log /app/nginx/logs/error.log { 
daily 
rotate 3
dateext 
compress 
sharedscripts 
postrotate 
  [ ! -f /var/run/nginx.pid ] || kill -USR1 `cat /var/run/nginx.pid` 
endscript 
} 
```

| 字段                                | 含义                              |
| ----------------------------------- | --------------------------------- |
| daily                               | 频率：天                          |
| rotate 3                            | 日志文件被分割超过3次后将会被删除 |
| dateext                             | 旧日志文件加时间后缀              |
| compress                            | 默认使用gzip压缩旧日志文件        |
| sharedscripts                       | postrotate仅被执行一次            |
| postrotate                          | 后置操作                          |
| kill -USR1 `cat /var/run/nginx.pid` | NGINX收到-USR1后，会重启它的日志  |

更详细的参数可以查看man logrotate。

### 测试

```
logrotate -fv /etc/logrotate.d/nginx
```

测试时加-f参数表示强制处理，而-v参数则表示打印处理信息。

### 定时任务

```
vi /etc/crontab
  1  0  *  *  * root       logrotate /etc/logrotate.d/nginx
```
