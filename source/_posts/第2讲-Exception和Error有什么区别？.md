---
title: 第2讲-Exception和Error有什么区别？
date: 2019-04-25 14:15:45
tags: 
     - JAVA 
     - Java核心技术36讲
toc: true
---

# 第2讲-Exception和Error有什么区别？

杨晓峰 2018-05-05

极客时间版权所有: https://time.geekbang.org/column/article/6845



**请对比Exception和Error，另外，运行时异常与一般异常有什么区别？**

**典型回答**

Exception和Error都是继承了Throwable类，在java中只有Throwable类型的实例才可以被抛出（throw）或者捕获（catch），它是异常处理机制的基本组成类型。

Exception和Error体现了java平台设计者对不同异常情况的分类。Exception是程序正常运行中，可以预料的意外情况，可能并且应该被捕获，进行相应处理。

Error是指在正常情况下，不大可能出现的情况，绝大部分的Error都会导致程序（比如JVM自身）处于非正常的、不可恢复状态。既然是非正常情况，所以不便于也不需要捕获。常见的比如OutOfMemoryError之类，都是Error的子类。

Exception又分为**可检查**（checked）异常和**不检查**（unchecked）异常，可检查异常在源代码里必须显式地进行捕获处理，这是编译期检查的一部分。前面介绍的不检查的Error，是Throwable不是Exception。

不检查异常就是所谓的运行时异常，类似NullPointerException、ArrayIndexOutOfBoundsException之类，通常是可以编码避免的逻辑错误，具体跟进需要来判断是否需要捕获，并不会在编译期强制要求。

<!--more-->

**考点分析**

分析Exception和Error的区别，是从概念角度考察了java处理机制。需要掌握两个方面。

第一，**理解Throwable、Exception、Error的设计和分类**。比如，掌握那些应用最为广泛的子类，以及如何自定义异常等。

很多面试官会进一步追问一些细节，比如，你了解哪些Error、Exception或者RuntimeException？我画了一个简单的类图，并列出典型例子。



