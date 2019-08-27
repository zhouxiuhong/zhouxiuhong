---
title:第9讲-对比Hashtable、HashMap、TreeMap有什么不同？
date: 2019-06-09 11:17:05
tags: 
     - JAVA 
     - Java核心技术36讲
toc: true
---

**对比Hashtable、HashMap、TreeMap有什么不同？**

极客时间版权所有: https://time.geekbang.org/column/article/7810

**典型回答**

Hashtable、HashMap、TreeMap都是最常见的一些Map实现，是以 **键值对** 的形式存储和操作数据的容器类型。

Hashetable是早期Java类库提供的一个 [哈希表](https://zh.wikipedia.org/wiki/%E5%93%88%E5%B8%8C%E8%A1%A8) 实现，本身是同步的，不支持null键和值，由于同步导致的性能开销，所以已经很少被推荐使用。

HashMap是应用更加广泛的哈希表实现，行为上大致与HashTable一致，主要区别在于HashMap不是同步的，支持null键和值等。通常情况下，HashMap进行put或者get操作，可以达到常数时间的性能，所以 **它的绝大部分利用键值对存取场景的首选** ，比如，实现一个用户ID和用户信息对应的运行时存储结构。



TreeMap则是基于红黑树的一种提供顺序访问的Map，和HashMap不同，它的get、put、remove之类操作都是O（log（n））的时间复杂度，具体顺序可以由指定的Comparator来决定，或者根据键的自然顺序来判断。





**考点分析**

上面的回答，只是对一些基本特征的简单总结，针对Map相关可以扩展的问题很多，从各种数据结构、典型应用场景，到程序设计实现的技术考量，尤其是在java8里，HashMap本身发生了非常大的变化，这些都是经常考察的方面。

- 理解Map相关类似整体结构，尤其是有序数据结构的一些要点。
- 从源码去分析HashMap的设计和实现要带你，理解容量、负载因子等，为什么需要这些参数，如何影响Map的性能，实践中如何取舍等。
- 理解树化改造的相关原理和改进原因。

除了典型的代码分析，还有一些有意思的并发相关问题也经常会被提到，如HashMap在并发环境可能出现 [无限循环占用CPU](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6423457) 、size不准确等诡异的问题。

我认为者是一种典型的使用错误，因为HashMap明确声明不是线程安全的数据结构，如果忽略这一点，简单用在多线程场景里，难免会出现问题。

理解导致这种错误的原因，也是深入理解并发程序运行的好办法。对于具体发生了什么，你可以参考这篇很久以前的 [分析](http://mailinator.blogspot.com/2009/06/beautiful-race-condition.html) ，里面甚至提供了示意图，我就不在重复别人写好的内容了。



**知识扩展**

1. Map整体结构

   首先，我们想对Map相关类型有个整体了解，Map虽然通常被包括在Java集合框架里，但是其本身并不是侠义上的集合类型（Collection），具体你可以参考下面这个简单类图。

   ![](/picture/content/java36-9.png)

   Hashtable比较特别，作为类似Vector、Stack的早期集合相关类型，它是扩展了Dictionary类的，类结构上与HashMap之类明显不同。

   HashMap等其他Map实现则是都扩展了AbstractMap，里面包含了通用方法抽象。不同Map的用途，从类图结构就能体现出来，设计目的已经体现在不同接口上。

   大部分使用Map的场景，通常就是放入、访问或者删除，而对顺序没有特别要求，HashMap在这种情况下基本是最好的选择。 **HashMap的性能表现非常依赖于哈希码的有效性，请务必掌握hashCode和equals的一些基本约定，** 比如

   - equals相等，hashCode一定要相等。
   - 重写了hashCode也要重写equals。
   - hashCode需要抱持一致性，状态改变返回的哈希值仍然要一致。
   - equals的对称、反射、传递等特性。

   针对有序Map的分析内容比较有限，我再补充一些，虽然LinkedHashMap和TreeMap都可以保证某种顺序，但二者还是非常不同的。

   - LinkedHashMap通常提供的是遍历顺序符合插入顺序，它的实现是通过为条目（键值对）维护一个双向链表。注意，通过特定构造函数，我们可以创建反映访问顺序的实例，所谓的put、get、compute等，都算作“访问”。

   这种行为适用于一些特定应用场景，例如，我们构建一个空间占用敏感的资源池，希望可以自动将最不常被访问的对象释放掉，这就可以利用LinkedHashMap提供的机制来实现，参考下面的示例：

   ```、
   import java.util.LinkedHashMap;
   import java.util.Map;  
   public class LinkedHashMapSample {
       public static void main(String[] args) {
           LinkedHashMap<String, String> accessOrderedMap = new LinkedHashMap<String, String>(16, 0.75F, true){
               @Override
               protected boolean removeEldestEntry(Map.Entry<String, String> eldest) { // 实现自定义删除策略，否则行为就和普遍 Map 没有区别
                   return size() > 3;
               }
           };
           accessOrderedMap.put("Project1", "Valhalla");
           accessOrderedMap.put("Project2", "Panama");
           accessOrderedMap.put("Project3", "Loom");
           accessOrderedMap.forEach( (k,v) -> {
               System.out.println(k +":" + v);
           });
           // 模拟访问
           accessOrderedMap.get("Project2");
           accessOrderedMap.get("Project2");
           accessOrderedMap.get("Project3");
           System.out.println("Iterate over should be not affected:");
           accessOrderedMap.forEach( (k,v) -> {
               System.out.println(k +":" + v);
           });
           // 触发删除
           accessOrderedMap.put("Project4", "Mission Control");
           System.out.println("Oldest entry should be removed:");
           accessOrderedMap.forEach( (k,v) -> {// 遍历顺序不变
               System.out.println(k +":" + v);
           });
       }
   }
   ```

```
- ​       对于TreeMap，它的整体顺序是由键的顺序关系决定的，通过Comparator或Comparable（自然顺序）来决定。

我在上一讲留给你的思考题提到了，构建一个具有优先级的调度系统的问题，其本质就是个典型的优先队列场景，Java标准库提供了基于二叉堆实现的PriorityQueue，它们都是依赖于同一种排序机制，当然也包括TreeMap的马甲TreeSet。

类似hashCode和equals的约定，为了避免摸棱两可的情况，自然顺序同样需要符合一个约定，就是compareTo的返回值需要和equals一致，否则就会出现摸棱两可情况。

我们可以分析TreeMap的put方法实现：
```

public V put(K key, V value) {

```
Entry<K,V> t = …
cmp = k.compareTo(t.key);
if (cmp < 0)
    t = t.left;
else if (cmp > 0)
    t = t.right;
else
    return t.setValue(value);
    // ...
```

   }

```


从代码里，你可以看出什么呢？当我不遵守约定时，两个不符合唯一性（equals）要求的对象被当作是同一个（因为，compareTo返回0），这会导致歧义的行为表现。

2. HashMap源码分析

   前面提到，HashMap设计与实现是个非常高频的面试题，所以我会在这进行相对详细的源码解读，主要围绕：

   - HashMap内部实现基本点分析。
   - 容量（capacity）和负载系数（load factor）。
   - 树化。

   首先，我们一起看看HashMap内部的结构，它可以看作是数组（Node<K, V>[] table）和链表结合组成的复合结构，数组被分为一个个桶（bucket），通过哈希值决定了键值对在这个数组的寻址；哈希值相同的键值对，则以链表形式存储，你可以参考下面的示意图。这里需要注意的是，如果链表大小超过阙值（TREEIFY_THRESHOLD,8）,图中的链表就会被改造为树形结构。

   ![](/picture/content/java36-9-2.png)

   从非拷贝构造函数的实现来看，这个表格（数组）似乎并没有在最初就初始化好，仅仅设置了一些初始值而已。
```

   public HashMap(int initialCapacity, float loadFactor){  

```
   // ... 
   this.loadFactor = loadFactor;
   this.threshold = tableSizeFor(initialCapacity);
```

   }

```
   所以，我们深刻怀疑，HashMap也许是按照lazy-load原则，在首次使用时被初始化（拷贝构造函数除外， 我这里仅介绍最通用的场景）。既然如此，我们去看看put方法实现，似乎只有一个putVal的调用：
```

   public V put(K key, V value) {

```
   return putVal(hash(key), key, value, false, true);
```

   }

```
​       看来主要的密码似乎藏在putVal里面，到底有什么秘密呢？为了节省空间，我这里只截取了putVal比较关键的几部分。

​     
```

final V putVal(int hash, K key, V value, boolean onlyIfAbent,

```
           boolean evit) {
Node<K,V>[] tab; Node<K,V> p; int , i;
if ((tab = table) == null || (n = tab.length) = 0)
    n = (tab = resize()).length;
if ((p = tab[i = (n - 1) & hash]) == ull)
    tab[i] = newNode(hash, key, value, nll);
else {
    // ...
    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for first 
       treeifyBin(tab, hash);
    //  ... 
 }
```

}

```
从putVal方法最初的几行，我们就可以发现几个有意思的地方：

- 如果表格是null，resize方法会负责初始化它，这从tab=resize（）可以看出。

- resize方法兼顾两个职责，创建初始存储表格，或者在容量不满足需求的时候，进行扩容（resize）。

- 在放置性的键值对的过程中，如果发生下面条件，就会发生扩容。
```

  if (++size > threshold)

```
  resize();
```

```
**一课一练**




```

