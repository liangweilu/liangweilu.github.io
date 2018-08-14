---
layout: post
title: HashMap和Hashtable的区别
date: 2018-08-07
tags: Java基础
---
### 概述

&emsp;&emsp;HashMap和Hashtable的的比较在面试中应该比较常见的问题，这考验了我们
对常用集合的掌握程度，也考察了我们在开发中是否能正确的使用集合类。网上有很多的博文对这个问题进行了阐述，我这里只是作为一个自己的总结，也希望自己能尽量详尽，深入
的比较两者的差异。其实集合的比较无非就是从效率，安全性，接口，实现的算法等方面进行比较，下表是我在学习过程中总结出来的异同点，有些不宜对比的展示的特点，将在下文中进行说明和解释。表中的内容不需要记，理解了下面的原理，自然就明白了。     

 \\| HashMap  | Hashtable |
- | - |
效率 | 高 | 低
线程安全性 | 线程不安全 | 线程安全  
null Key | 支持 | 不支持
null Value | 支持 | 不支持
初始容量 | 16 | 11
扩充规则 | 原来的2n | 原来的2n + 1
数据结构 | hash表 | hash表
产生时间 | JDK1.2 | JDK1.0
父类 | AbstructMap | Dictionary ( 已经弃用了 )
实现 | Map、Cloneable、Serializable | Map、Cloneable、Serializable
迭代器 | fail-fast | enumerator

### 存储结构
&emsp;&emsp;HashMap和Hashtable都是使用哈希表来存储键值对，数据结构基本一致，都创建了一个实现了`Map.Entry<K,V>`内部类`Entry<K,V>`，但是在Java8中HashMap的内部类的名字变成了`Node<K,V>`。但是数据结构仍是一样的，Entry<K,V>是一个链表结构，用来存储键值对。其结构如下所示：  
```Java
static class Entry<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Entry<K,V> next;
        //......省略
      }
```  
HashMap和Hashtable中有多少个键值对，就有多少个`Entry<K,V>`对象，其存储结构如下图所示：  
![](https://byeluliangwei.github.io/images/java/hash bucket.png)  

&emsp;&emsp;这是一个bucket（哈希桶）容量为8，也称为hash数组，大小为5（存了5个键值对）的HashMap/Hashtable内部存储情况。我们可以清晰的看到其内部创建了一个`Entry<K,V>`引用的数组，用来表示哈希表，数组的长度，就是哈希桶的数量。数组的每一个元素都是一个`Entry<K,V>`引用，而从`Entry<K,V>`的属性我们也可以看出，它是一个单链表结构，每个`Entry<K,V>`包含着下一个键值对的引用。  

### 实现原理

&emsp;&emsp;关于Hashtable中不允许Null Key 和Null Value我们从源码中便可以看出，如果是value为空的话，会抛出空指针异常，如果是key为空的话，在计算hashCode的时候就会报空指针异常了，也就是下面的`key.hashCode()`会报异常，部分Hashtable源码如下：
```Java
public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
```

HashMap中，null可以作为Key，但是这样的键只有一个，可以有一个或者多个键对应的value为空。当使用`get()`方法获取返回null时，可能是该HashMap中没有key，也可能是对应的null为空，因此HashMap中不能用`get()`方法来判断是否具有某个key，而应该用`containsKey()`来判断。如果key为null，则直接从哈希表的第一个位置table[0]对应的链表上查找。记住，key为null的键值对永远都放在以table[0]为头结点的链表中，当然不一定是存放在头结点table[0]中。对于每次插入新的节点，都是插入到单链表的头结点的位置。    
&emsp;&emsp;Hashtable默认的初始大小为11，之后每次扩充为原来的2n+1。HashMap默认的初始化大小为16，之后每次扩充为原来的2倍。如果在创建时给定了初始化大小，那么Hashtable会直接使用你给定的大小，而HashMap会将其扩充为2的幂次方大小。  
&emsp;&emsp;也就是说Hashtable会尽量使用素数、奇数。而HashMap则总是使用2的幂作为哈希表的大小。当哈希表的大小为素数时，简单的取模哈希的结果会更加均匀，所以单从这一点上看，Hashtable的哈希表大小选择，似乎更高明些。但另一方面我们又知道，在取模计算时，如果模数是2的幂，那么我们可以直接使用位运算来得到结果，效率要大大高于做除法。所以从hash计算的效率上，又是HashMap更胜一筹。  
&emsp;&emsp;对于hash表中出现hash冲突的情况，采用了链表结构来存储键值对，也就是上面存储结构中所说的，对于hashCode相同的，会将键值对以头插法的方式插入链表。对于当hashCode相同时，我们的取值方式也就很明显了，通过hashCode找到找到bucket的位置，由于存储了key和value，然后遍历链表即可访问到我们想获取的值。        
&emsp;&emsp;就是HashMap和Hashtable在计算hash时都用到了一个叫hashSeed的变量。这是因为映射到同一个hash桶内的Entry对象，是以链表的形式存在的，而链表的查询效率比较低，所以HashMap/Hashtable的效率对哈希冲突非常敏感，所以可以额外开启一个可选hash（hashSeed），从而减少哈希冲突。这个优化在JDK 1.8中已经去掉了，因为JDK 1.8中，映射到同一个哈希桶（数组位置）的Node对象，当链表长度大于8之后，会转换成红黑树，从而大大加速了其查找效率。  

### 总结

&emsp;&emsp;由于Hashtable中的父类已经被弃用了，所以一般情况下我们也没有适用Hashtable，对于单线程情况下，通常是使用的HashMap。但是当多线程环境下，我们需要线程安全的键值对集合是，可以使用`ConCurrentHashMap`。关于HashMap和ConcurrentHashMap本来我是打算再重新写一篇的，但是看了这篇[HashMap与ConcurrentHashMap详细讲解](https://mp.weixin.qq.com/s/wNmAi1FICNu7rkmCe1GDyw)之后，我感觉已经没必要写了，直接点击链接进去看便会一目了然。

### 参考
Hashtable源码剖析：<https://blog.csdn.net/ns_code/article/details/36191279>  
HashMap源码剖
：<http://blog.csdn.net/ns_code/article/details/36034955>
