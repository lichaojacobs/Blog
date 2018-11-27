---
title: JVM知识总结
date: 2018-03-26 20:29:18
tags:
	- JVM
	- Java
---

## HotSpot JVM Architecture

![HotSpot JVM Architecture](http://imgs.wanhb.cn/hotspotjvm-1.png)

## JVM运行时数据区域

- 程序计数器 (program counter registers)
- java虚拟机栈：线程私有，生命周期与线程相同。每个方法在运行期间都会创建一个栈帧用于存储局部变量、操作数栈、动态链接、方法出口等信息。
	- 经常有人把java内存区分为堆内存和栈内存，这种分法比较粗糙。其中所指的栈就是虚拟机栈。或者说是虚拟机栈中局部变量表部分。
	- 局部变量表存放了编译期间可知的各种基本数据类型(boolean byte char,…)、对象引用和returnAddress类型
- 本地方法栈 为虚拟机提供Native方法服务
- java堆： 是JVM所管理的内存中最大的一块。是被所有线程共享的一块内存区域。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配（随着JIT编译器的发展与逃逸分析技术的成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化，导致在堆上分配对象不是那么绝对）。
- 方法区： 与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开来。
	- 对于HotSpot熟悉的人员来说，很多人更愿意把方法区称为“永久代”本质上并不等价，仅仅是因为HotSpot虚拟机设计团队选择把GC分代收集扩展到方法区，或者说使用永久代来实现方法区而已。但是这种实现方式更容易遇到内存溢出的问题，现在HotSpot虚拟机也放弃永久代逐步改为采用Native Memory来实现方法区的规划了（如：把原本放在永久代的字符串常量池移除）
	- JVM规范对方法区限制非常宽松，但也不是永久存在，还是会存在类的卸载，常量池的回收问题

- 运行时常量池：方法区的一部分，Class文件中出了有类的版本、字段、方法、接口等描述信息外，还有常量池，用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。
	- 运行期间也可以将新的常量放入池中，如String类的intern()方法（目前被废除）

- 直接内存(Direct Memory)：不是JVM规范定义的内存区域，但是也被频繁使用，而且也可能导致内存溢出异常。在JDK1.4新加入了NIO 类，引入了一种基于通道与缓冲区Buffer的IO方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。显然，这部分内存不会受到Java堆大小的限制，但是还是受到本机总内存的限制


## 对象创建
### 分配实例内存的时候可能会出现并发问题的解决方案：

- 对分配内存空间的动作进行同步处理———实际上JVM采用CAS配上失败重试的方式保证更新操作的原子性
- 把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存称为本地线程分配缓冲（TLAB）。哪个线程要分配内存就在哪个TLAB上分配，只有TLAB用完并分配新的的时候才需要同步锁定。虚拟机是否使用TLAB，可以通过-XX：+／-UseTLAB参数来设定

### GC 算法：

- 可达性分析算法（不算GC算法，仅仅是思想）:
- GC Roots的对象包括下面几种：
	- 1、虚拟机栈（栈帧中的本地变量表）中引用的对象
	- 2、方法区中类静态属性引用的对象
	- 3、方法区中常量引用的对象
	- 4、本地方法栈中JNI（即一般说的Native方法）引用的对象。


- 引用计数法：
- 标记清除（Mark-Sweep)：
- 复制算法 (Copying)
- 标记-压缩算法 (Mark-Compact)
- 增量算法 (Incremental Collecting)

### JVM 垃圾回收器分类

- Young generation:
	- Serial
	- ParNew
	- Parallel Scavenge
- Tenured generation:
	- CMS
	- Parallel Old
	- Serial Old
- All
	- G1

