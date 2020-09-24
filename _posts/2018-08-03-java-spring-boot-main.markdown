---
layout: post
title:  "SpringBoot源码阅读（一）"
date:   2018-08-03 10:55:56 +0800
categories: spring schedule
tags:
    - java
    - spring-boot
    - source
---

> 一直在用Spring，非常好用，但是对于Spring的理解并不深刻，所以计划通过走读spring-boot代码，深化对Spring的理解。  
> 包版本：spring-boog:2.0.2.RELEASE  

### SpringApplication.java

&emsp;&emsp; 通过main方法启动项目后，会调用SpringApplication类中的run方法，代码如下：  
  
	/**
	 * Run the Spring application, creating and refreshing a new
	 * {@link ApplicationContext}.
	 * 渣翻译：运行Spring应用，创建并刷新一个新的ApplicationContext
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return a running {@link ApplicationContext}
	 */
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;  // 被创建的ApplicationContext
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();  // 实例化ApplicationContext
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			// 为单例实例化做准备，读取Resource，向registry中注册beanDefinition
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);  
			refreshContext(context); // 实例化单例bean
			afterRefresh(context, applicationArguments); // 后置方法，启动定时任务
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}
	
		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
	
&emsp;&emsp; 实际上，这个方法调用完成后，一个spring-boot应用就已经启动了。这种写法配合注释，使得阅读代码轻松了不少。接下来我们依次看一下跟ApplicationContext有关的代码。  

#### createApplicationContext()  

	/**
	 * Strategy method used to create the {@link ApplicationContext}. By default this
	 * method will respect any explicitly set application context or application context
	 * class before falling back to a suitable default.
	 * 渣翻译：
	 * 创建ApplicationContext的策略方法。默认情况下，这个方法会在返回合适的默认context
	 * 之前，根据指定的application context或application context class选择相应的context
	 * @return the application context (not yet refreshed)
	 * @see #setApplicationContextClass(Class)
	 */
	protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				switch (this.webApplicationType) {
				case SERVLET:
					contextClass = Class.forName(DEFAULT_WEB_CONTEXT_CLASS);
					break;
				case REACTIVE:
					contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
					break;
				default:
					contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
				}
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, "
						+ "please specify an ApplicationContextClass",
						ex);
			}
		}
		return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
	
&emsp;&emsp; 可以看到，在找到合适的contextClass之后（此处是AnnotationConfigApplicationContext），调用了工具方法，完成实例化。此方法会调用contextClass的无参构造进行实例化。下面我们看一下AnnotationConfigApplicationContext的无参构造：  
&emsp;&emsp; 至此，createApplicationContext()方法就调用完了，从结果上看相当于使用new AnnotationConfigApplicationContext()生成了一个ApplicationContext实例。  

#### prepareContext()  

&emsp;&emsp; 此方法为后续refresh()操作做铺垫，设置environment，将boot指定的单例注册到beanFactory中。  

#### refresh()

&emsp;&emsp; 此方法位于抽象类AbstractApplicaitontext中，它是AnnotationApplicationContext的父类的父类。由于这是一个重写方法，它的注释在它实现的接口ConfigurableApplicationContext中，我们先来看下注释。  
  
	/**
	 * Load or refresh the persistent representation of the configuration,
	 * which might an XML file, properties file, or relational database schema.
	 * <p>As this is a startup method, it should destroy already created singletons
	 * if it fails, to avoid dangling resources. In other words, after invocation
	 * of that method, either all or no singletons at all should be instantiated.
	 * 渣翻译：
	 * 从XML文件、properties文件或关系型数据库中载入或刷新配置。
	 * 由于这是一个启动方法，如果它失败了，应该销毁所有已创建的单例，以避免dangling(悬挂？我理解为长时间占用)
	 * 资源。换句话说，在这个方法调用完成后，要么所有的单例都被实例化，要么没有单例实例化。
	 * P.S. synchroinzed锁+失败后的回滚机制，个人理解为开启了事物。
	 * @throws BeansException if the bean factory could not be initialized
	 * @throws IllegalStateException if already initialized and multiple refresh
	 * attempts are not supported
	 */
	void refresh() throws BeansException, IllegalStateException;

&emsp;&emsp; 接下来就是实现方法了：

	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 为刷新context做准备
			prepareRefresh();
	
			// Tell the subclass to refresh the internal bean factory.
			// 通知子类刷新内部bean工厂(此处我们获得的是一个DefaultListableBeanFactory)
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
	
			// Prepare the bean factory for use in this context.
			// 对在context中使用的bean工厂进行设置
			prepareBeanFactory(beanFactory);
	
			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 允许context子类对bean factory进行后置操作（钩子方法）
				postProcessBeanFactory(beanFactory);
	
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);
	
				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);
	
				// Initialize message source for this context.
				initMessageSource();
	
				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();
	
				// Initialize other special beans in specific context subclasses.
				onRefresh();
	
				// Check for listener beans and register them.
				registerListeners();
	
				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);
	
				// Last step: publish corresponding event.
				finishRefresh();
			}
	
			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context " +
					 "initialization - cancelling refresh attempt: " + ex);
				}
	
				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();
	
				// Reset 'active' flag.
				cancelRefresh(ex);
	
				// Propagate exception to caller.
				throw ex;
			}
	
			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}

&emsp;&emsp; 熟悉的代码形式，这里使用了模板方法模式，定义了refresh的执行流程。

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
