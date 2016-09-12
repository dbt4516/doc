## 1 Alluxio是什么
Alluxio大数据存储系统源自于UC Berkeley AMPLab，目前由Alluxio公司在开源社区主导开发。它是世界上第一个以内存为中心的分布式存储系统。
<br>
> Alluxio, formerly Tachyon, enables any application to interact with any data from any storage system at memory speed.

<br>
## 2 Alluxio的特性
* 世代关系（Lineage）：当任务失败时，启动重计算（re-computation）来重做。
* 分布式，内存为中心，多级（内存+SSD+硬盘+HDFS）存储（调度策略有LFU、LRU等），重要的文件可以pin在内存中。省去手动冷热分离。
* 提供统一的文件API，无论这份数据是在内存还是在硬盘还是在HDFS还是在亚马逊S3。
* 提供了类似redis的KV键值对存储。
<br>
<br>

## 3 现实意义
* Alluxio比较重大的意义我认为在于完全省去了程序员__手动冷热分离__的繁琐，而且对不同层级的存储提供了__统一的接入__，甚至可以mount到linux的本地文件系统。用户不用考虑这份数据是在内存还是在硬盘还是在分布式HDFS上，在程序实现上只需要一套。<br>
* 在数据处理中，经常会有job之间相互依赖，为了不同job之间能够共享到这些数据时常需要把数据写出到磁盘，落地缓慢。有了Alluxio，这个部分就转变为内存读写了。对于分布式系统而言，Alluxio __相当于在HDFS之上加了一层高速缓存__，对运行于其上的计算框架能带来很大的性能提升。<br>
* 如果使用Alluxio的KV存储，对于开发人员，就可以少部署一个redis，而且它的KV还是分布式的。虽然目前redis也有官方的分布式方案，但似乎并没有广泛应用。<br>
* 目前世代关系还处于alpha测试阶段，功能目前比较单一（可见参考文档部分）。另，把数据处理层的内容下放到数据存储层这一做法是否合适我认为也应该再讨论。<br>
* 总体而言，利用不同介质和组件进行 __分级缓存的需求在应用中十分常见__，出现一套具备统一接口并自动实现内容调度的工具也是大势所趋。
<br>
<br>

## 4 应用实例
### 4.1 去哪儿（Qunar）日志流处理系统
#### 4.1.1 项目背景
目前，去哪儿网的流处理流水线每天需要处理的业务日志量大约60亿条，总计约4.5TB的数据量。处理逻辑大致为 数据源 -> kafka -> Spark streaming -> HDFS。遇到的主要问题是HDFS在远程集群，网络时延大，且本身HDFS也是基于磁盘IO，速度无法与内存相比；Spark为了提高计算速度将大量内容载入Executor的JVM，导致GC压力大。
#### 4.1.2 解决方案
保持原有流程不变，将核心存储由HDFS替换为Alluxio，并与计算节点部署在一起，HDFS仅作为备份。优化结果：Spark在读取文件时达到内存速度，且不再需要以MEMORY_ONLY的模式运行，缓解了本身JVM的GC压力。将生产环境中整个流处理流水线的性能总体提高了近10倍，峰值时甚至达到300倍左右。<br><br>
### 4.2 百度基础查询优化
#### 4.2.1 项目背景
百度的数据存储节点分散于全球各地，网络吞吐能力成为查询时的最大瓶颈。希望能将热数据直接存放在计算节点的内存中。
> Since the data was distributed over multiple data centers, it was highly likely that a query would need to transfer data from a remote data center to the compute data center — this is what caused the biggest delay when a user ran a query. 

#### 4.2.2 解决方案
Alluxio worker部署到计算节点上，用户发起的请求到达计算节点时，先检查本地的Alluxio是否有所需数据，如果未命中则向HDFS数据仓库请求，并将其缓存进本地的Alluxio。这样的结构比原始的Hive查询快30倍，比切换为Spark SQL查询后快5倍。并且该套方案运行一年余都非常稳定。百度已建立全球最大Alluxio集群。
### 4.3 实例总结
两个case的背景都有跨机房数据传输的问题，解决方案中都可以看到将存储节点和计算节点布置到一起的做法，其中Qunar是直接把HDFS换成了Alluxio，百度则是将Alluxio作为计算节点上的数据缓存。Qunar将原本在分析层管理的缓存剥离到存储层进行管理的思路具有启发性。<br>
<br>
## 5 其他
Alluxio的webUI已很完善（直接浏览目录、查看文件内容、各种配置、worker运行情况、看日志、哪些内容pin在内存里等），一点都不像一个才做了三年的开源项目。不少文章也指出虽然这个开源项目非常新，但却获得了大量厂商的关注，其项目成长速度远远超过Hadoop、Spark等Apache顶级项目在初始阶段的速度。可见业界对自动分级存储、统一接口的需求已经非常强烈。<br><br>
## 6 参考文档
官网<br>
http://www.alluxio.org/<br>
官方中文文档<br>
http://www.alluxio.org/docs/master/cn/<br>
知乎专栏介绍<br>
https://zhuanlan.zhihu.com/p/20624086<br>
中文社区<br>
http://www.alluxioone.cn/<br>
世系关系源码分析<br>
http://www.cnblogs.com/xiamodeqiuqian/p/5330949.html<br>
百度实例<br>
http://www.alluxio.com/assets/uploads/2016/02/Baidu-Case-Study.pdf<br>
去哪儿网实例<br>
http://www.alluxioone.cn/2016/05/31/Qunar-case/<br>
分级存储调度逻辑说明<br>
http://www.alluxio.org/docs/master/cn/Tiered-Storage-on-Alluxio.html#section-5<br>
<br>
hongzhan<br>
2016-09-11
