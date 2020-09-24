---
layout: post
title:  "Spring多线程定时任务"
date:   2018-08-01 09:52:55 +0800
categories: spring schedule
tags:
    - java
    - spring
    - schedule
---

> 最近使用Spring Boot的@Schedule配置定时任务，但使用的时候发现如果不配置线程池，所有任务都会在单线程执行。任务执行时间较长时，会阻塞该线程，导致某些任务未执行。  

#### 问题描述

&emsp;&emsp; 包版本：spring-context:4.3.11.RELEASE 在这个版本使用@schedule注解时，如果不提供TaskScheduler，Spring会提供一个默认实现，但这个实现仅有一个线程，所以会出现上述问题。  
&emsp;&emsp; 下面代码位于类org.springframework.scheduling.config.ScheduledTaskRegistrar的scheduleTasks方法，可以看到当taskScheduler为null时，会提供一个单线程的线程池。  

	if (this.taskScheduler == null) {
		this.localExecutor = Executors.newSingleThreadScheduledExecutor();
		this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
	}


#### 解决方案
	
&emsp;&emsp; 只需要在@Configuration注解的类中提供一个TaskScheduler bean，并设置线程池大小即可解决问题，代码如下：  

	@Bean
	public TaskScheduler taskScheduler() {
	    ThreadPoolTaskScheduler threadPoolTaskScheduler = new ThreadPoolTaskScheduler();
	    threadPoolTaskScheduler.setPoolSize(20);
	    return threadPoolTaskScheduler;
	}

-------------------------------------------------------------------------------------
&emsp;&emsp; 需要注意的是，在使用spring-context:5.0.6.RELEASE时，如果不提供TaskScheduler bean，Spring使用NoOpScheduler而不是ThreadPoolTaskScheduler作为默认实现，NoOpScheduler是org.springframework.web.socket.config.annotation.WebSocketConfigurationSupport内部的一个静态类，代码如下：  

	private static class NoOpScheduler implements TaskScheduler {
	        private NoOpScheduler() {
	        }
	
	        @Nullable
	        public ScheduledFuture<?> schedule(Runnable task, Trigger trigger) {
	            throw new IllegalStateException("Unexpected use of scheduler.");
	        }
	
	        public ScheduledFuture<?> schedule(Runnable task, Date startTime) {
	            throw new IllegalStateException("Unexpected use of scheduler.");
	        }
	
	        public ScheduledFuture<?> scheduleAtFixedRate(Runnable task, Date startTime, long period) {
	            throw new IllegalStateException("Unexpected use of scheduler.");
	        }
	
	        public ScheduledFuture<?> scheduleAtFixedRate(Runnable task, long period) {
	            throw new IllegalStateException("Unexpected use of scheduler.");
	        }
	
	        public ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, Date startTime, long delay) {
	            throw new IllegalStateException("Unexpected use of scheduler.");
	        }
	
	        public ScheduledFuture<?> scheduleWithFixedDelay(Runnable task, long delay) {
	            throw new IllegalStateException("Unexpected use of scheduler.");
	        }
	    }

&emsp;&emsp; 可见，在这个版本如果不提供默认实现，Spring会通过异常的方式提醒你需要自己配置而不是默认实现。

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
