---
title: "JVM内存相关知识"
date: 2019-02-03T10:27:42+08:00
draft: true
---

## 一、StackOverflowError与OutOfMemoryError的定义

> The following exceptional conditions are associated with native method stacks:
If the computation in a thread requires a larger native method stack than is permitted, the Java Virtual Machine throws a StackOverflowError .
If native method stacks can be dynamically expanded and native method stack expansion is attempted but insufficient memory can be made available, or if insufficient memory can be made available to create the initial native method stack for a new thread, the Java Virtual Machine throws an OutOfMemoryError.

> --- 《The Java® Virtual Machine Specification Java SE 8 Edition》

- 如果一个线程在计算过程中不允许获取更大的本地方法栈（虚拟机栈）则抛出StackOverflowError。
- 如果本地方法栈（虚拟机栈）可以动态地扩展，并且本地方法栈（虚拟机栈）尝试过扩展了，但是没有足够的内容分配给它，再或者没有足够的内存为线程初始化本地方法栈（虚拟机栈），那么JVM抛出的就是OutOfMemoryError。
  实际上JVM的虚拟机栈没有支持栈动态扩展，所以可以理解为没有足够的内存为新建线程初始化本地方法栈（虚拟机栈）。

## 二、JVM 内存模型

JVM 规范定义：JVM 内存共分为虚拟机栈、堆、方法区、程序计数器、本地方法栈五个部分。

