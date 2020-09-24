---
layout: post
title:  "Helm tiller清理过期configmap"
date:   2020-07-14 14:03:00 +0800
categories: helm
tags:
    - helm
    - tiller
    - kubernetes
---

# Helm tiller清理过期configmap #

我们使用helm（v2.13）作为持续集成的工具，今天突然发现kube-system下有很多tiller生成的configmap。这些configmap从未清理过，导致命令行联想很慢，因此，需要有相应的清理机制删除不需要的release历史。

### 生成时机

这些configmap是什么时候生成的呢？

每一个relase版本就会有一个configmap生成，从而方便回滚。

这里，release版本的定义为：Chart + Values = Release，可以理解为对于同一个Chart，每变更一次配置便会生成一个新的release。同时，也会有一个configmap在kube-system命名空间下产生。

直接查看这些configmap时，内容无法理解，可以通过helm get 命令查看。

### 清理机制

在2.7版本后，可以在运行helm init时，指定--history-max参数设置release最多保留多少版本（0表示无限制）。对于已安装的tiller可以手动编辑deployment，修改环境变量TILLER_HISTORY_MAX完成变更。

```
kubectl  edit -n kube-system deployment tiller-deploy
...
      containers:
      - env:
        - name: TILLER_NAMESPACE
          value: kube-system
        - name: TILLER_HISTORY_MAX
          value: "1"
...
```

参数修改并重启后，再次使用helm upgrade命令更新Chart时，历史记录就会被清理掉。

### 清理脚本

如果想先全量清理一遍，可以使用[ultimateboy](https://github.com/helm/helm/issues/2332#issuecomment-336565784)提供的脚本。

```
#!/usr/bin/env bash

TARGET_NUM_REVISIONS=10
TARGET_NUM_REVISIONS=$(($TARGET_NUM_REVISIONS+0))

RELEASES=$(kubectl --namespace=kube-system get cm -l OWNER=TILLER -o go-template --template='{{range .items}}{{ .metadata.labels.NAME }}{{"\n"}}{{ end }}' | sort -u)

# create the directory to store backups
mkdir configmaps

for RELEASE in $RELEASES
do
  # get the revisions of this release
  REVISIONS=$(kubectl --namespace=kube-system get cm -l OWNER=TILLER -l NAME=$RELEASE | awk '{if(NR>1)print $1}' | sed 's/.*\.v//' | sort -n)
  NUM_REVISIONS=$(echo $REVISIONS | tr " " "\n" | wc -l)
  NUM_REVISIONS=$(($NUM_REVISIONS+0))

  echo "Release $RELEASE has $NUM_REVISIONS revisions. Target is $TARGET_NUM_REVISIONS."
  if [[ $NUM_REVISIONS -gt $TARGET_NUM_REVISIONS ]]; then
    NUM_TO_DELETE=$(($NUM_REVISIONS-$TARGET_NUM_REVISIONS))
    echo "Will delete $NUM_TO_DELETE revisions"

    TO_DELETE=$(echo $REVISIONS | tr " " "\n" | head -n $NUM_TO_DELETE)

    for DELETE_REVISION in $TO_DELETE
    do
      CMNAME=$RELEASE.v$DELETE_REVISION
      echo "Deleting $CMNAME"
      # Take a backup
      kubectl --namespace=kube-system get cm $CMNAME -o yaml > configmaps/$CMNAME.yaml
      # Do the delete
      kubectl --namespace=kube-system delete cm $CMNAME
    done
  fi
done
```

