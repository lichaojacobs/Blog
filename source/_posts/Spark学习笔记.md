---
title: Spark学习笔记
date: 2017-04-11 00:12:35
tags:
    - 学习
    - 大数据
    - Spark
---

# Spark 基础

## Spark 的构成

- ClusterManager: 在standalone模式中即为，Master主节点，控制整个集群，监控worker。在yarn模式中为资源管理器
- worker :从节点，负责控制计算节点，启动Executro和Driver。在yarn模式中NodeManager,负责计算节点的控制。
- Driver：运行Application的main()函数并且创建SparkContext。
- Executor: 执行器，是为某Application运行在worker node上的一个进程，启动线程池运行任务上，每个Application拥有一组独立的executors
- SparkContext: 整个应用程序的上下文，控制整个应用的生命周期
- RDD：Spark的基本计算单元，一组RDD形成执行的有向无环图RDD Graph(DAG)
- DAG Scheduler: 根据Job构建基于stage的DAG，并且提交stage给TaskScheduler
- TaskScheduler: 可以将提交给它的stage 拆分为更多的task并分发给Executor执行
- SparkEnv: 线程级别的上下文，存储运行时的重要组件的引用
- DStream: 是一个RDD的序列，由若干RDD组成。在一个batchInterval中，会产生一个RDD，产生的数据统一塞入到这个RDD中，采用内存+磁盘的模式，尽可能放到内存中，当数据量太大时会spill到磁盘中。



## spark 工作机制

- 一个Job被拆分成若干个Stage，每个Stage执行一些计算，产生一些中间结果。它们的目的是最终生成这个Job的计算结果。而每个Stage是一个task set，包含若干个task。Task是Spark中最小的工作单元，在一个executor上完成一个特定的事情。
- 除非用户指定持久化操作，否则转换过程中产生的中间数据在计算完毕后会被丢弃，即数据是非持久化的。
- 窄依赖:父RDD中的一个分区最多只会被子RDD中的一个分区使用，父RDD中，一个分区内的数据是不能被分割的，必须整个交付给子RDD中的一个分区。
- 宽依赖（Shuffle依赖）：父RDD中的分区可能会被多个子RDD分区使用。因为父RDD中一个分区内的数据会被分割，发送给子RDD的所有分区。因此Shuffle依赖也意味着父RDD与子RDD之间存在着Shuffle过程。

### Spark作业

- Application: 用户自定义的Spark程序，用户提交之后，Spark为App分配资源程序转换并执行。
- Driver Program: 运行Application的main函数并且创建SparkContext
- RDD DAG： 当RDD遇到Action算子，将之前的所有算子形成一个有向无环图（DAG）。再在Spark中转化为Job,提交到集群进行执行，一个App中可以包含多个Job
- Job： RDD Graph触发的作业，由spark Action算子触发，在SparkContext中通过runJob方法向spark提交Job
- stage： 每个Job会根据RDD的宽依赖关系被切分很多stage ,每个stage包含一组相同的task，这一组task也叫taskset
- Task: 一个分区对应一个Task,Task 执行RDD中对应stage中所包含的算子，Taksk 被封装好后放入Executor的线程池中执行。

### spark 运行流程

- spark程序转换
- 输入数据块
- 根据调度策略执行各个stage的task（每个stage之间要进行shuffle）
- 结果返回


## spark调度原理

### 作业调度

系统的设计很重要的一环便是资源调度。设计者将资源进行不同粒度的抽象建模，然后将资源统一放入调度器，通过一定的算法进行调度。

- spark的多种运行模式：Local模式，standalone模式、YARN模式，Mesos模式。


### application调度

用户提交到spark中的作业集合，通过一定的算法对每个按一定次序分配集群中资源的过程。
- FIFO模式，用户先提交的作业1优先分配需要的资源，之后提交的作业2再分配资源，依次类推。
- Mesos: 粗粒度模式和细粒度模式
- YARN模式：独占模式，可以控制应用分配资源
  - yarn-cluster: 适用于生产环境。client将用户程序提交到到spark集群中就与spark集群断开联系了，此时client将不会发挥其他任何作用，仅仅负责提交。在此模式下。AM和driver是同一个东西，但官网上给的是driver运行在AM里，可以理解为AM包括了driver的功能就像Driver运行在AM里一样，此时的AM既能够向AM申请资源并进行分配，又能完成driver划分RDD提交task等工作
  
  - yarn-client: y适用于交互、调试，希望立即看到app的输出。Driver运行在客户端上，先有driver再用AM，此时driver负责RDD生成、task生成和分发，向AM申请资源等 ,AM负责向RM申请资源，其他的都由driver来完成

