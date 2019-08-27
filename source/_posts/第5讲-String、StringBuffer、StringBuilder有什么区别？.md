---
title: 第5讲 | String、StringBuffer、StringBuilder有什么区别？
date: 2019-06-05 11:17:05
tags: 
     - JAVA 
     - Java核心技术36讲
toc: true
---

**理解java的字符串，String、StringBuffer、StringBuilder有什么区别？**

**典型回答**

String是Java语言非常基础和重要的类，提供了构造和管理字符串的各种基本逻辑。它是典型的Immutable类，被声明成为final class，所有属性也都是final的。也由于它的不可变性，类似拼接、剪裁字符串等动作，都会产生新的String对象。由于字符串操作的普遍性，所以相关操作的效率往往对应用性能有明显影响。



StringBuffer是为解决上面提到拼接产生太多中间对象的问题而提供的一个类，我们可以用append或者add方法，把字符串添加到已有序列的末尾或者指定位置。StringBuffer本质是一个线程安全的可修改字符序列，它保证了线程安全，也随之带来了额外的性能开销，所以除非有线程安全的需要，不然还是推荐使用它的后继者，也就是StringBuilder。



StringBuiler是java1.5中新增的，在能力上和StringBuffer没有本质区别，但是它去掉了线程安全的部分，有效减小了开销，是绝大部分情况下进行字符串拼接的首选。

**考点分析**

几乎所有的应用开发都离不开操作字符串，理解字符串的设计和实现以及相关工具如拼接类的使用，对写出高质量代码是非常有帮助的。关于这个问题，我前面的回答是一个通常的该药性回答，至少你要指定String是Immutable的，字符串操作不当可能会产生大量临时字符串，以及线程安全方面的区别。

如果继续深入，面试官可以从各种不同的角度考察，比如可以：
（1）通过String和相关类，考察基本的线程安全设计与实现，各种基础变成实践。

（2）考察JVM对象缓存机制的理解以及如何良好地使用。

（3）考察JVM优化Java代码的一些技巧。

（4）String相关类的演进，比如java9中实现的巨大变化。

.........



**知识扩展**

1.字符串设计和实现考量

String是Immutable类的典型实现，原生的保证了基础线程安全，因为你无法对它内部数据进行任何修改，这种便利甚至体现在拷贝构造函数中，由于不可变，Immutable对象在拷贝时不需要额外复制数据。



我们再来看看StringBuffer实现的一些细节，它的线程安全是通过把各种修改数据的方法都加上synchronization关键字实现的，非常直白。其实，这种简单粗暴的实现方式，非常适合我们常见的线程安全类实现，不必纠结于synchronized性能之类的，有人说“过早优化是万恶之源”，考虑可靠性、正确性和代码可读性才是大多数应用开发最重要的因素。



为了实现修改字符串序列的目的，StringBuffer和StringBuilder底层都是利用可修改的（char， JDK9以后是byte）数组，二者都继承了AbstractStringBuilder，里面包含了基本操作，区别仅在于最终的方法是否加了synchronized。



另外，这个内部数组应该创建成多大的呢？如果太小，拼接的时候可能要重新创建足够大的数组；如果太大，又会浪费空间。目前的实现是，构建时初始字符串长度加16（这意味着，如果没有构建对象时输入最初的字符串，那么初始值就是16）。我们如果确定拼接会发生非常多次，而且大概是可预计的，那么久可以指定合适的大小，避免很多次扩容的开销。扩容会产生多重开销，因为要抛弃原有数组，创建新的（可以简单认为是倍数）数组，还要进行arraycopy。

在没有线程安全问题的情况下，全部拼接操作是应该都用StringBuilder实现吗？毕竟这样书写的代码，还是要敲很多字的，可读性也不理想，下面的对比非常明显。

```
String strByBuilder  = new
StringBuilder().append("aa").append("bb").append("cc").append
            ("dd").toString();
             
String strByConcat = "aa" + "bb" + "cc" + "dd";

```

其实，在通常情况下，没有必要过于担心，要相信java还是非常智能的。

我们来做个实验，把下面一段代码，利用不同版本的JDK编译，然后再反编译，例如：

```
public class StringConcat {
     public static String concat(String str) {
       return str + “aa” + “bb”;
     }
}

```

先编译再反编译，比如使用不同版本的JDK：

```
${JAVA_HOME}/bin/javac StringConcat.java
${JAVA_HOME}/bin/javap -v StringConcat.class

```

JDK8的输出片段是：

```
         0: new           #2                  // class java/lang/StringBuilder
         3: dup
         4: invokespecial #3                  // Method java/lang/StringBuilder."<init>":()V
         7: aload_0
         8: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        11: ldc           #5                  // String aa
        13: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        16: ldc           #6                  // String bb
        18: invokevirtual #4                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        21: invokevirtual #7                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;

```

而在JDK9中，反编译的结果就会有点特别了，片段是：

