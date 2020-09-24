---
layout: post
title:  "SpringBoot读取Xml"
date:   2018-08-21 09:59:56 +0800
categories: spring schedule
tags:
    - spring-boot
    - xml
---

> 在spring-boot或以注解配置为主的spring项目中，如果想使用XML定义bean，并交由spring容器管理，应该如何实现呢？  

#### 问题描述  
&emsp;&emsp; 在以XML配置为主的spring项目中，若想使用java配置，只需要将被@Configuration注解的类放在spring扫描路径下即可。然而，在spring-boot中如何使用XML配置bean，并交由spring容器管理呢？  

#### 解决方案  
&emsp;&emsp; 此时，只需要在被@Configuration注解的类上新增@ImportResource注解，并指定XML文件路径即可，下面是示例代码：  
           
	MyXmlBean.java
	@Configuration
	@ImportResource("classpath:xmlConfig/MyXmlConfig.xml")
	public class MyXmlBean {
	
	    @Autowired
	    private Player wx;
	
	    @Bean
	    public Player playerWx() {
	        System.out.println(wx);
	        return wx;
	    }
	
	}                                                                                   

---
	Player.class
	public class Player {
	    private String name;
	    private double currCost;
	    private double totalCost;
	    private StrategyFactory factory;
	
	    public void buySth(double price) {
	        System.out.println(name + " buy a " + price + " good");
	        totalCost += price;
	        CalculatePrice calculateStrategy = factory.determineStrategy(this.totalCost);
	        double payPrice = calculateStrategy.calculatePrice(price);
	        System.out.println("actual pay " + payPrice);
	    }
	
	    public String getName() {
	        return name;
	    }
	
	    public void setName(String name) {
	        this.name = name;
	    }
	
	    public double getCurrCost() {
	        return currCost;
	    }
	
	    public void setCurrCost(double currCost) {
	        this.currCost = currCost;
	    }
	
	    public double getTotalCost() {
	        return totalCost;
	    }
	
	    public void setTotalCost(double totalCost) {
	        this.totalCost = totalCost;
	    }
	
	    public StrategyFactory getFactory() {
	        return factory;
	    }
	
	    public void setFactory(StrategyFactory factory) {
	        this.factory = factory;
	    }
	
	    @Override
	    public String toString() {
	        return "Player{" +
	                "name='" + name + '\'' +
	                ", currCost=" + currCost +
	                ", totalCost=" + totalCost +
	                '}';
	    }
	
	}	

---
	myConfig-dev.yml
	<?xml version="1.0" encoding="UTF-8"?>
	<beans xmlns="http://www.springframework.org/schema/beans"
	       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
	
	    <bean id="wx" class="com.example.demo.strategy.player.Player">
	        <property name="name" value="wx"></property>
	        <property name="totalCost" value="1000"></property>
	    </bean>
	
	</beans>

&emsp;&emsp; 项目启动时，控制台输出"Player{name='wx', currCost=0.0, totalCost=1000.0}"，在XML文件中配置的bean已经成功注入进来。  

&emsp;&emsp;参考文档：<https://docs.spring.io/spring/docs/5.0.x/spring-framework-reference/core.html>

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
