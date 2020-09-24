---
layout: post
title:  "Spring读配置文件"
date:   2018-08-16 17:38:56 +0800
categories: spring schedule
tags:
    - spring
    - config
---

> 本文给出了一种根据不同Profile读取响应配置文件的方法。  

#### 问题描述  
&emsp;&emsp; 在SpringBoot中，基本使用@Configuration和@Bean注解完成bean配置。而application.properties和application.yml文件(下文将这两个文件称为中央配置文件)中的配置文件会被Spring自动读取，并可以用@Value注解完成注入。但实际工作中仍需要将一些配置放在非中央配置文件中。

#### 解决方案  
&emsp;&emsp; 此时，可以通过@PropertySource注解解决问题，并使用占位符根据当前profile载入不同的配置文件，下面是示例代码：  

	@Configuration
	@PropertySource("classpath:myConfig-${spring.profiles.active}.yml")
	public class MyConfig {
	
	    @Bean
	    public String url(@Value("${ip:0.0.0.0}")String ip, @Value("${port:0}")String port) {
	        String url = ip + " + " + port;
	        System.out.println(url);
	        return url;
	    }
	
	}                                                                                      
	
---
	myConfig-dev.yml
	ip: 127.0.0.1
	port: 8080

-----
	myConfig-test.yml
	ip: 192.168.0.0

&emsp;&emsp; 当启动profile是dev时，输出"127.0.0.1 + 8080"。  
&emsp;&emsp; 当启动profile是test时，输出"192.168.0.0 + 8080"。  
&emsp;&emsp; 至此，我们完成了根据不同profile读取相应配置文件的功能。其中，spring.profiles.active还可以替换为其它参数。
#### @Value

&emsp;&emsp; 此注解可以在成员变量、构造器参数、setter方法参数中使用，将配置文件中的数据注入到相应字段内。有@Value("${xx:x}")和@Value("#file")

&emsp;&emsp; 项目启动后，控制台输出"127.0.0.1 + 8080"，配置文件注入成功！  

&emsp;&emsp;参考文档：<https://docs.spring.io/spring/docs/5.0.x/spring-framework-reference/core.html>

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
