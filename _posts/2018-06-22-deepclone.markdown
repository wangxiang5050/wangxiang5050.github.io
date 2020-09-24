---
layout: post
title:  "java-深度克隆"
date:   2018-06-25 14:21:55 +0800
categories: reading
---
&emsp;&emsp;Hello，我又回来啦！最近在工作中需要深度克隆一个对象，但又不想每个对象都实现clone接口，于是就研究了一下如何优雅的深度克隆一个对象呢？    
&emsp;&emsp;首先是为什么不用clone接口呢，因为它需要你手动实现clone方法，并填充方法，否则就是浅clone。个人感觉一点都不优雅。  
&emsp;&emsp;那么优雅的方法是什么呢？可以使用org.springframework.util.SerializationUtils或org.apache.commons.lang.SerializationUtils来实现。  
&emsp;&emsp;下面用spring的工具类来实现。  
	
	package com.example.clone;
	
	
	import org.springframework.util.SerializationUtils;
	
	import java.util.Objects;
	
	/**
	 * wang.xiang 2018.6.22
	 * 克隆对象工具类
	 */
	public class CloneUtils {
	
	    /**
	     * 使用SerializationUtils深度克隆一个对象。
	     * @param source 源对象
	     * @param <T> 源对象类型
	     * @return 克隆对象
	     */
	    public static <T> T deepClone(T source) {
	        if(Objects.isNull(source)) {
	            throw new IllegalArgumentException("source can not be null!");
	        }
	        byte[] serializedObject = SerializationUtils.serialize(source);
	        return (T) SerializationUtils.deserialize(serializedObject);
	    }
	
	}
  
&emsp;&emsp;可以看到，会先使用serialize方法将对象序列化（底层实际是使用流将对象转成byte[]），之后再将其转换为对象。需要注意的是**实体类必须实现serializable接口！！！**  
&emsp;&emsp;来测试一下，测试实体：  

	package com.example.clone;
	
	import java.io.Serializable;
	import java.util.Optional;
	
	public class Book implements Serializable {
	
	    private String name;
	    private int totalPageNbr;
	    private BookMark bookMark;
	
	    public Book(String name, int totalPageNbr) {
	        this.name = name;
	        this.totalPageNbr = totalPageNbr;
	    }
	
	    public void setBookMarkInfo(int currPage, int currLine) {
	        bookMark = Optional.ofNullable(bookMark).orElse(new BookMark());
	        bookMark.setCurrPage(currPage);
	        bookMark.setCurrLine(currLine);
	    }
	
	    public String getName() {
	        return name;
	    }
	
	    public void setName(String name) {
	        this.name = name;
	    }
	
	    public int getTotalPageNbr() {
	        return totalPageNbr;
	    }
	
	    public void setTotalPageNbr(int totalPageNbr) {
	        this.totalPageNbr = totalPageNbr;
	    }
	
	    public BookMark getBookMark() {
	        return bookMark;
	    }
	
	    public void setBookMark(BookMark bookMark) {
	        this.bookMark = bookMark;
	    }
	
	    public static class BookMark implements Serializable {
	        private int currPage;
	        private int currLine;
	
	        public int getCurrPage() {
	            return currPage;
	        }
	
	        public void setCurrPage(int currPage) {
	            this.currPage = currPage;
	        }
	
	        public int getCurrLine() {
	            return currLine;
	        }
	
	        public void setCurrLine(int currLine) {
	            this.currLine = currLine;
	        }
	    }
	}
	
&emsp;&emsp;单元测试：  
	
	package com.example.clone;
	
	import org.junit.Assert;
	import org.junit.Test;
	import org.junit.runner.RunWith;
	import org.springframework.test.context.junit4.SpringRunner;
	
	@RunWith(SpringRunner.class)
	public class DeepCloneTest {
	
	    @Test
	    public void testDeepCloneUseSerializationUtils() {
	        Book book = new Book("白夜行", 521);
	        book.setBookMarkInfo(128, 11);
	        Book clonedBook = CloneUtils.deepClone(book);
	        clonedBook.setBookMarkInfo(129, 22);
	        Assert.assertNotEquals(book.getBookMark().getCurrPage(), clonedBook.getBookMark().getCurrPage());
	    }
	}
        
&emsp;&emsp;上面单元测试并未抛出异常，表示book和clonedBook是两个不同的对象。测试通过~

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