- 新生代串行收集器（GC日志中:Default New Generation）： 第一，它仅仅使用单线程进行垃圾回收；第二，它独占式的垃圾回收。复制算法
- 老年代串行收集器： 标记-压缩算法。
- -XX:+UseParNewGC 参数设置，表示新生代使用并行收集器，老年代使用串行收集器
- -XX:+UseParallelGC 参数设置，表示新生代和老年代均使用并行回收收集器
- 新生代并行回收 (Parallel Scavenge) 收集器，对应的GC日志中新生代的名称为PSYoungGen
- 老年代并行回收收集器
- CMS 收集器: 它使用的是标记-清除算法，同时它又是一个使用多线程并发回收的垃圾收集器。
	- 主要步骤有：初始标记、并发标记、重新标记、并发清除和并发重置。其中初始标记和重新标记是独占系统资源的，而并发标记、并发清除和并发重置是可以和用户线程一起执行的。因此，从整体上来说，CMS 收集不是独占式的，它可以在应用程序运行过程中进行垃圾回收。
	- 标记-清除算法将会造成大量内存碎片，离散的可用空间无法分配较大的对象。在这种情况下，即使堆内存仍然有较大的剩余空间，也可能会被迫进行一次垃圾回收，以换取一块可用的连续内存，这种现象对系统性能是相当不利的，为了解决这个问题，CMS 收集器还提供了几个用于内存压缩整理的算法。
	- -XX:+UseCMSCompactAtFullCollection 参数可以使 CMS 在垃圾收集完成后，进行一次内存碎片整理。内存碎片的整理并不是并发进行的。
	- -XX:CMSFullGCsBeforeCompaction 参数可以用于设定进行多少次 CMS 回收后，进行一次内存压缩。使用CMS收集器后，默认新老年代分别使用ParNew和CMS收集器进行垃圾回收

	- G1 收集器 (Garbage First) : 与 CMS 收集器相比，G1 收集器是基于标记-压缩算法的
		- G1 收集器还可以进行非常精确的停顿控制。它可以让开发人员指定当停顿时长为 M 时，垃圾回收时间不超过 N。
		- 使用参数-XX:+UnlockExperimentalVMOptions –XX:+UseG1GC 来启用 G1 回收器，设置 G1 回收器的目标停顿时间：-XX:MaxGCPauseMills=20,-XX:GCPauseIntervalMills=200。

### JVM内存分配情况