```
         // concat method
         1: invokedynamic #2,  0              // InvokeDynamic #0:makeConcatWithConstants:(Ljava/lang/String;)Ljava/lang/String;
         
         // ...
         // 实际是利用了 MethodHandle, 统一了入口
         0: #15 REF_invokeStatic java/lang/invoke/StringConcatFactory.makeConcatWithConstants:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;

```

你可以看到，非静态的拼接逻辑在JDK8中会自动被javac转换为StringBuilder操作；而在JDK9里面，则是体现了思路的变化。java9利用InvokeDynamic，将字符串拼接的优化与javac生成的字节码解耦，假设未来JVM增强相关运行时实现，将不需要依赖javac的任何修改。

在日常编程中，保证程序的可读性、可维护性，往往比所谓的最优性能更重要，你可以根据实际需求酌情选择具体的编码方式。

2.字符串缓存

我们粗略统计过，把常见应用进行堆转储（Dump Heap），然后分析对象组成，会发现平均25%的对象是字符串，并且其中约半数是重复的。如果能避免创建重复字符串，可以有效降低内存消耗和对象创建开销。

String在Java6以后提供了intern（）方法，目的是提示JVM把相应字符串缓存起来，以备重复使用。在我们创建字符串对象并调用intern（）方法的时候，如果已经有缓存的字符串，就会返回缓存里的实例，否则将其缓存起来。一般来说，JVM会将所有的类似“abc”这样的文本字符串，或者字符串常量之类缓存起来。

看起来不错是吧？但实际情况估计会让你大跌眼镜。一般使用java6这种历史版本，并不推荐大量使用intern，为什么呢？魔鬼存在于细节中，被缓存的字符串是存在所谓PermGen里的，也就是臭名昭著的“永久代”，这个空间是很有限的，也基本不会被FullGC之外的垃圾收集照顾到。所以，如果使用不当，OOM就会光顾。

在后续版本中，这个缓存被放置在堆中，这样就极大避免了永久代占满的问题，甚至永久代在JDK8中被MetaSpace（元数据区）替代了。而且，默认缓存大小也在不断地扩大中，从最初的1009，到7u40以后被修改为60013。你可以使用下面的参数直接打印具体数字，可以拿自己的JDK立刻试验一下。

```
-XX:+PrintStringTableStatistics

```

你也可以使用下面的JVM参数手动调整大小，但是绝大部分情况下并不需要调整，除非你确定它的大小已经影响了操作效率。

```
-XX:StringTableSize=N

```

Intern是一种  **显式地排重机制**，但是它也有一定的副作用，因为需要开发者写代码时明确调用，一是不方便，每一个都显式调用是非常麻烦的；另外就是我们很难保证效率，应用开发阶段很难清除地预计字符串的重复情况，有人认为这是一种污染代码的实践。

幸好在Oracle JDK 8u之后，推出了一个新的特性，也就是G1 GC下的字符串排重。它是通过将相同数据的字符串指向同一份数据来做到的，是JVM底层的改变，并不需要Java类库做什么修改。

注意这个功能目前是默认关闭的，你需要使用下面参数开启，并且记得指定使用G1 GC:

```
-XX:+UseStringDeduplication


```

前面说到的几个方面，只是Java底层对字符串各种优化的一角，在运行时，字符串的一些基础操作会直接利用JVM内部的Intrinsic机制，往往运行的就是特殊优化的本地代码，而根本就不是Java代码生成的字节码。Intrinsic可以简单理解为，是一种利用native方式hard-coded的逻辑，算是一种特别的内联，很多优化还是需要直接使用特定的CPU指令，具体可以看相关源码，搜索“string”以查找相关Intrinsic定义。当然，你也可以在启动实验应用时，使用下面参数，了解intrinsic发生的状态。

