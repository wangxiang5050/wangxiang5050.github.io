---
layout: post
title:  "shell tips"
date:   2020-07-16 11:39:00 +0800
categories: shell
tags:
    - shell
---

# Shell tips #

### 问题

现在使用tekton作为CI/CD工具，相关脚本也从jenkins groovy迁移到了shell。但一直有两个问题比较困扰，一个是在脚本中某行命令出现异常时，不会中断，导致任务状态不对，需要判断可能出错的命令行，并用exit命令退出，导致脚本较为冗余。另外一个问题是，在命令执行时，所执行的命令未打印到标准输出，调试很痛苦，如果能实现jenkins的效果，即在运行命令前将命令先打印出来，就很方便了。

### 解决方案

今天在逛stackoverflow尝试解决第一个问题时，发现了set -e参数，它可以实现我想要的效果，即任何返回非0的命令都会导致脚本异常退出。又偶然一瞥，发现了更好用的-x参数，它可以完美解决第二个问题。NICE呀！不过想来之前也看到过这行命令，但从未深究过，导致脚本长时间存在上述两个问题，细节还是不能忽略呀，一行不起眼的命令，竟然有如此效果。[man set](http://linuxcommand.org/lc3_man_pages/seth.html) 这两个参数摘录如下：

```
 -e  Exit immediately if a command exits with a non-zero status.
 -x  Print commands and their arguments as they are executed.
```