![img](https://static001.geekbang.org/resource/image/ac/00/accba531a365e6ae39614ebfa3273900.png)

其中有些子类型，最好重点理解一下，比如NoClassDefFoundError和ClassNotFoundException有什么区别，这也是个经典的入门题目。

第二，理解java语言中操作Throwable的元素和实践。掌握最基本的语法是必须的，如try-catch-finally块，throw、throws关键字等。与此同时，也要懂得如何处理典型场景。

异常处理代码比较繁琐，比如我们需要写很多千篇一律的捕获代码，或者在finally里面做一些资源回收工作。随着java语言的发展，引入了一些更加便利的特性，比如try-with-resources和multiple catch，具体可以参考下面的代码段。在编译时期，会自动生成相应的处理逻辑，比如，自动按照约定俗成close那些扩展了AutoCloseable或者Closeable的对象。

```
try (BufferedReader br = new BufferedReader(…);
     BufferedWriter writer = new BufferedWriter(…)) {// Try-with-resources
// do something
catch ( IOException | XEception e) {// Multiple catch
   // Handle it
} 

```

**知识扩展**

下面的代码反映了异常处理中哪些不当之处？

```
try {
  // 业务代码
  // …
  Thread.sleep(1000L);
} catch (Exception e) {
  // Ignore it
}

```

这段代码虽然很短，但是已经违反了异常处理的两个基本原则。

第一，**尽量不要捕获类似Exception这样的通用异常，而是应该捕获特定异常，** 在这里是Thread.sleep()抛出的InterruptedException。

这是因为在日常的开发和合作中，我们读代码的机会往往超过写代码，软件工程是门协作的艺术，所以我们有义务让自己的代码能够直观地体现出尽量多的信息，而泛泛的 Exception 之类，恰恰隐藏了我们的目的。另外，我们也要保证程序不会捕获到我们不希望捕获的异常。比如，你可能更希望 RuntimeException 被扩散出来，而不是被捕获。

进一步讲，除非深思熟虑了，否则不要捕获 Throwable 或者 Error，这样很难保证我们能够正确程序处理 OutOfMemoryError。



第二，**不要生吞（swallow）异常**。这是异常处理中要特别注意的事情，因为很可能会导致非常难以诊断的诡异情况。

生吞异常，往往是基于假设这段代码可能不会发生，或者感觉忽略异常是无所谓的，但是千万不要在产品代码做这种假设！

如果我们不把异常抛出来，或者也没有输出到日志（Logger）之类，程序可能在后续代码以不可控的方式结束。没人能够轻易判断究竟是哪里抛出了异常，以及是什么原因产生了异常。

第二段代码：

```
try {
   // 业务代码
   // …
} catch (IOException e) {
    e.printStackTrace();
}

```

这段代码作为一段实验代码，它是没有任何问题的，但是在产品代码中，通常都不允许这样处理。你先思考一下这是为什么呢？

我们先来看看<a href="https://docs.oracle.com/javase/9/docs/api/java/lang/Throwable.html#printStackTrace--">printStackTrace()</a>的文档，开头就是“Prints this throwable and its backtrace to the **standard error stream**”。问题就在这里，在稍微复杂一点的生产系统中，标准出错（STERR）不是个合适的输出选项，因为你很难判断出到底输出到哪里去了。

尤其是对于分布式系统，如果发生异常，但是无法找到堆栈轨迹（stacktrace），这纯属是为诊断设置障碍。所以，最好使用产品日志，详细地输出到日志系统里。

我们接下来看下面的代码段，体会一下**Throw early, catch late** 原则。

```
public void readPreferences(String fileName){
	 //...perform operations... 
	InputStream in = new FileInputStream(fileName);
	 //...read the preferences file...
}

```

如果 fileName 是 null，那么程序就会抛出 NullPointerException，但是由于没有第一时间暴露出问题，堆栈信息可能非常令人费解，往往需要相对复杂的定位。这个 NPE 只是作为例子，实际产品代码中，可能是各种情况，比如获取配置失败之类的。在发现问题的时候，第一时间抛出，能够更加清晰地反映问题。

我们可以修改一下，让问题“throw early”，对应的异常信息就非常直观了。

```
public void readPreferences(String filename) {
	Objects. requireNonNull(filename);
	//...perform other operations... 
	InputStream in = new FileInputStream(filename);
	 //...read the preferences file...
}

```

至于“catch late”，其实是我们经常苦恼的问题，捕获异常后，需要怎么处理呢？最差的处理方式，就是我前面提到的“生吞异常”，本质上其实是掩盖问题。如果实在不知道如何处理，可以选择保留原有异常的cause信息，直接再抛出去。在更高层面，因为有了清晰的（业务）逻辑，往往会更清楚合适的处理方式是什么。

有的时候，我们会根据需要自定义异常，这个时候除了保证提供足够的信息，还有两点需要考虑：

1.是否需要定义成 Checked Exception，因为这种类型设计的初衷更是为了从异常情况恢复，作为异常设计者，我们往往有充足信息进行分类。

2.在保证诊断信息足够的同时，也要考虑避免包含敏感信息，因为那样可能导致潜在的安全问题。如果我们看 Java 的标准类库，你可能注意到类似 java.net.ConnectException，出错信息是类似“ Connection refused (Connection refused)”，而不包含具体的机器名、IP、端口等，一个重要考量就是信息安全。类似的情况在日志中也有，比如，用户数据一般是不可以输出到日志里面的。



业界有一种争论（甚至可以算是某种程度的共识），Java 语言的 Checked Exception 也许是个设计错误，反对者列举了几点：

1.Checked Exception 的假设是我们捕获了异常，然后恢复程序。但是，其实我们大多数情况下，根本就不可能恢复。Checked Exception 的使用，已经大大偏离了最初的设计目的。

2.Checked Exception 不兼容 functional 编程，如果你写过 Lambda/Stream 代码，相信深有体会。



很多开源项目，已经采纳了这种实践，比如 Spring、Hibernate 等，甚至反映在新的编程语言设计中，比如 Scala 等。 如果有兴趣，你可以参考：

http://literatejava.com/exceptions/checked-exceptions-javas-biggest-mistake/

当然，很多人也觉得没有必要矫枉过正，因为确实有一些异常，比如和环境相关的 IO、网络等，其实是存在可恢复性的，而且 Java 已经通过业界的海量实践，证明了其构建高质量软件的能力。我就不再进一步解读了，感兴趣的同学可以点击http://v.qq.com/x/page/d0635rf5x0o.html，观看 Bruce Eckel 在 2018 年全球软件开发大会 QCon 的分享 Failing at Failing: How and Why We’ve Been Nonchalantly Moving Away From Exception Handling。

我们从性能角度来审视一下 Java 的异常处理机制，这里有两个可能会相对昂贵的地方：

1.try-catch 代码段会产生额外的性能开销，或者换个角度说，它往往会影响 JVM 对代码进行优化，所以建议仅捕获有必要的代码段，尽量不要一个大的 try 包住整段的代码；与此同时，利用异常控制代码流程，也不是一个好主意，远比我们通常意义上的条件语句（if/else、switch）要低效。

2.Java 每实例化一个 Exception，都会对当时的栈进行快照，这是一个相对比较重的操作。如果发生的非常频繁，这个开销可就不能被忽略了。

所以，对于部分追求极致性能的底层类库，有种方式是尝试创建不进行栈快照的 Exception。这本身也存在争议，因为这样做的假设在于，我创建异常时知道未来是否需要堆栈。问题是，实际上可能吗？小范围或许可能，但是在大规模项目中，这么做可能不是个理智的选择。如果需要堆栈，但又没有收集这些信息，在复杂情况下，尤其是类似微服务这种分布式系统，这会大大增加诊断的难度。

当我们的服务出现反应变慢、吞吐量下降的时候，检查发生最频繁的 Exception 也是一种思路。关于诊断后台变慢的问题，我会在后面的 Java 性能基础模块中系统探讨。

今天，我从一个常见的异常处理概念问题，简单总结了 Java 异常处理的机制。并结合代码，分析了一些普遍认可的最佳实践，以及业界最新的一些异常使用共识。最后，我分析了异常性能开销，希望对你有所帮助。



扩展面试题：

####  常见的异常类有哪些？

- NullPointerException 空指针异常
- ClassNotFoundException 指定类不存在
- NumberFormatException 字符串转换为数字异常
- IndexOutOfBoundsException 数组下标越界异常
- ClassCastException 数据类型转换异常
- FileNotFoundException 文件未找到异常
- NoSuchMethodException 方法不存在异常
- IOException IO 异常
- SocketException Socket 异常

<p>Checked Exception 的假设是我们捕获了异常，然后恢复程序。但是，其实我们大多数情况下，根本就不可能恢复。Checked Exception 的使用，已经大大偏离了最初的设计目的。</p>



**NoClassDefFoundError与ClassNotFoundException的区别？**

NoClassDefFoundError是一个错误（Error），而ClassNotFoundException是一个异常，在java中对于错误和异常的处理是不同的，我们可以从异常中恢复程序但却不应该尝试从错误中恢复程序。

ClassNotFoundException的产生原因：

java支持使用Class.forName方法来动态地加载类，任意一个类的类名如果被作为参数传递给这个方法都将导致该类被加载到JVM内存中，如果这个类在类路径中没有被找到，那么此时就会在运行时抛出ClassNotFoundException异常。

ClassNotFoundException的产生原因主要是：
Java支持使用反射方式在运行时动态加载类，例如使用Class.forName方法来动态地加载类时，可以将类名作为参数传递给上述方法从而将指定类加载到JVM内存中，如果这个类在类路径中没有被找到，那么此时就会在运行时抛出ClassNotFoundException异常。
解决该问题需要确保所需的类连同它依赖的包存在于类路径中，常见问题在于类名书写错误。
另外还有一个导致ClassNotFoundException的原因就是：当一个类已经某个类加载器加载到内存中了，此时另一个类加载器又尝试着动态地从同一个包中加载这个类。通过控制动态类加载过程，可以避免上述情况发生。



NoClassDefFoundError产生的原因在于：
如果JVM或者ClassLoader实例尝试加载（可以通过正常的方法调用，也可能是使用new来创建新的对象）类的时候却找不到类的定义。要查找的类在编译的时候是存在的，运行的时候却找不到了。这个时候就会导致NoClassDefFoundError.
造成该问题的原因可能是打包过程漏掉了部分类，或者jar包出现损坏或者篡改。解决这个问题的办法是查找那些在开发期间存在于类路径下但在运行期间却不在类路径下的类