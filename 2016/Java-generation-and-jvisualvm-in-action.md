##1 分代
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
![](https://github.com/dbt4516/doc/blob/master/2016/pic/Java-generation-and-jvisualvm-in-action-2.png.jpg)<br>
把三大块打散之后，整个内存分配机制更加灵活，也减轻了compact过程的压力。<br>
##2 基本回收策略
一般而言，minor gc使用Serial GC，也即用一个处理器将S0中有用的对象顺序地复制到S1中（mark-copy），这样的做法使得S1中的可用和不可用内存都是连续的，省去了compact过程。大家也会留意到因为划出了S0、S1两个部分来做来回复制，实际上浪费了一半的内存空间（在一个时间点上S0和S1中肯定有一个是完全空置的），但也只有这样才能做速度很快的mark-copy（还顺便有compact的效果），属于用空间换时间的做法。<br>
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

##3 内存三态在Java Virtual VM上的实践
打开JDK目录bin文件夹下jvisualvm.exe，安装Visual GC插件，就能看到以下画面：<br>
![](https://github.com/dbt4516/doc/blob/master/2016/pic/Java-generation-and-jvisualvm-in-action-3.png.jpg)<br>
在画面的左边可以看到永生代（Java8将这个部分改名叫元空间MetaSpace），老年代和分成了三部分的年轻代的内存占用，程序运行时能看到Eden一满对象就会移到S区，S0和S1不断互换，且总有一个是空闲的（同时也可以发现minor gc的过程对用户而言几乎是没有感觉的）；tenured区缓慢增长，快满的时候就会卡一下，然后掉下来一点，这就是major gc了。而MetaSpace的占用基本是没变化。<br>

##4 关于永生代
个人觉得Java8中元空间这个名字比“永生代”更为直观地表达了这个空间的用途：存放JVM需要的类及方法定义。一般来说用户是无法操作这个区域的，但有一个特例：String.intern()，这个方法可将字符串放到JVM的常量字符串池中，在需要大量相同的字符串时可以使用。示例代码如下：
```java
String s1 = "Test";
String s2 = "Test";
String s3 = new String("Test");
String s4 = s3.intern();
System.out.println(s1 == s2); //用常量形式声明的字符串直接从永生代的字符常量池中引用，地址相等。true
System.out.println(s2 == s3); //new出来的字符串在heap区，地址和永生代不相等。false
System.out.println(s3 == s4); //s4是intern的，在永生代，地址和s3不相等。false
System.out.println(s1 == s4); //同样内容又都是intern的，地址相等。true
```
在Java7之前，JVM字符串池位于永生代，因为永生代空间有限（默认仅4M），所以大量使用肯定是不行的。Java7已经将常量字符串池移动到了heap区。<br>

##参考文档
Java内存分代、基本GC策略及如何安装Visual GC插件（Oracle文档，基于JDK7）<br>
http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html<br>
详解Concurrent Mark Sweep、G1回收（图文并茂，而且是中文的，专题中的其他几篇也很值得一看）<br>
http://www.jianshu.com/p/778dd3848196<br>
Java内存分代简介<br>
http://stackoverflow.com/questions/2070791/young-tenured-and-perm-generation<br>
详解String#intern（来自美团技术点评团队，详细介绍了JDK6和7在该功能的区别）<br>
http://tech.meituan.com/in_depth_understanding_string_intern.html
