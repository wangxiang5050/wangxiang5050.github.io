---
layout: post
title:  "Spring Autowire"
date:   2018-08-16 10:36:56 +0800
categories: spring schedule
tags:
    - spring
    - Autowired
    - Resource
---

> 一直在用spring，也没有仔细读过文档，正好最近在交接，读了一下。本文把使用注解自动注入这部分内容总结一下。  

### 自动注入  
&emsp;&emsp; Spring容器能够自动注入bean，自动注入有如下优势：  
- 可以减少属性配置和构造器参数
- 可以在对象变化时自动更新配置  

#### @Autowired  
&emsp;&emsp; 在语意上，@Autowired表示按类型注入，并可以配合@Qualifier进行类型限定。但@Autowired在按类型查找失败时，会根据bean名称查找。使用范围：  

- 构造器中
- setter方法上
- 任意方法上，此方法可以包含多个参数
- 成员变量上，也可以配合构造器一起使用  

&emsp;&emsp; @Autowired也支持将ApplicationContext中的同一类bean注入到数组、集合中，并使用@Order控制这些bean加入集合的顺序。注意，@Order注解并不能控制bean实例化的顺序。示例代码如下：    

	public class MovieRecommender {
		
	    @Autowired
	    private MovieCatalog[] movieCatalogs;
	
	    // ...
	} 
	
&emsp;&emsp; MovieCatalog是一个接口，在实例变量上使用@Autowired注解后，就会向这个数组中注入所有由Spring管理的MovieCatalog接口的实现类的实例。  
&emsp;&emsp; 默认情况下，若没找到候选bean，自动注入就会失败，抛出异常，可以通过以下方式避免异常抛出：  

	public class SimpleMovieLister {
	
	    private MovieFinder movieFinder;
	
	    @Autowired(required = false)
	    public void setMovieFinder(MovieFinder movieFinder) {
	        this.movieFinder = movieFinder;
	    }
	
	    // ...
	}           
	
#### @Qualifier
&emsp;&emsp; qualifier限定符，顾名思义，它配合@Autowired注解使用，以对自动注入实现更细粒度的控制。下面给出一个两注解配合使用的例子：  

	public interface Phone {  
	    void open();        
	}
	
---
	@Component("huawei")
	@Qualifier("domesticPhone")
	public class HuaWeiPhone implements Phone{
	
	    @Override
	    public void open() {
	        System.out.println("HuaWeiPhone is opening!");
	    }
	
	}

---
	@Component("apple")
	@Qualifier("americanPhone")
	public class ApplePhone implements Phone {
	
	    @Override
	    public void open() {
	        System.out.println("Apple phone is opening!");
	    }
	
	}

---
	@Component
	public class PhoneOwner {
	
	    @Autowired
	    @Qualifier("domesticPhone")
	    private Phone myPhone;
	
	    @PostConstruct
	    public void openMyPhone() {
	        myPhone.open(); // HuaWeiPhone is opening!
	    }
	
	}  
	
&emsp;&emsp; 可以看到，在PhoneOwner中我们只需要注入国产手机，所以，myPhone被注入了一个HuaWeiPhone的实例，并输出了"HuaWeiPhone is opening!"。  

#### @Resource 
&emsp;&emsp; @Resource注解可以根据bean名称注入，但@Resource的使用具有局限性，它只能在成员变量和单参数的setter方法中使用。  
#### 结论  
&emsp;&emsp; 所以，对于自动注入，几乎任何情况都可以使用@Autowired配合@Qualifier实现。@Resource在使用时要考虑到它的局限性。

&emsp;&emsp;参考文档：<https://docs.spring.io/spring/docs/5.0.x/spring-framework-reference/core.html>

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
