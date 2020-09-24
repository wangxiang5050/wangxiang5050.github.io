---
layout: post
title:  "ArrayList"
date:   2018-07-31 09:52:55 +0800
categories: java collections
tags:
    - java
    - ArrayList
---

&emsp;&emsp;最近在阅读java集合框架源代码，确实在阅读源码之后对API的理解又深了一层，相较死记API用法，我更愿意阅读源码来获取更深刻的记忆。而另外一个收获则是知道了什么时候用什么集合，比如要实现排序、去重可以用TreeSet/TreeMap实现（红黑树）等等。那么废话不多说，先让我们看看ArrayList的实现吧。  

### ArrayList

&emsp;&emsp; 这可能是我用的最多的集合实现类了，从名字可以看出它是用数组实现的List，所以就具有数组的特性，有较好的查找效率，可以通过index完成数据访问，但删除操作性能开销较大。

#### 构造方法
	
	transient Object[] elementData;
	private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
	
	public ArrayList() {
	    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
	}

&emsp;&emsp; 可以看到实际存储数据就是用的elementData这个Object数组，当调用无参构造时，将elementData指向一个空数组，完成实例化。

#### add(E e)

&emsp;&emsp; add方法向elementData中添加数据，如果添加成功则使实例变量size++。其中较为复杂的是以下这段扩容代码：

	private void grow(int minCapacity) {
	    // overflow-conscious code
	    int oldCapacity = elementData.length;
	    int newCapacity = oldCapacity + (oldCapacity >> 1);
	    if (newCapacity - minCapacity < 0)
	        newCapacity = minCapacity;
	    if (newCapacity - MAX_ARRAY_SIZE > 0)
	        newCapacity = hugeCapacity(minCapacity);
	    elementData = Arrays.copyOf(elementData, newCapacity);
	}
	
&emsp;&emsp; 在经过判断逻辑确定了newCapacity后使用Arrays.copyOf进行扩容。所以实质还是进行了一次数组扩容。  
&emsp;&emsp; 上述代码中出现了位运算，oldCapacity >> 1，效果是将oldCapacity右移1位，下面用代码详细解释位运算操作：

	@Test
	public void test_bitOperator() {
	    int i = 15;
	    System.out.println(Integer.toBinaryString(i)); // 1111
	    System.out.println(Integer.toBinaryString(i >> 1)); // 111
	}  
	
&emsp;&emsp; 可以看到15的二进制表示是1111，右移一位过后是111，转换为十进制为7。所以正整数x的 >> 1在结果上相当于x/2向下取整。  
#### remove(Object o)  
&emsp;&emsp; 实质就是迭代elementData，找到与o相等的元素，将其删除并移动数组。实际的删除代码如下：
	
	private void fastRemove(int index) {
	    modCount++;
	    int numMoved = size - index - 1;
	    if (numMoved > 0)
	        System.arraycopy(elementData, index+1, elementData, index,
	                         numMoved);
	    elementData[--size] = null; // clear to let GC do its work
	}

&emsp;&emsp; 实质是使用System.arraycopy完成数组移动。需要注意的是最后一行，数组移动后，将最后一个元素置为null,此时GC才能够回收它，避免内存泄漏。  
#### 遍历  
&emsp;&emsp; java5增加的foreach其实质是语法糖，代码编译后，依然使用的是迭代器。  
&emsp;&emsp; 值得一提的是java8中Iterator接口新增了默认方法forEachRemaining，之后我们也可以直接用这个方法完成遍历，下面是一个例子：

	List<String> list = new ArrayList<>();
	list.add("wx");
	list.add("lyj");
	list.iterator().forEachRemaining(s -> System.out.println(s)); // 依次输出 wx lyj

### 总结
> 1. ArrayList实质是一个数组，所以需要连续的存储空间，有较快的查找效率，但删除效率较低。  
> 2. java8后可以使用Iterator.forEachRemaining完成迭代。

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