![](https://ws3.sinaimg.cn/large/006tNc79gy1fzt133q8uuj30vy0jo40i.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1fzt115ls1hj30kq0cvtdj.jpg)

### 1. 虚拟机栈（问题重灾区）

每个线程创建的时候也会创建一个私有的栈。栈里面存着的是一种叫“栈帧”的东西，每个方法会创建一个栈帧，栈帧中存放了局部变量表（基本数据类型和对象引用）、操作数栈、方法出口等信息。栈的大小可以固定也可以动态扩展，可以使用java -Xss (大小值)配置栈大小。

- **StackOverflowError**：当栈中方法调用深度大于JVM所允许的范围，会抛出StackOverflowError的错误，所以可以理解为嵌套调用or递归调用出现了死循环。不过这个深度范围不是一个恒定的值，局部变量表内容越多，栈帧越大，栈深度越小。
- **OutOfMemoryError**：当新建线程申请不到存放栈的空间时，会抛出 OutOfMemoryError，所以可以理解为线程数量爆炸。

### 2. 本地方法栈（烧香拜佛不要出问题）

本地方法栈的功能和特点类似于虚拟机栈，均具有线程隔离的特点以及都能抛出**StackOverflowError**和**OutOfMemoryError**异常。虚拟机规范并未对本地方法栈进行强制规定，因此不同的虚拟机实可以进行自由实现，我们常用的HotSpot虚拟机选择合并了虚拟机栈和本地方法栈。
~~一般来说菜鸟不需要关心本地方法栈的问题。。。~~

### 3. 程序计数器（天堂净土阿瓦隆）

程序计数器，也叫PC 寄存器。JVM支持多个线程同时运行，每个线程都有自己的程序计数器。倘若当前执行的是 JVM 的方法，则该寄存器中保存当前执行指令的地址；倘若执行的是native 方法，则PC寄存器中为空。
程序计数器不会有StackOverflowError和OutOfMemoryError异常，~~所以一般来说菜鸟不需要关心程序计数器的问题（反正程序计数器也不会有异常）。。。~~

### 4. 堆（**问题重灾区**）

堆内存是 JVM 所有线程共享的部分，在虚拟机启动的时候就已经创建。所有的对象和数组都在堆上进行分配。这部分空间可通过 GC 进行回收。当申请不到空间时会抛出 OutOfMemoryError，堆大小由-Xmx和-Xms来调节。

- **OutOfMemoryError**：不同于栈OOM，堆如果出现OOM情况是很复杂的。可能是逻辑导致的循环创建，也有可能是真的内存不足。

### 5. 方法区**（问题轻灾区）**

方法区也是所有线程共享。主要用于存储类的信息、常量池、方法数据、方法代码等。方法区逻辑上属于堆的一部分，但是为了与堆进行区分，通常又叫“非堆”。
方法区也会出现OutOfMemoryError，其大小由-XX:PermSize和-XX:MaxPermSize来调节。并且这个区域很少进行垃圾回收，回收目标主要是针对常量池的回收和对类型的卸载。

- **PermGen（永久代，存在于JDK6/7中）**：绝大部分 Java 程序员应该都见过 "java.lang.OutOfMemoryError: PermGen space "这个异常。这里的 “PermGen space”其实指的就是方法区。
  不过方法区和“PermGen space”又有着本质的区别。前者是 JVM 的规范，而后者则是 JVM 规范的一种实现，并且只有 HotSpot 才有 “PermGen space”，而对于其他类型的虚拟机，如 JRockit（Oracle）、J9（IBM） 并没有“PermGen space”。
  由于方法区主要存储类的相关信息，所以对于动态生成类的情况比较容易出现永久代的内存溢出。最典型的场景就是，在 jsp 页面比较多的情况容易出现永久代内存溢出以及动态创建类的情况。
  需要注意的是方法区的OOM并不是new了很多类，而是有很多类的信息被ClassLoader加载导致OOM。

- **Metaspace（元空间，存在于JDK8+中）：**元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。
  元空间是为了解决永久代的各种问题，例如：　
  1、字符串存在永久代中，容易出现性能问题和内存溢出。
  2、类及方法的信息等比较难确定其大小，因此对于永久代的大小指定比较困难，太小容易出现永久代溢出，太大则容易导致老年代溢出。
  3、永久代会为 GC 带来不必要的复杂度，并且回收效率偏低。
  可以通过以下参数来指定元空间的大小：
  -XX:MetaspaceSize：初始空间大小，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize时，适当提高该值。
  -XX:MaxMetaspaceSize：最大空间，默认是没有限制的。
  除了上面两个指定大小的选项以外，还有两个与 GC 相关的属性：
  -XX:MinMetaspaceFreeRatio：在GC之后，最小的Metaspace剩余空间容量的百分比，减少为分配空间所导致的垃圾收集
  -XX:MaxMetaspaceFreeRatio：在GC之后，最大的Metaspace剩余空间容量的百分比，减少为释放空间所导致的垃圾收集
- **永久代、元空间报错信息比较：**

JDK1.6：PermGen space
![](https://ws4.sinaimg.cn/large/006tNc79gy1fzt11hhyi7j31hc08b40u.jpg)
JDK1.7：Java heap sapce
![](https://ws4.sinaimg.cn/large/006tNc79gy1fzt11on66pj318e07adqc.jpg)
JDK1.8，指定 PermSize 和 MaxPermSize：                    
![](https://ws4.sinaimg.cn/large/006tNc79gy1fzt11v0yfzj318a085amo.jpg)
JDK1.8，指定 MetaSpaceSize 和 MaxMetaSpaceSize的大小：
![](https://ws4.sinaimg.cn/large/006tNc79gy1fzt12045kfj30zi06tdmv.jpg)

### 6. 堆外内存（要什么自行车？）

 JVM可以使用的内存分外2种：堆内存和堆外内存。堆外内存就是把内存对象分配在Java虚拟机的堆以外的内存，这些内存直接受操作系统管理（而不是虚拟机），这样做的结果就是能够在一定程度上减少垃圾回收对应用程序造成的影响。JDK5.0之后，代码中能直接操作本地内存的方式有2种：使用未公开的Unsafe和NIO包下ByteBuffer。

**Direct Memory的回收机制：**Direct Memory是受GC控制的，例如ByteBuffer bb = ByteBuffer.allocateDirect(1024)，这段代码的执行会在堆外占用1k的内存，Java堆内只会占用一个对象的指针引用的大小，堆外的这1k的空间只有当bb对象被回收时，才会被回收，这里会发现一个明显的不对称现象，就是堆外可能占用了很多，而堆内没占用多少，导致还没触发GC，那就很容易出现Direct Memory造成物理内存耗光。为了避免这种悲剧的发生，通过-XX:MaxDirectMemorySize来指定最大的堆外内存大小，当使用达到了阈值的时候将调用System.gc来做一次full gc，以此来回收掉没有被使用的堆外内存。但实际上Direct ByteBuffer分配出去的内存其实也是由GC负责回收的，而不像Unsafe是完全自行管理的，Hotspot在GC时会扫描Direct ByteBuffer对象是否有引用，如没有则同时也会回收其占用的堆外内存。

在日常业务代码中（非基础组件），非常不推荐自己使用堆外内存，第一是内存建立和释放需要自己管理很容易出问题，第二是出了问题很难从java层面去排查。
但是也不排除使用的组件或框架使用到堆外内存，所以在出错的时候也要清楚意识到可能是堆外内存的问题。