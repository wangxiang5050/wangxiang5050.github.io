---
layout: post
title:  "Spring lookup注解"
date:   2018-05-29 23:46:55 +0800
categories: reading
---
&emsp;&emsp;最近工作中重构代码，需要将prototype类型的组件注入到singleton类型的组件中，但是只是声明scope是prototype，并使用@autowire注入的话，并不能满足需要。个人理解是项目启动的时候已经组装好了，导致单例bean中的原型bean也是同一个。下面看一个例子：  

	// 原型组件
	@Component
	@Scope(value = "prototype")
	public class MyHandlerImpl implements IHandler {
	
	    private String str;
	
	    public MyHandlerImpl(String str) {
	        this.str = str;
	    }
	
	    @Override
	    public void handle() {
	        System.out.println(this + ", " + str);
	    }
	
	    public String getStr() {
	        return str;
	    }
	
	    public void setStr(String str) {
	        this.str = str;
	    }
	}

***

	// 单例组件
	@Service
	public  class PersonServiceImpl implements PersonService {
	
	    @Autowired
	    private MyHandlerImpl h1;
	
	    @Override
	    public String getPerson() {
	        MyHandlerImpl iHandler = createHandler("I'm @lookup handler!");
	        iHandler.handle();
	
	        h1.setStr("I'm autowired handler!");
	        h1.handle();
	        return "success";
	    }
	
	    @Lookup
	    public MyHandlerImpl createHandler(String str) {
	        return null;
	    }
	
	}
	
***
	
	// 单元测试
	@RunWith(SpringRunner.class)
	@SpringBootTest(classes = ConfigdemoApplication.class)
	@WebAppConfiguration
	public class ConfigdemoApplicationTests {
	
		@Autowired
		private PersonService personServiceImpl;
	
		@Test
		public void contextLoads() {
			personServiceImpl.getPerson();
			personServiceImpl.getPerson();
		}
	
	}
	
***
	
	// 代码输出
	com.example.configdemo.spring.service.mycomponent.impl.MyHandlerImpl@12958360, I'm @lookup handler!
	com.example.configdemo.spring.service.mycomponent.impl.MyHandlerImpl@c6e0f32, I'm autowired handler!
	com.example.configdemo.spring.service.mycomponent.impl.MyHandlerImpl@6f3f0fae, I'm @lookup handler!
	com.example.configdemo.spring.service.mycomponent.impl.MyHandlerImpl@c6e0f32, I'm autowired handler!
	
&emsp;&emsp;可以看到，使用@lookup时，两次的MyHandlerImpl不是同一个对象，而使用autowired注入的话，则是同一个对象。  
&emsp;&emsp;需要注意的是，**被@lookup注解的方法需要是public的哦！**
&emsp;&emsp;这是lookup使用方法的原文：<http://www.baeldung.com/spring-lookup>  
&emsp;&emsp;下面我来大致翻译一下。
<h3>1. 介绍</h3>  
&emsp;&emsp;这次我们来看一下如何通过方法级的@Lookup注解完成依赖注入的。  
<h3>2. 为什么用@Lookup?</h3>  
&emsp;&emsp;一个有@Lookup注解的方法会告诉Spring，当我们调用这个方法时，Spring会返回一个方法返回值类型的实例。  
&emsp;&emsp;实质上，Spring会父爱我们的注解方法并使用我们方法的返回值和参数作为BeanFactory#getBean方法的入参。  
&emsp;&emsp;@Lookup注解的使用情景：  
> 1) 向一个单例bean注入一个原型bean(prototype-scoped bean)(类似于Provider)  
> 2) Injecting dependencies procedurally(暂译为程序依赖注入，有不妥请指正)  
&emsp;&emsp;@Lookup相当于XML元素中的lookup-method  
<h3>3. 如何使用@Lookup</h3>  
<h4>3.1 向一个单例bean注入原型bean</h4>  
&emsp;&emsp;当我们决定使用一个原型bean时，我们必须面对的问题就是我们的单例bean要如何使用这些原型bean呢？  
&emsp;&emsp;当然可以使用Provider，但是@Lookup在某些方面更合适一些。  
&emsp;&emsp;首先，我们先创建一个之后需要注入到单例bean的原型bean。  

	@Component
	@Scope("prototype")
	public class SchoolNotification {
	    // ... prototype-scoped state
	}
	
&emsp;&emsp;之后我们使用@Lookup创建一个单例bean:  

	@Component
	public class StudentServices {
	 
	    // ... member variables, etc.
	 
	    @Lookup
	    public SchoolNotification getNotification() {
	        return null;
	    }
	 
	    // ... getters and setters
	}
	
&emsp;&emsp;使用@Lookup，我们能在单例bean中得到一个SchoolNotification实例。  

	@Test
	public void whenLookupMethodCalled_thenNewInstanceReturned() {
	    // ... initialize context
	    StudentServices first = this.context.getBean(StudentServices.class);
	    StudentServices second = this.context.getBean(StudentServices.class);
	        
	    assertEquals(first, second); 
	    assertNotEquals(first.getNotification(), second.getNotification()); 
	}
	
&emsp;&emsp;注意，在StudentServices中，我们将getNotification方法设置成了一个空方法(we left the getNotification method as a stub)。  
&emsp;&emsp;这是因为，spring通过调用beanFactory.getBean(StudentNotification.class)重写了这个方法，所以我们能将它置空。  
&emsp;&emsp;3.2 部分就是说使用@Lookup注入的话，还可以通过构造器传参，其实就是spring调用beanFactory.getBean(class, name)实现的。

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
