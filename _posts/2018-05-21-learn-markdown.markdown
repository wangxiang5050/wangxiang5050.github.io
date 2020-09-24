---
layout: post
title:  "学习markDown!"
date:   2018-05-21 17:38:55 +0800
categories: jekyll update
---
刚用git + jekyll搭建了自己的博客，现在学习下markdown的语法~  
#### 换行  
>space + space + enter  

之前都使用回车 + 回车换行，但在页面上间距太大了，故使用这种双空格+回车的方式。

#### 区块引用
>	>  
仅仅一个>符号，就能实现区块引用，还是很方便  

#### 代码块
> space + space + space + space 或 tab  

这应该是最常用的功能了，贴代码就靠它啦。用UE的列编辑模式 + tab，贴代码还是很方便的。而作为我博客的第一段代码，当然还是Hello World!啦。  

	package com.example.markdown;
	
	public class HelloWorld {
	
	    public static void main(String[] args) {
	        System.out.println("Hello World!");
	    }
	    
	}
	
#### 分割线  
>	* * *  
>	***  
>	*****  
>	- - -  
>	---------------------------------------

一行中用三个以上的星号、减号、底线来建立一个分隔线，行内不能有除了空格意外其他东西。  
#### 链接  
##### 行内式  
>	[链接文字](链接地址 "链接title")  

要建立一个行内式的链接，只要在方块括号后面紧接着圆括号并插入网址链接即可，如果你还想要加上链接的 title 文字，只要在网址后面，用双引号把 title 文字包起来即可。下面是一个例子。

	这是我的[gitHub](https://github.com/wangxiang5050 "风亦憔悴的github")。  
		
这是我的[gitHub](https://github.com/wangxiang5050 "风亦憔悴的github")。  
  
##### 参考式  
> [链接文字][id]  
> [id]: 链接地址 "title"  

参考式的链接是在链接文字的括号后面再接上另一个方括号，而在第二个方括号里面要填入用以辨识链接的id，若不填则id为链接文字本身。下面是一个例子。

	我经常使用[谷歌][]、[百度][]等搜索引擎查阅API文档。
	[谷歌]:http://google.com.hk/ "google"
	[百度]:http://www.baidu.com/ "baidu" 
	
我经常使用[谷歌][]、[百度][]等搜索引擎查阅API文档。 

[谷歌]:http://google.com.hk/ "google"
[百度]:http://www.baidu.com/ "baidu"  

参考文档：<https://www.appinn.com/markdown/index.html> 


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