### Job调度

Job调度就是在application内部的一组Job集合，在application分配到的资源量，通过一定的算法，对每个按一定次序分配Application中资源的过程。
- FIFO模式：先进先出模式
- FAIR模式：spark在多个job之间以轮询的方式给任务进行资源分配，所有的任务拥有大致相当的优先级来共享集群的资源。这就意味着当一个长任务正在执行时，短任务仍可以分配到资源，提交并执行，并且获得不错的响应时间。

### tasks延迟调度

- 数据本地性：尽量的避免数据在网络上的传输，传输任务为主，将任务传输到数据所在的节点

- 延时调度机制：拥有数据的节点当前正被其他的task占用，如果预测当前节点结束当前任务的时间要比移动数据的时间还要少，那么调度会等待，直到当前节点可用。否则移动数据到资源充足节点，分配任务执行。

## job 小文件处理问题

- 因为每次job partition数量过多，导致hive表中过多小文件产生，所以需要重新指定分区，有以下俩种方法：repartition(numPartitions:Int):RDD[T]和coalesce(numPartitions:Int，shuffle:Boolean=false):RDD[T]
他们两个都是RDD的分区进行重新划分，repartition只是coalesce接口中shuffle为true的简易实现，（假设RDD有N个分区，需要重新划分成M个分区）
	- N<M。一般情况下N个分区有数据分布不均匀的状况，利用HashPartitioner函数将数据重新分区为M个，这时需要将shuffle设置为true。
	- 如果N>M并且N和M相差不多，(假如N是1000，M是100)那么就可以将N个分区中的若干个分区合并成一个新的分区，最终合并为M个分区，这时可以将shuff设置为false，在shuffl为false的情况下，如果M>N时，coalesce为无效的，不进行shuffle过程，父RDD和子RDD之间是窄依赖关系。
	- 如果N>M并且两者相差悬殊，这时如果将shuffle设置为false，父子ＲＤＤ是窄依赖关系，他们同处在一个Ｓｔａｇｅ中，就可能造成Spark程序的并行度不够，从而影响性能，如果在M为1的时候，为了使coalesce之前的操作有更好的并行度，可以讲shuffle设置为true。

总之：如果shuff为false时，如果传入的参数大于现有的分区数目，RDD的分区数不变，也就是说不经过shuffle，是无法将RDD的分区数变多的。

## [spark 优化与参数调优](http://tech.meituan.com/spark-tuning-basic.html)

- 最重要的是数据序列化和内存调优。对于大多数程序选择Kyro序列化器并持久化序列后的数据能解决常见的性能问题。

- Executor
  - 每个节点可以起一个或多个Executor。每个Executor上的一个核只能同时执行一个task,如果一个Executor被分到了多个task只能排队依次执行
  - Executor内存主要分三块：1、让task执行我们自己编写的代码，默认占总内存的20%。2、让rask通过shuffle过程拉取了上一个stage的task输出后，进行聚合等操作时，默认占用总内存20%；3、让RDD持久化使用，默认60%
  - task的执行速度是跟每个Executor进程的CPU core数量有直接关系的。一个CPU core同一时间只能执行一个线程。而每个Executor进程上分配到的多个task，都是以每个task一条线程的方式，多线程并发运行的。如果CPU core数量比较充足，而且分配到的task数量比较合理，那么通常来说，可以比较快速和高效地执行完这些task线程。

- 广播大变量
  - 当需要用到外部变量时，默认每个task都会存一份，这样会增加GC次数，使用广播变量能确保一个Executor中只有一份

- 使用Kryo优化序列化性能(如果希望RDD序列化存储在内存中，面临GC问题的时候，优先使用序列化缓存技术)
  - spark没有默认使用Kryo作为序列化类库，是因为Kryo要求注册所有需要序列化的自定义类型，这对开发比较麻烦

- spark submit示例:

	```
	  sudo -u mobvoidata spark-submit \
   		--class com.mobvoi.data.analytics.Main \
   		--num-executors 25 \
   		--executor-cores 1 \
   		--driver-memory 4G \
   		--conf spark.kryoserializer.buffer.max=256m \
   		mobvoi-analytics-job-assembly-1.0.1.jar "all" "true/false"

	```