---
layout:     post
title:      "Java常用容器"
date:       2017-08-30 12:00:00
author:     "ShiYu"
catalog: true
tags:
    - Java
---
#java容器

>利用了一个月时间，浏览完了《thinking in java》，俗话说好记性不如烂笔头，何况我的记性并不好，从今天开始，将不定期将从书中学习到的知识整理出来，今天首先整理一下util包内各种类型容器之间的关系，及每种容器的应用场景，下周将逐步详细整理每种容器的具体实现。

util包内，不同类型容器之间对关系如下图:
![容器类图](http://139.224.133.147/dist/images/collection.png)

圆形代表接口，矩形代表类，虚线代表对接口的实现，实线代表继承。

通过上图可以看出，java容器分为两大类，一类是普通的对象持有对象容器，全部实现自Collection接口，另一类是key-value形式的对象持有容器，全部实现自Map接口。

## Collection

Collection接口有3个直接子接口:

* List: 承诺可以将元素维护在特定的序列中，List接口在Collection的基础上添加了大量方法，使得可以在List的中间插入和移除元素。
* Set: Set不保存重复的元素，如果你试图将相同对象的多个实例添加到Set中，那么它会阻止这种重复现象。Set具有与Collection完全一样的方法，因此并没有额外的功能。
* Queue:队列是典型的先进先出（FIFO)的容器，即从容器一端放入事物，从另一端取出。

### List
抛去抽象类不说，List拥有两个具体的实现类，ArrayList和LinkedList，其中ArrayList底层用数组维护，所以它擅长随机访问元素，但是在List中间插入和删除操作则会很慢。LinkedList底层用链表维护，所以它不擅长随机访问，但是对于在中间插入和删除的性能则比ArrayList强大很多。所以业务场景中，随机访问比较频繁时，使用ArrayList,在List中间频繁添加或删除时，则使用LinkedList。

### Set

Set有4个具体实现类，其中HashSet使用的最频繁，它专门对快速查找做了优化，擅长查找元素，所以在没有排序需求的情况下，应该使用HashSet。TreeSet将元素存储在红黑树数据结构中，TreeSet按照比较结果的升序保存元素，LinkedHashSet按照元素被添加的顺序保存元素，所以对于有排序需求的业务场景，可以使用TreeSet或者LinkedHashSet。EnumSet是一个与枚举类型一起使用的专用Set实现，EnumSet中所有元素必须来自单个枚举类型。

### Queue

Queue有一个子接口Deque，它代表的是双向队列，该队列两端元素既能入队也能出队，所以当需要使用到双向队列的时候，应该食用直接实现它的ArrayDeque类。此外Queue接口有两个直接实现它的具体类，LinkedList和PriorityQueue，LinkedList实现类Queue接口，它代表了最典型的先进先出队列，PriorityQueue是优先级队列，它弹出的元素是最需要的元素，所以当没有优先级需求的时候直接使用LinkedList,否则使用PriorityQueue。

## Map

Map通过key-value持有对象，它有4个实现类，与HashSet一样，HashMap也是为查询速度设计的实现类，当没有排序需求时应该使用HashMap,TreeMap按照比较结果的升序保存键，LinkedHashMap按照插入顺序保存键，EnumMap则是专为枚举设计的一个实现类。所以当有排序需求时，根据具体需求选择TreeMap或者LinkedHashMap，当键值为枚举时，则使用EnumMap。

util包提供的容器工具类就是这些，今天梳理到这里，明天梳理HashMap的具体实现。