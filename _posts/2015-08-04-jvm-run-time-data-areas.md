---
layout: post
title: 深入理解JVM虚拟机（JAVA内存区域与内存溢出异常）
category: 技术
tags: [Java,JVM]
description: 
---

> JAVA虚拟机运行时数据区
><a class="group" rel="group1" href="/assets/img/blogimg/java_memory.png"><img src="/assets/img/blogimg/java_memory.png" alt=""></a>
> 阅读 <<深入理解JAV虚拟机--JVM高级特性与最佳实践>>和[Java虚拟机规范](http://docs.oracle.com/javase/specs/jvms/se8/html/index.html)，结合一些自己的理解


Java虚拟机管理的内存包含以下几个运行时数据区域。

# 1、线程私有区 #

顾名思义，所谓线程私有区，就是每个线程运行时独享的内存区域，每个线程独享的内存区域又可分为三个部分

## 1.1、程序计数器（PC）##

几乎所有的程序里都需要一个程序计数器，来选取下一条需要执行的指令。这里可把它看做是线程所执行字节码的行号指示器，通过改变这个PC的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等功能都要这个计数器来完成。

为了保证在JAVA多线程中，每个线程轮流切换后能恢复到正确的执行位置，因此每个线程都需要一个PC计数器，互不影响。

如果线程正在执行的是一个JAVA方法，PC记录的就是下一条需要执行的字节码指令地址（行号），而如果正在执行是一个native方法，则值为空。

注： 这是JVM虚拟机中没有规定任何 `OutOfMemoryError` 的区域


## 1.2、Java虚拟机栈 ##

和PC一样，它也是线程私有的，生命周期与线程相同。 虚拟机栈描述的是java中方法调用执行的内存模型，每个方法执行的同时都会创建一个栈帧（Stack Frame），栈帧中存储着`局部变量表`、`操作数栈`、	`方法出口`、`引用`、`动态链接`等信息，每一个方法从调用到执行完成的过程，就对应着栈帧在虚拟机栈中入栈道出栈的过程

<a class="group" rel="group1" href="/assets/img/blogimg/StackFrame.png"><img src="/assets/img/blogimg/StackFrame.png" alt=""></a>

局部变量表中存储着8种基本数据类型`boolean`、`byte`、`char`、`short`、
`int`、`float`、`long`、`double`，对象引用（一个指向堆中对象起始地址的指针，或者是指向代表对象的句柄），还有返回地址类型（ReturnAddress,指向了一条字节码指令的地址）。

我们一般的说的栈就是指这一块区域。

## 1.3、本地方法栈 ##

1.本地方法栈与虚拟机栈所发挥的作用是非常相似的，它们之间的区别不过是虚拟机栈为虚拟机执行Java方法服务（也就是字节码），而本地方法栈为虚拟机使用到的Native方法服务。

2.Java虚拟机规范对本地方法栈使用的语言、使用方法与数据结构并没有强制规定，因此可以由虚拟机自由实现。例如：HotSpot虚拟机直接将本地方法栈和虚拟机栈合二为一。

3.同虚拟机栈相同，Java虚拟机规范对这个区域也规定了两种异常情况StackOverflowError 和 OutOfMemoryError异常。


	
# 2、共享内存区域 #

## 2.1、Heap堆 ##

Java堆是被所有的线程共享的一块内存区域，在虚拟机启动时创建。Java堆的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存.

Java堆是垃圾回收器管理的主要区域，因此也被称为"GC堆"。从内存回收的角度看，由于现在收集器基本都采用分代收集算法，所以Java堆可以细分为：新生代、老生代；从内存分配的角度看，线程共享的Java堆可能划分出多个线程私有的分配缓冲区（TLAB）；不论如何划分，都与存放的内容无关，无论哪个区域，存储的仍然是对象实例。

Java虚拟机规范规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。如果在堆上没有内存完成实例分配，并且堆上也无法再扩展时，将会抛出OutOfMemoryError异常。


## 2.2、运行时方法区 ##

方法区也是被所有的线程共享的一块内存区域。它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

Java虚拟机规范对方法区的限制非常宽松，除了和Java堆一样 不需要连续的内存和可以选择固定大小或者可扩展之外，还可以选择不实现垃圾回收。
这区域的内存回收目标主要是针对常量池的回收和类型的卸载，一般而言，这个区域的内存回收比较难以令人满意，尤其是类型的回收，条件相当苛刻，但是这部分区域的内存回收确实是必要的。

Java虚拟机规范规定，当方法区无法满足内存分配的需求时，将抛出OutOfMemoryError异常。

## 2.3、运行时常量池 ##

运行时常量池是方法区的一部分。
CLass文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。
运行时常量池相对于CLass文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入CLass文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用比较多的就是String类的intern()方法。


***一些问题***

1.内存泄露和内存溢出

内存泄露：指程序中一些对象不会被GC所回收，它始终占用内存，即被分配的对象引用链可达但已无用。（可用内存减少）

内存溢出：程序运行过程中无法申请到足够的内存而导致的一种错误。内存溢出通常发生于OLD段或Perm段垃圾回收后，仍然无内存空间容纳新的Java对象的情况。

2.String.intern()方法

String.intern()是一个Native方法，它的作用是：如果字符串常量池中已经包含了一个等于此String对象的字符串，则返回代表池中这个字符串的String对象；否则，将此String对象包含的字符串添加到常量池中，并且返回此字符串的引用。

	 String str1 = new StringBuilder("计算机").append("软件").toString();
	 System.out.println(str1.intern() == str1);
	
	 String str2 = new StringBuilder("ja").append("va").toString();
	 System.out.println(str2.intern() == str2);

这段代码在JDK1.6中运行，会得到两个false，而在JDK1.7中运行，会得到一个true和一个false。原因是：

在JDK1.6中intern()方法会把首次遇到的字符串实例复制到永久代中，返回的也是永久代中这个字符串实例的引用，而由StringBuilder创建的字符串实例在Java堆上，所以必然不是一个引用。

在JDK1.7中intern()方法不会复制实例，只是在常量池中记录首次出现的实例引用，因此intern()返回的引用和由StringBuilder创建的字符串实例是同一个。
str2返回false是因为"Java"这个字符串在执行StringBuilder("ja").append("va").toString()之前已经出现过，字符串常量池中已经有它的引用了，不符合首次出现的原则，而"计算机软件"这个字符串是首次出现的。