- [参考链接](http://trustmeiamadeveloper.com/2016/03/18/where-is-my-memory-java/)
- 容器内存 = 文件缓存 + Java 应用内存
Java 应用内存 = 堆内存 + MetaSpace + 堆外内存
堆外内存 = 线程栈 + 缓冲区（例如 NIO）等等
- 所以按照目前的经验来看，堆内存至少应该小于 2/3 的容器内存（？）

### happens-before规则

- 定义
	- 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
	- 两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么这种重排序并不非法（也就是说，JMM允许这种重排序）
- 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
- 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
- start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
- join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
- 程序中断规则：对线程interrupted()方法的调用先行于被中断线程的代码检测到中断时间的发生。
- 对象finalize规则：一个对象的初始化完成（构造函数执行结束）先行于发生它的finalize()方法的开始。



### 对象转入老年代的情况：
- 1、经过了正常的MinoGC过程数次之后（可以通过-XX:MaxTenuringThreshold配置）晋升到老年代
- 2、大于一定值的对象直接进入老年代（ 通过-XX:PretenureSizeThreshold配置）
- 3、空间分配担保：MinorGC之前，jvm会检查老年代最大可用的连续空间是否大于新生代所有对象总空间，如果成立，那么MinorGC可以确保是安全的。如果不成立，则虚拟机会查看HandlePromotionFailure设置值是否允许担保失败。如果允许，那么会继续检查老年代最大可用连续空间是否大于历次晋升到老年代对象的平均大小，如果大于则尝试一次MinorGC，尽管会有风险，如果小于。或者HandlePromotionFailure设置不允许冒险，那这时候也需要改为进行一次Full GC.
- 因为新生代是使用复制收集算法，但是为了内存利用率，只使用其中一个Survivor空间作为轮换备份，因此当出现大量对象在MinorGC后仍然存活。就需要老年代进行分配担保。把Survivor无法容纳的对象直接进入老年代。前提是老年代还有容纳这些对象的剩余空间，一共有多少对象会存活下来在实际完成内存回收之前是无法明确知道的，所以只好取之前每一次回收晋升到老年代对象容量的平均大小值作为经验值，与老年代的剩余空间比较，决定是否进行FullGc来让老年代腾出空间。如果允许担保失败(HandlePromotionFailure)并且老年代最大可用连续空间大于历次晋升到老年代对象的平均大小，将尝试进行一次MinorGC，如果小于或者不允许担保失败则会进行一次 FullGC，所以一般会将HandlePromotionFailure打开，以防止FullGC频率过高

### 一些JVM工具
- jps : 虚拟机进程状况工具
	- [详细参考文章](http://www.importnew.com/?p=18132&preview=true)
	- -q 只输出LVMID,省略主类名称
	- -m 输出虚拟机进程启动时传递给主类main()函数的参数
	- -l 输出主类的全名，如果进程执行的是Jar包，输出jar路径
	- -v 输出虚拟机进程启动时JVM参数

- jstat: 虚拟机统计信息监视工具，可以显示本地或者远程虚拟机进程中的类装载、内存、垃圾收集、JIT编译等运行数据
	- [详细参考文章](http://www.importnew.com/18202.html)
	- -class pid 监视类装载、卸载数量、总空间以及类装载所耗费时间
	- -gc pid 监视Java堆状况，包括Eden区，两个Survivor区、老年代、永久代容量、已用空间、GC时间合计等信息
	- -gccapacity pid 可以显示，VM内存中三代（young,old,perm）对象的使用和占用大小
	- -gcnew pid 年轻代对象的信息
	- -gcold 年老代对象信息

- jinfo: java配置信息工具
	- 实时地查看和调整虚拟机各项参数，未被显示指定的参数的系统默认值
	- jinfo -flag CMSInitiatingOccupancyFraction 14444

- jmap java内存映象工具 用于生成堆转储快照（一般称为heapdump或dump文件）如果不使用jmap命令，还可以使用暴力的手段如：-XX:+HeapDumpOnOutOfMemoryError参数，可以让虚拟机在 OOM异常之后自动生成dump文件。通过-XX:+HeapDumpOnCtrlBreak参数则可以使用[Ctrl]+[Break]键让虚拟机生成dump文件。
	- [详细参考文章](http://www.importnew.com/18196.html)
	- jmap 的作用不仅仅是为了获取dump文件，还可以查询finalize执行队列、Java堆和永久代的详细信息，如空间使用率、当前用的是哪种收集器等。
	- -dump 生成java堆转储快照 -dump:[live, ]format=b, file=,其中live子参数说明是否只dump出存活的对象
	- -heap 显示java堆详细信息，如使用那种回收器、参数配置、分代状况等／
	- -histo 显示堆中对象的统计信息，包括类，实例数量，合计容量
	- -F 当虚拟机进程对-dump选项没有响应时，可使用这个选项强制生成dump快照。
	- 例子：jmap -dump:format=b,file=exlipse.bin 3500 (3500 是通过jps命令查询到的LVMID)

- jhat: 虚拟机堆转储快照分析工具
	- sun JDK提供jhat 与jmap 搭配使用，来分析jmap 生成的堆转储快照。jhat内置了一个微型的Http/Html服务器，生成dump文件的分析结果后，可以在浏览器中查看之后可以用visualVM来分析dump文件

- jstack: java堆栈跟踪工具，用于生成虚拟机当前时刻的线程快照。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环
	- [详细参考文章](http://www.importnew.com/18176.html)


### 垃圾收集器参数总结：
- UseSerialGC ：虚拟机运行在Client模式下的默认值，打开此开关后，使用Serial+Serial Old 的收集器组合进行内存回收
- UseParNewGC: 打开此开关后使用ParNew+SerialOld的收集器组合进行内存回收
-  UseConcMarkSweepGC：打开此开关后，使用ParNew+CMS+Serial Old的收集器组合进行内存回收，Serial Old收集器将作为CMS收集器出现Concurrent Mode Failure失败后的后备收集器使用
- UseparallelGC 虚拟机运行在Server模式下的默认值，打开此开关后，使用Parallel Scavenge+Serial Old (PS MarkSweep) 的收集器组合进行内存回收
UseParallelOldGC: 使用Parallel Scavenge+ Parallel Old的收集器组合进行内存回收

### GC 相关参数总结

- 与串行回收器相关的参数

```
-XX:+UseSerialGC:在新生代和老年代使用串行回收器。
-XX:+SurvivorRatio:设置 eden 区大小和 survivor 区大小的比例。
-XX:+PretenureSizeThreshold:设置大对象直接进入老年代的阈值。当对象的大小超过这个值时，将直接在老年代分配。
-XX:MaxTenuringThreshold:设置对象进入老年代的年龄的最大值。每一次 Minor GC 后，对象年龄就加 1。任何大于这个年龄的对象，一定会进入老年代。

```
- 与并行 GC 相关的参数

```
-XX:+UseParNewGC: 在新生代使用并行收集器。
-XX:+UseParallelOldGC: 老年代使用并行回收收集器。
-XX:ParallelGCThreads：设置用于垃圾回收的线程数。通常情况下可以和 CPU 数量相等。但在 CPU 数量比较多的情况下，设置相对较小的数值也是合理的。
-XX:MaxGCPauseMills：设置最大垃圾收集停顿时间。它的值是一个大于 0 的整数。收集器在工作时，会调整 Java 堆大小或者其他一些参数，尽可能地把停顿时间控制在 MaxGCPauseMills 以内。
-XX:GCTimeRatio:设置吞吐量大小，它的值是一个 0-100 之间的整数。假设 GCTimeRatio 的值为 n，那么系统将花费不超过 1/(1+n) 的时间用于垃圾收集。
-XX:+UseAdaptiveSizePolicy:打开自适应 GC 策略。在这种模式下，新生代的大小，eden 和 survivor 的比例、晋升老年代的对象年龄等参数会被自动调整，以达到在堆大小、吞吐量和停顿时间之间的平衡点。

```
- 与 CMS 回收器相关的参数

```
-XX:+UseConcMarkSweepGC: 新生代使用并行收集器，老年代使用 CMS+串行收集器。
-XX:+ParallelCMSThreads: 设定 CMS 的线程数量。
-XX:+CMSInitiatingOccupancyFraction:设置 CMS 收集器在老年代空间被使用多少后触发，默认为 68%。
-XX:+UseFullGCsBeforeCompaction:设定进行多少次 CMS 垃圾回收后，进行一次内存压缩。
-XX:+CMSClassUnloadingEnabled:允许对类元数据进行回收。
-XX:+CMSParallelRemarkEndable:启用并行重标记。
-XX:CMSInitatingPermOccupancyFraction:当永久区占用率达到这一百分比后，启动 CMS 回收 (前提是-XX:+CMSClassUnloadingEnabled 激活了)。
-XX:UseCMSInitatingOccupancyOnly:表示只在到达阈值的时候，才进行 CMS 回收。
-XX:+CMSIncrementalMode:使用增量模式，比较适合单 CPU。
```

- 与 G1 回收器相关的参数

```
-XX:+UseG1GC：使用 G1 回收器。
-XX:+UnlockExperimentalVMOptions:允许使用实验性参数。
-XX:+MaxGCPauseMills:设置最大垃圾收集停顿时间。
-XX:+GCPauseIntervalMills:设置停顿间隔时间。
```
- 其他参数

```
-XX:+DisableExplicitGC: 禁用显示 GC。
```


### JVM调优实战

除了Java堆和永久代之外还有以下区域还会占用较多的内存，这些内存总和受到操作系统进程最大内存的限制
- DirectMemory：可通过 -XX:MaxDirectMemorySize调整大小，内存不足时抛出OutOfMemoryError 或者 OutOfMemoryError: Direct buffer memory.如果把系统大部分的内存都分配给了java堆，并且很长时间没有发生GC操作。那么DirectMemory可能会溢出。因为DirectMemory只是等待老年代满了之后FullGC，然后顺便帮它清理掉内存的废弃对象。否则就只能一直等到抛出内存溢出异常。
- 线程堆栈： 可通过-Xss调整大小，内存不足时抛出StackOverflowError
- Socket 缓存区：每个Socket连接都Receive和Send两个缓存区，分别占大约37KB和25KB内存，连续多的话这块内存也很可观。
- JNI 代码：如果代码中使用JNI调用本地库，那本地库使用的内存也不在堆中。
- 虚拟机和GC：虚拟机、GC的代码执行也要消耗一定的内存


- 解读GC 日志

```
[GC (Allocation Failure) 2017-10-15T16:02:33.567+0800: 790.871: [ParNew
Desired survivor size 17432576 bytes, new threshold 6 (max 6)
- age 1: 7619992 bytes, 7619992 total
- age 2: 1081760 bytes, 8701752 total

```

可以明显看出上述 GC 日志包含两次 Minor GC。 注意到第二次 Minor GC 的情况， 日志打出 "Desired survivor size 53673984 bytes"， 但是却存活了 "- age 1: 107256200 bytes, 107256200 total" 这么多。 可以看出明显的新生代的 Survivor 空间不足。正因为 Survivor 空间不足， 那么从 Eden 存活下来的和原来在 Survivor 空间中不够老的对象占满 Survivor 后， 就会提升到老年代， 可以看到这一轮 Minor GC 后老年代由原来的 0K 占用变成了 105782K 占用， 这属于一个典型的 JVM 内存问题， 称为 "premature promotion"(过早提升)。"premature promotion” 在短期看来不会有问题， 但是经常性的"premature promotion”， 最总会导致大量短期对象被提升到老年代， 最终导致老年代空间不足， 引发另一个 JVM 内存问题 “promotion failure”（提升失败： 即老年代空间不足以容乃 Minor GC 中提升上来的对象）。 “promotion failure” 发生就会让 JVM 进行一次 CMS 垃圾收集进而腾出空间接受新生代提升上来的对象， CMS 垃圾收集时间比 Minor GC 长， 导致吞吐量下降、 时延上升， 将对用户体验造成影响。


### Java什么情况下会报OutOfMemoryError

- 如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StatckOverflowError异常；若果虚拟机栈可以动态扩展(当前大部分的Java虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈)，当扩展时无法申请到足够的内存时会抛出OutOfMemoryError异常。
- 根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样，在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前主流的虚拟机都是按照可扩展来实现的(通过-Xmx和-Xms控制)。如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。
- 根据Java虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。
- 运行时常量池是方法区的一部分，自然会受到方法区内存的限制，当常量池无法在申请到内存时会抛出OutOfMemoryError异常。
- 直接内存(Direct Memory)并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用，而且也可能导致OutOfMemoryError异常的出现。在JDK1.4中新加入了NIO(New Input/Output)类，引入了一种基于通道(Channel)与缓冲区(Buffer)的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作，这样能在一些场景中显著提高性能，因为避免了在Java堆和Native中来回复制数据。显然，本机直接内存的分配不会受到Java堆大小的限制，但是，既然是内存，则肯定还是会受到本机总内存(包括RAM及SWAP区或分页文件)的大小及处理器寻址空间的限制。

### 虚拟机类加载机制
- 类加载的生命周期：加载—验证—准备—解析—初始化—使用—卸载
- 类初始化阶段：
是类加载过程的最后一步，到了初始化阶段，才真正开始执行类中定义的Java程序代码（或者说字节码）
在准备阶段，变量已经赋值锅系统要求的初始值，，而在初始化阶段，则根据程序员通过程序制定的主观计划去初始化类变量和其他资源，从另一个方面表达：
初始化阶段是执行类构造器()方法的过程。
方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块(static{}块)中的语句合并产生的，编译器收集的顺序是由语句在原文件中出现的顺序决定的。

- 静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。如：

	```
public class test{
static{
i=0; //给变量赋值可以正常编译通过
System.out.print(i); //这句编译器会提示“非法向前引用”
}
static int i=1;
}
	```
	() 方法与类的构造函数不同，它不需要显示的调用父类构造器，虚拟机会保证在子类的clinit方法执行之前，父类的cinit方法已经执行完毕。因此在虚拟机中第一个被执行的cinit方法的类肯定是java.lang.object。由于父类的clinit方法先执行，也就意味着父类中定义的静态语句块要优先于子类变量赋值操作。比如如下B的值将会是2而不是1

	```
	static class Parent{
	public static int A =1;
		static {
		A =2
		}
	}

	static class Sub extends Parent{
	public static int B=A
	}

	public static void main(String[] args){
	system.out.println(Sub.B);
	}
	```
	
	- 1）clinit 方法不是必须的，如果一个类中没有静态语句块，也没有对变量的赋值操作，也就没有必要为这个类生成clint方法。
	- 2）接口中不能使用静态语句块，但仍然有变量初始化赋值操作，因此接口和类一样都会生成clinit方法。但接口与类不同的是，执行接口的clinit方法不需要先执行父接口的clinit方法。只有当父接口中定义的变量使用时，父接口才会初始化。另外，接口实现类在初始化时也一样不会执行接口的clinit方法
	- 3）虚拟机保证一个类的clinit方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的clinit方法，其他线程都需要阻塞等待，直到活动线程执行clinit方法完毕。同一个类加载器下，一个类型只会初始化一次


- 类加载器
	- 1）用于实现类的加载动作.比较两个类是否相等，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则即使来源于同一个Class文件，被同一个虚拟机加载，只要类加载器不同，那么这两个类就必定不相等。相等指的是 equals() isAssignableFrom()方法，isInstance()方法返回的结果。也包括使用instanceof关键字做对象所属关系判定等情况。
	- 2）双亲委派模型
从Java虚拟机的角度，只存在两种不同的类加载器：一种是启动类加载器(Bootstrap ClassLoader)，这个类加载器由C++实现，另一种就是所有其他的类加载器，由Java实现并且都继承自抽象类java.lang.ClassLoader
	- 3）从开发人员的角度，类加载器还可以分更细：
		- 启动类加载器(Bootstrap ClassLoader)：负责加载Java_HOME\lib目录中的，或者被-Xbooclasspath参数所指定的路径中的，并且是虚拟机识别的类库加载到虚拟机内存中。无法被java程序直接饮用。
		- 扩展类加载器（Extension ClassLoader） 负责加载JAVA_HOME\lib\ext目录中，或者被java.ext.dirs系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器
		- 应用程序加载器（Application ClassLoader） 这个类加载器是ClassLoader中的getSystemClassLoader()方法的返回值，所以也被称为系统类加载器。负责加载用户类路径（ClassPath）上所指定的类库，如果应用程序没有自定义锅自己的类加载器，一般情况下这个就是程序中默认的类加载器

### [JVM逃逸分析](http://www.importnew.com/23150.html)
- 逃逸分析的基本行为就是分析对象动态作用域：当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他地方中，称为方法逃逸
- 如果能证明一个对象不会逃逸到方法或线程外，则可能为这个变量进行一些高效的优化

#### 优化点
-  栈上分配
	- 我们都知道Java中的对象都是在堆上分配的，而垃圾回收机制会回收堆中不再使用的对象，但是筛选可回收对象，回收对象还有整理内存都需要消耗时间。如果能够通过逃逸分析确定某些对象不会逃出方法之外，那就可以让这个对象在栈上分配内存，这样该对象所占用的内存空间就可以随栈帧出栈而销毁，就减轻了垃圾回收的压力。
	- 在一般应用中，如果不会逃逸的局部对象所占的比例很大，如果能使用栈上分配，那大量的对象就会随着方法的结束而自动销毁了。
-  同步消除：参考[高并发Java 九锁的优化和注意事项](https://my.oschina.net/hosee/blog/615865)
-  标量替换
	- Java虚拟机中的原始数据类型（int，long等数值类型以及reference类型等）都不能再进一步分解，它们就可以称为标量。相对的，如果一个数据可以继续分解，那它称为聚合量，Java中最典型的聚合量是对象。如果逃逸分析证明一个对象不会被外部访问，并且这个对象是可分解的，那程序真正执行的时候将可能不创建这个对象，而改为直接创建它的若干个被这个方法使用到的成员变量来代替。拆散后的变量便可以被单独分析与优化，可以各自分别在栈帧或寄存器上分配空间，原本的对象就无需整体分配空间了


### JIT 与Java10
- Introduction:
	- 对于大部分应用开发者来说，Java编译器指的是JDK自带的javac指令。这一指令可将Java源程序编译成.class文件，其中包含的代码格式我们称之为Java bytecode（Java字节码）。这种代码格式无法直接运行，但可以被不同平台JVM中的interpreter解释执行。*由于interpreter效率低下，JVM中的JIT compiler（即时编译器）会在运行时有选择性地将运行次数较多的方法编译成二进制代码，直接运行在底层硬件上。*Oracle的HotSpot VM便附带两个用C++实现的JIT compiler：C1及C2。
	- 与interpreter，GC等JVM的其他子系统相比，JIT compiler并不依赖于诸如直接内存访问的底层语言特性。它可以看成一个输入Java bytecode输出二进制码的黑盒，其实现方式取决于开发者对开发效率，可维护性等的要求。Graal是一个以Java为主要编程语言，面向Java bytecode的编译器。与用C++实现的C1及C2相比，它的模块化更加明显，也更加容易维护。Graal既可以作为动态编译器，在运行时编译热点方法；亦可以作为静态编译器，实现AOT编译。在Java 10中，Graal作为试验性JIT compiler一同发布（JEP 317）。这篇文章将介绍Graal在动态编译上的应用。有关静态编译，可查阅JEP 295或Substrate VM。
- Tiered Compilation
	- HotSpot中的tiered compilation
		- JIT compiler — C1及C2（或称为Client及Server）
			- 前者没有应用激进的优化技术，因为这些优化往往伴随着耗时较长的代码分析。因此，C1的编译速度较快，而C2所编译的方法运行速度较快。在Java 7前，用户需根据自己的应用场景选择合适的JIT compiler。举例来说，针对偏好高启动性能的GUI用户端程序则使用C1，针对偏好高峰值性能的服务器端程序则使用C2。
			- Java 7引入了tiered compilation的概念，综合了C1的高启动性能及C2的高峰值性能。这两个JIT compiler以及interpreter将HotSpot的执行方式划分为五个级别：
				- level 0：interpreter解释执行
				- level 1：C1编译，无profiling
				- level 2：C1编译，仅方法及循环back-edge执行次数的profiling
				- level 3：C1编译，除level 2中的profiling外还包括branch（针对分支跳转字节码）及receiver type（针对成员方法调用或类检测，如checkcast，instnaceof，aastore字节码）的profiling
				- level 4：C2编译。其中，1级和4级为接受状态 — 除非已编译的方法被invalidated（通常在deoptimization中触发），否则HotSpot不会再发出该方法的编译请求。

### JVM优秀博客
- [Java字节码结构剖析一：常量池](http://www.importnew.com/30461.html)
- [Java字节码结构剖析二：字段表
](http://www.importnew.com/30505.html)
- [Java字节码结构剖析三：方法表
](http://www.importnew.com/30521.html)
- [从字节码和 JVM 的角度解析 Java 核心类 String 的不可变特性](http://www.importnew.com/26595.html)
- [高并发java博客系列](https://my.oschina.net/hosee?tab=popular)


