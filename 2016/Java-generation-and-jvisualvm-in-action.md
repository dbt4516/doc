##分代
Java中的内存分为三大块：年轻代（young generation）、老年代（old/tenured generation）和永生代（permanent generation）。其中年轻代分为三个小块，分别为Eden、S0和S1；而永生代是用来存储JVM运行时必须的内容的，不参与GC过程，一般由只由JVM操作，不受编程人员控制，但也有特例，将在本文最末作介绍。<br>
简图如下：
![](https://github.com/dbt4516/doc/blob/master/2016/pic/Java-generation-and-jvisualvm-in-action-1.png.jpg)
接下来用一个简单流程进行说明。
* 一个对象新生成时他会被放在Eden区。
* 过了不久，Eden区被装满，触发第一次minor gc。
* Eden区中无用的对象被清除，有用的对象被转移到S0，JVM会记录这个对象被转移了1次。这时Eden区又有空间了。
* 时过境迁，Eden又满了，再次触发minor gc。
* Eden区中无用对象被清除，有用的对象被转移到S1，JVM也会记录这些对象被转移了1次。同时，S0中仍然有用的对象也将被搬到S1，JVM会记录这些对象被转移了2次。
* 多次minor gc之后有些对象的挪动次数达到阈值（比如说16次），这些对象将被promote成老年代。
* 当老年代的区域满到一定程度时，会出发major gc（也叫full gc），对老年代区域的对象进行检测回收。平时的minor gc是不扫描老年代的。

所以为什么要分年轻代和老年代呢？这里有个basic assumption是刚生成没多久的对象很可能可以被清除，而生成了很久还一直有用的对象，在接下来的一段时间里往往还是不能被清除（类似操作系统里的局部性原理）。将这两类对象分开，就能用minor gc扫描比较少的对象，仍能获得较理想的内存回收效果，迫不得已时才做full gc。<br>
###补充说明
之前的简图中内存的三大块都是连续存储的，这就导致gc回收空间之后，还需要做一次compact（不然可能空间太过细碎，仍不足以放入新的对象，导致连续gc）。在Java7之后，三大块的分布可以是打散的（region式内存划分），如图：
![](https://github.com/dbt4516/doc/blob/master/2016/pic/Java-generation-and-jvisualvm-in-action-2.png.jpg)
把三大块打散之后，整个内存分配机制更加灵活，也减轻了compact过程的压力。<br>
##基本回收策略
一般而言，minor gc使用Serial GC，也即用一个处理器将S0中有用的对象顺序地复制到S1中（mark-copy），这样的做法使得S1中的可用和不可用内存都是连续的，省去了compact过程。<br>
Major gc的策略则比较多种，Java 7之后使用的是优化版的CMS（Concurrent Mark Sweep），Oracle将其命名为G1 Garbage Collecctor。<br>
CMS的基本思路是
* 暂停程序运行，并行地从根节点开始，扫描第一层子节点。（这步叫做initial mark）
* 恢复程序运行，以第一次扫描的结果为根标记tenured区中的可达对象。（concurrent mark）
* 因为第二次扫描时程序在跑，所以扫描结果可能会有遗漏，此时再次暂停程序，确保所有可达对象都被标记。（remark）
* 恢复程序运行，开始根据标记清理内存。（concurrent sweep）
* 做一次compact，去除内存碎片。

可以看到CMS的优点在于只有两个步骤需要程序暂停（名字中的concurrent指的是应用线程和GC线程可以同时运行），比传统的Parallel Scavenge策略显著高效。而目前更为先进的G1优化策略（和之前提到的region式内存划分一起使用）体现在：
* 在minor gc时会顺便不暂停程序地做initial mark。
* Concurrent mark时如果发现某个内存块中没有存活对象，就直接把这块内存回收了，而不是等sweep时再回收。
* sweep阶段会对存活对象较少的内存块做合并。

##内存三态在Java Virtual VM上的实践