[http://hg.openjdk.java.net/jdk/jdk/file/44b64fc0baa3/src/hotspot/share/classfile/vmSymbols.hpp]: 	&amp;amp;quot;源码&amp;amp;quot;

```
-XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining
    // 样例输出片段    
        180    3       3       java.lang.String::charAt (25 bytes)  
                                  @ 1   java.lang.String::isLatin1 (19 bytes)   
                                  ...  
                                  @ 7 java.lang.StringUTF16::getChar (60 bytes) intrinsic 

```

可以看出，仅仅是字符串一个实现，就需要Java平台工程师和科学家付出如此大且默默无闻的努力，我们得到的很多便利都是来源于此。

3.String自身的演化



如果你仔细观察过Java的字符串，在历史版本中，它是使用char数组来存数据的，这样非常直接。但是Java中的char是两个bytes大小，拉丁语系语言的字符，根本就不需要太宽的char，这样无区别的实现就造成了一定的浪费。密度是编程语言平台永恒的话题，因为归根结底绝大部分任务是要来操作数据的。

其实在Java6的时候，Oracle JDK就提供了压缩字符串的特性，但是这个特性的实现并不是开源的，而且在实践中也暴露出了一些问题，所以在最新的JDK版本中已经将它移除了。

在Java9中，我们引入了Compact Strings的设计，对字符串进行了大刀阔斧的改进。将数据存储方式从char数据，改变为一个byte数组加上一个标识编码的所谓coder，并且将相关字符串操作类都进行了修改。另外，所有相关的Intrinsic之类也都进行了重写，以保证没有任何性能损失。

虽然底层实现发生了这么大的改变，但是Java字符串的行为并没有任何大的变化，所以这个特性对于绝大部分应用来说是透明的，绝大部分情况不需要修改已有代码。

当然，在极端情况下，字符串也出现了一些能力退化，比如最大字符串的大小。你可以思考下，原来char数组的实现，字符串的最大长度就是数组本身的长度限制，但是替换成byte数组，同样数组长度下，存储能力是退化了一倍的！还好这是存在于理论中的极限，还没有发现现实应用受此影响。

在通用的性能测试和产品实验中，我们能非常明显地看到紧凑字符串带来的优势， **即更小的内存占用、更快的操作速度**。

今天我从String、StringBuffer和StringBuilder的主要设计喝实现特点开始，分析了字符串缓存的intern机制、非代码侵入性的虚拟机层面排重、Java9中紧凑字符的改进，并且初步接触了JVM的底层优化机制intrinsic。从实践的角度，不管是Compact Strings还是底层intrinsic优化，都说明了使用Java基础类库的优势，它们往往能够得到最大程度、最高质量的优化，而且只要升级JDK版本，就能零成本地享受这些益处。





**学习心得**

1.String

(1)String的创建机理

由于String在java世界中使用过于频繁，Java为了避免在一个系统中产生大量的String对象，引入了字符串常量池。其运行机制是：创建一个字符串时，首先检查池中是否有值相同的字符串对象，如果有则不需要创建直接从池中刚查找到的对象引用；如果没有则新建字符串对象，返回对象引用，并且将新创建的对象放入池中。但是，通过new方法创建的String对象是不检查字符串池的，而是直接在堆区或栈区创建一个新的对象，也不会把对象放入池中。上述原则只适用于通过直接量给String对象引用赋值的情况。

举例：String str1 = “123”; //通过直接量赋值方式，放入字符串常量池

String str2 = new String("123"); //通过new方式赋值方式，不放入字符串常量池

注意：String提供了inter()方法。调用该方法时，如果常量池中包括了一个等于此String对象的字符串（由equals方法确定），则返回池中的字符串。否则，将此String对象添加到池中，并且返回池中对象的引用。



（2）String的特性

[A]不可变。是指String对象一旦生成，则不能再堆它进行改变。不可变的主要作用在于当一个对象需要被多线程共享，并且访问频繁时，可以省略同步和锁等待的时间，从而大幅度提高系统性能。不可变模式是一个可以提高多线程程序的性能，降低多线程程序复杂度的设计模式。

[B]针对常量池的优化。当2个String对象拥有相同的值时，他们只引用常量池中的同一个拷贝。当同一个字符串反复出现时，这个技术可以大幅度节省内存空间。



2.StringBuffer/StringBuilder

StringBuffer和StringBuilder都实现了AbstractStringBuilder抽象类，拥有几乎一致对外提供的调用接口；其底层在内存中的存储方式与String相同，都是以一个有序的字符序列（char类型的数组）进行存储，不同点是StringBuffer/StringBuilder对象的值是可以改变的，并且值改变以后，对象引用不会发生改变；两者对象在构造过程中，首先按照默认大小申请一个字符数组，由于会不断加入新数据，当超过默认大小后，会创建一个更大的数组，并将原先的数组内容复制过来，再丢弃旧的数组。因此，由于较大的对象的扩容会涉及大量的内存复制操作，如果能够预先评估大小，可提升性能。

唯一需要注意的是：StringBuffer是线程安全的，但是StringBuilder是线程不安全的。可参看Java标准类库的源代码，StringBuffer类中方法定义前面都会由synchronize关键字。为此，StringBuffer的性能要远低于StringBuilder。

3.应用场景

[A]在字符串内容不经常发生变化的业务场景优先使用String类。例如：常量声明、少量的字符串拼接操作等。如果有大量的字符串内容拼接，避免使用String与String之间的“+”操作，因为这样会产生大量无用的中间对象，耗费空间且执行效率低下（新建对象、回收对象花费大量时间）。

[B]在频繁进行字符串的运算（如拼接、替换、删除等），并且运行在多线程环境下，建议使用StringBuffer，例如XML解析、HTTP参数解析与封装。

[C]在频繁进行字符串的运算（如拼接、替换、删除等），并且运行在单线程环境下，建议使用StringBuilder，例如SQL语句拼装、JSON封装等。