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

### Job调度

Job调度就是在application内部的一组Job集合，在application分配到的资源量，通过一定的算法，对每个按一定次序分配Application中资源的过程。
- FIFO模式：先进先出模式
- FAIR模式：spark在多个job之间以轮询的方式给任务进行资源分配，所有的任务拥有大致相当的优先级来共享集群的资源。这就意味着当一个长任务正在执行时，短任务仍可以分配到资源，提交并执行，并且获得不错的响应时间。

### tasks延迟调度

- 数据本地性：尽量的避免数据在网络上的传输，传输任务为主，将任务传输到数据所在的节点

- 延时调度机制：拥有数据的节点当前正被其他的task占用，如果预测当前节点结束当前任务的时间要比移动数据的时间还要少，那么调度会等待，直到当前节点可用。否则移动数据到资源充足节点，分配任务执行。

## Spark DataFrame大数据处理框架介绍
