---
layout: post
title:  "虚拟机类加载机制"
date:   2018-07-14 10:49:55 +0800
categories: reading
tags:
    - java
    - ClassLoader
---
&emsp;&emsp;最近在读周志明老师的深入理解java虚拟机。这篇博文总结一下第七章虚拟机类加载机制的内容。  
> 什么是类加载机制？  

&emsp;&emsp;将.class文件加载到内存，最终形成虚拟机可以直接使用的java类型的过程。  

> 类加载的全过程  

&emsp;&emsp;加载、连接（验证、准备、解析）、初始化。  

> 加载阶段  

&emsp;&emsp;1) .class -> 二进制字节流（ClassLoader控制字节流的获取方式)  
&emsp;&emsp;2) 字节流 -> 方法区的运行时数据结构  
&emsp;&emsp;3) 生成java.lang.Class对象（对于HotSpot而言，在方法区中）  

> 验证  

> 准备  

&emsp;&emsp;为类变量(存在方法区)分配内存、设置初始值。  
&emsp;&emsp;1) static变量 -> 设置为对应初始值   
&emsp;&emsp;2) static final常量 -> 设置为常量值  

> 解析  

&emsp;&emsp;符号引用 -> 直接引用

> 初始化  

&emsp;&emsp;根据程序员制定的主观计划初始化类变量和其它资源，即执行类构造器<clinit>方法，<clinit>由以下两部分组成。  
&emsp;&emsp;1) 类变量的赋值操作  
&emsp;&emsp;2) static块中的代码  

> 类加载器  

&emsp;&emsp;类加载器实现了“通过一个类的全限定名获取描述此类的二进制字节流”过程。  
&emsp;&emsp;两个类相等 = 类加载器相同 + 源于同一个Class文件。  
&emsp;&emsp;分类：启动类加载器(Bootstrap ClassLoader)、扩展类加载器(Extension ClassLoader)、应用程序类加载器(Application ClassLoader)。  
&emsp;&emsp;启动类加载器：加载<JAVA_HOME>\lib目录中或被-Xbootclasspath参数指定的路径中的类库。  
&emsp;&emsp;扩展类加载器：加载<JAVA_HOME>\lib\ext目录中的，或被java.ext.dirs系统变量指定的路径中的类库。  
&emsp;&emsp;应用程序类加载器：加载ClassPath上所指定的类库。  

> 双亲委派模型  

&emsp;&emsp;一个类加载器收到一个类加载请求时，首先交给父加载器加载，只有父加载器未找到这个类的时候，子类加载器才尝试自己加载。  

> 破坏双亲委派模型  

&emsp;&emsp;1) JDK 1.2之前  
&emsp;&emsp;2) 基础类回调用户类时，引入了线程上下文类加载器  
&emsp;&emsp;3) 用户对程序动态性的追求导致的，包括代码热替换、模块热部署等

&emsp;&emsp;

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
