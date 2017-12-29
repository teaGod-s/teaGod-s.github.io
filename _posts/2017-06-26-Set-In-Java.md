---
layout:     post                       # 使用的布局（不需要改）
title:      聊聊Java中的Set            # 标题 
subtitle:   《Java语言程序设计进阶篇》读书笔记 # 副标题
date:       2017-06-26                 # 时间
author: teaGod # 作者
header-img: img/post-bg-java.jpg     # 这篇文章标题背景图片
catalog: true                        # 是否归档
tags:                                # 标签
    - Java
    - 读书
    - 程序员
featured-tags: true  
featured-condition-size: 1     # A tag will be featured if the size of it is more than this condition value
disqus_username: disqus_r5ccOvlSVU
---

Set是java集合框架中集合的一种。

java集合包括List，Set还有Queue。我们今天的主角就是Set。

下图就是java集合框架的继承关系
![java集合框架.png](http://upload-images.jianshu.io/upload_images/2946411-38cebe430db15aad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Set区别于另外两个集合的最大特点就是，它不允许有重复元素。

AbstractSet扩展了AbstractCollection抽象类，并且实现了set接口，重写了equals方法和hashCode方法。

Set接口的三个具体类是：HashSet，LinkedHashset和TreeSet。

# HashSet
你可以用Hashset默认的无参构造方法来创建一个空的HashSet，也可以由一个现有的集合创建一个HashSet。
默认情况下，Hashset的capacity是16，load factor是0.75。
其中load factor是一个0.0~1.0之间的值，当hashset中的元素数量超过capacity * load factor时，capacity就会翻倍。

HashSet中的元素是无序的，当你要遍历一个set时，你可以用for each循环或者使用迭代器，迭代器的写法如下:

`Iterator<T> iterator = set.iterator();`

# LinkedHashSet
LinkedHashSet使用一个链表实现来扩展HashSet类，它支持对HashSet中的元素排序，默认情况下是按照元素的插入顺序排序。

所以，如果你不需要维护元素被插入的顺序，你就应该使用HashSet，它会比LinkedHashSet高效的多。

# TreeSet
TreeSet实现了SortedSet接口的一个具体类，SortedSet是Set的一个子接口，顾名思义，它里面的元素都是有序的。

只要对象是可以比较的，我们就可以把它们添加到TreeSet中。那么我们怎么定义TreeSet中元素的顺序呢？有两种方法：

- 使用Comparable接口。由于添加到TreeSet中的对象都是Comparable的实例，所以可以使用comparTo方法对它们进行比较。这种方法定义的顺序被称为*自然顺序(natural order)*。

- 给Set中的元素定义一个比较器(Comparator)。
你需要创建一个实现Comparator接口的类，重写接口中两个方法：compare和equals。而且，定义的这个比较器最好实现Serializable接口，以使数据结构能够成功序列化。接下来就是使用TreeSet的构造方法 `TreeSet(Comparator comparator)` 来创建一个TreeSet。
好了，大功告成，通过这种方法定义的顺序称为*比较器顺序(order by comparator)*。

>![teaGodyeah](/img/teaGodyeah.jpg)
