---
title: InnoDB读书笔记
date: 2017-02-15 22:44:21
tags:
    - InnoDB
    - mysql
    - 成长
    - 读书
---

### 基础知识

 1. 数据库的四种隔离级别

    - Read Uncommitted(读未提交): 如果一个事务已经开始写数据，则另外一个事务则不允许同时进行写操作，但允许其他事务读此行数据。于是事务B可能读取到了事务A未提交的数据。
    - Read -Committed(读提交): 读取数据的事务允许其他事务继续访问该行事务，但是未提交的写事务存在时禁止一切其他事务，该隔离级别避免了脏读，但是却可能出现不可重复读。事务A事先读取了数据，事务B紧接了更新了数据，并提交了事务，而事务A再次读取该数据时，数据已经发生了改变。
    - Repeated read （可重复读） 读取数据的事务将会禁止写事务（但允许读事务），写事务则禁止任何其他事务。mysql的InnoDB的Repeated read 级别就可以解决幻读的问题(源自Next-Key Locking算法)，而oracle只能将隔离级别设置在Serializable才能解决幻读的问题。
    - Serializable（串行化）： 序列化是最高的事务隔离级别，同时代价也花费最高，性能很低，一般很少使用，在该级别下，事务顺序执行，不仅可以避免脏读、不可重复读，还避免了幻像读。 提供严格的事务隔离。它要求事务序列化执行，事务只能一个接着一个地执行，但不能并发执行。

 2. InnoDB关键特性

    - 插入缓冲（Insert Buffer） 不可能每张表上只有一个聚集索引，在进行插入操作时候，数据页存放还是按照主键a进行顺序存放的，但是对于非聚集索引叶子节点的插入不再是顺序的了，这时会需要离散地访问非聚集索引页，需要注意，辅助索引的插入顺序依然是顺序的，或者说比较顺序的，比如用户购买表中的时间字段，通常情况下，用户购买时间是一个辅助索引，用来根据时间条件进行查询，但是在插入时却是根据时间递增而插入的，因此插入较为顺序。Insert Buffer 对于非聚集索引的插入和更新操作，不是每一次直接插入到索引页面，而是判断插入的是否在非索引页是否在缓冲池，若在，则直接插入。若不在先放入一个Insert Buffer中，好似欺骗。 因此  Insert Buffer需要满足的俩个条件： 索引是辅助索引，并且索引不是唯一的。
    - 俩次写（Double Write）
    - 自适应哈希索引（Adaptive Hash Index）
    - 异步IO
    - 刷新邻接页

### 索引与算法

 1. InnoDB存储引擎支持的几种常见索引
    - B+ 树索引：其下索引又分为：聚集索引，辅助索引（内部均为B+树）
    - 全文索引
    - 哈希索引 （哈希索引是自适应的，根据表的使用情况自动生成） 注意：B+ 树索引并不能找到一个给定键值的具体行。B+树索引能找到的只是被查找数据行所在的页，然后数据库通过把页读入到内存，再在内训中进行查找，最后得到想要查找的数据。

#### B+树索引

 1. 定义和性质
    - 由二叉树和平衡二叉树演化而来。是为磁盘或其他直接存取辅助设备设计的一种平衡查找树，在B+树中，所有记录的节点都是按照键值的大小顺序存放在同一层的叶子节点上。由各叶子节点指针进行连接。
    - B+树的插入必须保证插入之后叶子节点的记录依然排序。
    - B+树在数据库中的有一个特点就是高扇出性，因此在数据库中B+树的高度一般都在2～4层，也就是说查找一个键值记录最多只需要2到4次IO。
 2. B+树定义
    - 分为聚集索引和辅助索引。俩者的不同之处在于叶子节点存放的是否是一整行的信息。MyISAM引擎使用B+Tree作为索引结构，叶节点的data域存放的是数据记录的地址，而InnoDB的辅助索引data域存储相应记录主键的值而不是地址。而聚集索引能够在叶子节点上直接找到数据。此外，由于定义了数据的逻辑顺序，聚集索引能够特别快地针对范围值查询，查询优化器能够快速发现某一段范围的数据页需要扫描。
    - 聚集索引。由于实际的数据页只能按照一棵B+树进行排序，因此每张表只能拥有一个聚集索引。在多数情况下，查询优化器倾向于使用聚集索引。因为:
    1）聚集索引能够直接在B+树索引上找到数据。
    2）定义了数据的逻辑顺序，聚集索引能够特别快地访问针对范围值的查询。能够快速的发现某一段范围的数据页需要扫描
    **误区**：很多书和博客都介绍，聚集索引按照顺序物理地存储数据，其实这样会导致维护成本非常高，所以聚集索引的存储并不是物理上连续的。而是逻辑连续。**原因**：
    1）页通过双向链表链接，页按照主键的顺序排序；
    2）页中的记录也是通过双向链表进行维护的，物理存储上可以同样不按照主键存储。

 3. B+树索引的分裂
 4. B+树索引的管理
    - **Cardinality值：**
     1）并不是在所有的查询条件中出现的列都需要添加索引，一般的经验是，在访问表中很少一部分时使用B+树索引才有意义，对于性别、地区、类型字段，它们的可取值范围很小，称为低选择性，于是建索引是完全没有必要的。
     2）对于高选择性的确定，可以通过show index结果列中的列Cardinality来观察，表示的是索引中不重复记录数量的预估值。注意：仅仅是个预估值，而不是一个准确值。
 5. B+索引树的使用
    - 分析
    1）用Show index table 查看索引的情况
    2）用explain+sql语句 来分析这条sql的情况，包括使用的possible_keys（可能选择的索引）与key(实际选择的索引)

    - 联合索引
      是指对表上的多个列进行索引，创建方法与单个索引的创建方法一样，不同之处在于有多个索引列,如对于联合索引(a,b) ,以下是关于联合索引的一些情况
    
     - select * from table where a=x and b=x;
    	 	 		 显然是可以用(a,b)索引的，因为索引的顺序就是按照（a,b）来排序的
	  
	  - select * from table where a=xxx; 这个也是可以使用(a,b)索引的，只不过只能使用一部分
	  - select * from table where b=xxx; 这个是不能使用索引的，因为压根就没按照b做排序。
  - 联合索引的好处是，在第一个索引的基础上，已经给第二个键值做了排序的处理，从而可以减少一次排序操作：比如我们要查某个用户购买商品的情况，并且按照时间排序。
    
	- 覆盖索引
		- Innodb存储引擎支持覆盖索引，即从辅助索引中就可以得到查询的记录，而不需要查询聚集索引中的记录。使用覆盖索引的一个好处是辅助索引不包含整行记录的所有信息，因此，大小要远小于聚集索引，可以大大减少IO操作。
		- 对于Innodb的辅助索引而言，由于其包含了主键信息，因此其叶子节点存放的数据为(primary key1,primary key2,…)例如，下列语句都可以通过使用一次辅助索引来完成查询
		
			```
			select primary key1,key2,… from table
			where
			key1=xxx;
			对于覆盖索引的另一个好处是对某些统计问题而言，
			不会选择通过聚集索引来进行统计。因为辅助索引的量远小于聚集索引，
			可以大大减少IO操作。
			
			```

#### 优化器选择不使用索引的情况

```
select * from order 
where orderId>10000 and orderId<102000; 
（从orderId的基数看来非常大，这是前提）

```

- 这句sql按道理来说可以使用辅助索引的，可选的 KEYS有primary orderId, 等索引，然而最后选择了primary聚集索引，也就是全表扫描。
- 原因在于用户要选取的数据是整行信息，而orderId索引不能覆盖到我们要查询的信息，因此在对orderId索引查询到指定的数据之后，还需要一次书签访问来查询整行信息，虽然orderId索引中的数据是顺序存放的，但是再一次进行书签来查找数据则是无序的，因此变为了磁盘上的离散读操作。
- 如果要求访问的数据量很小。则优化器还会选择辅助索引，但是当访问的数据占整个表中数据的蛮大一部分的时候（通常是百分之二十），优化器则会选择通过聚集索引来查找数据，因此之前已经提到过，顺序读要远远快于离散读。
- 因此，对于不能进行索引覆盖的情况，优化器选择辅助索引的条件是占有的数据量很小。当然这是由当前传统的机械硬盘的特性决定的。
  
- InnoDB存储引擎中的哈希算法
自适应哈希索引：数据库自身创建并使用的，经过哈希函数映射到一个哈希表中，因此对于字典类型的查找非常快速。如 select * from table where index_cole=“xxx” 但是对于范围查找久无能为力了。

### InnoDB锁机制

#### lock 与 latch区别

- latch一般称为轻量级锁，因为其要求锁定的时间必须非常短。若持续的时间长，则应用的性能会非常差。在InnoDB存储引擎中，latch又可以氛围mutex（互斥量）与 rwlock 读写锁。目的是用来保证并发线程的操作临界资源的正确性，并且通常没有死锁检测的机制，仅仅通过应用程序加锁的顺序来保证无死锁的情况发生。
- lock的对象是事务，用来锁定的是数据库中的对象。如表，页，行。并且一般lock的对象仅在事务commit或rollback后进行释放。且有死锁机制
- latch查看 show engine innodb mutex。lock查看 show engine innodb status

#### InnoDB存储引擎中的锁
- **InnoDB实现了俩种标准的行级锁**：
  - 共享锁：允许事务读一行数据 
  - 排他锁：允许事务删除或者更新一行数据

如果一个事务T1已经获得了行r的共享锁，那么另外的事务T2可以立即获得行的共享锁，称这种情况为锁兼容。但若有其他事务T3想要获得这一行的排他锁，则必须等待事务T1，2释放行上的共享锁称这种情况为锁不兼容。
可以总结出：**X锁与任何锁都不兼容。而S锁仅仅和S锁兼容。需注意的是，S和X锁都是行级锁，兼容是指对同一记录锁的兼容性情况**。

- **意向锁的定义：**InnoDB支持多粒度锁定，这种锁定允许事务在行级上的锁和表级上的锁同时存在。为了支持在不同粒度上的枷锁操作。InnoDB存储引擎支持一种额外的枷锁方式，称为**意向锁**。它是将锁定的对象分为多个层次，意向锁意味着事务希望在更细粒度上进行加锁。

- 如需要在页上的记录r进行行上X锁，那么分别需要对数据库A、表、页上意向锁IX，最后对记录r上X锁。若其中任何一个部分导致等待，那么该操作需要等待粗粒度的锁完成。举例子说明：在对记录r加X锁之前。已经有事务对表1进行了S表锁，那么表1上一存在S锁，之后事务需要对记录R表1上加上IX，由于不兼容，所以该事务需要等待表锁操作的完成。
- InnoDB存储引擎支持意向锁设计比较简练。其意向锁即为表级别的锁，设计的目的主要是为了在事务中揭示下一行将被请求的锁类型。其支持俩种意向锁：
  - 意向共享锁**（IS Lock）**，事务想要获得一张表中某几行的共享锁。 
  - 意向排他锁（**IX Lock**），事务想要获得一张表中某几行的排他锁。

由于InnoDB支持的是**行级别的锁**，因此意向锁其实不会阻塞**除全表扫**以外的任何请求，故而意向锁与行级别锁的兼容性
![兼容性](http://jacobs.wanhb.cn/images/lock.png)

意向锁是InnoDB自动加的，不需用户干预。对于UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据集加**排他锁（X)**；对于普通SELECT语句，**InnoDB不会加任何锁**；事务可以通过以下语句显示给记录集加共享锁或排他锁。

#### InnoDB行锁实现方式
InnoDB行锁是通过给索引上的索引项加锁来实现的，这一点MySQL与Oracle不同，后者是通过在数据块中对相应数据行加锁来实现的。InnoDB这种行锁实现特点意味着：
```
只有通过索引条件检索数据，InnoDB才使用行级锁，否则，将使用表级锁
```
#### 一致性非锁定读
InnoDB中通过多版本控制的方式来读取当前执行时间数据库中的行的数据。如果读取的行正在执行DELETE或者UPDATE操作，这时读取操作不会因此等待下去，相反，InnoDB存储引擎会去读取行的一个快照数据**。是InnoDB默认的读取方式**

- 称之为非锁定读是因为不需要等待访问的行上X锁的释放。快照数据是指该行的之前版本的数据；也称作多版本并发控制（MVCC）
- 在不同的事务隔离级别Read Committed和Repeatable read下，**InnoDB使用非锁定的一致性读**。但是对于快照的定义却不相同。**在RC下，对于快照数据，总是读取被锁定行最新的一份快照数据，而在RR下，总是读取事务开始时的行数据版本**
- 快照隔离只是适用于**只读事务**，但是对于**读-写事务**，由于默认是**一致性非锁定读**它却无法解决棘手的**写倾斜问题**（具体定义看《数据密集型应用系统设计》），要想解决写倾斜问题（幻读的一种），还得在读取的时候显式加锁，即**一致性锁定读方式**

#### 一致性锁定读
在默认的配置下，即事务的隔离级别为Repeatable Read模式下，**InnoDB存储引擎的select操作使用一致性非锁定读**，锁实现采用**next-key-lock算法**解决幻读但是在某些情况下，用户需要显式地加锁来保证数据的一致性。InnoDB存储引擎对于SELECT语句支持俩种一致性的锁定读操作：
```
select....FOR UPDATE //对行记录加一个X锁，其他事务不能对已锁定的行加上任何锁。
select....LOCK IN SHARE MODE//对读取的行记录加一个S锁，其他事务可以向被锁定的行加S锁，但是如果加上X锁则会被阻塞。
```
必须在一个事务中，如果事务提交，锁也就释放了。

#### 自增长与锁

#### 外键和锁

 1. 外键
    - 对于外键列，如果没有显示地给这个列加索引，**则InnoDB存储引擎自动对其加一个索引，因为这样可以避免默认使用表级锁的情况（在博文InnoDB行锁实现方式一小节提到）**
    - 对于外键的插入或更新，首先需要查询父表中的记录，对于父表的SELECT操作，不是采用一致性非锁定读的方式，因为这样会发生数据不一致问题。使用的是Select ... lock in share mode方式，即主动对父表加一个S锁。

#### 锁的算法

 1. InnoDB 3种行锁的算法
    - RecordLock: 单个行记录上的锁。总会去锁住索引记录，如果InnoDB在建立的时候没有设置任何一个索引，那么这时，InnoDB存储引擎会使用隐士的主键来锁定
     - Gap Lock: 间隙锁，锁定一个范围，但不包含记录本身。目的是为了解决Phantom Problem，即阻止多个事务将记录插入到同一范围内。
    - Next-Key Lock: Gap Lock+ record Lock,锁定一个范围，并且锁定记录本身。结合了GapLock和RecordLock。在Next-Key Lock算法下，InnoDB对于行的查询都是采用这种锁定算法。例如一个索引有10，11，13，20这四个值，那么该索引可能被Next-Key Locking的区间为： (-8,10)、[10,11)...。

 2. Next-Key Lock
   
     - 采用的锁定技术为Next-Key Locking,锁定的不是单个值，而是一个范围，是谓词索引的一种改进。若事务T1已经通过next-key locking锁定了范围：(10,11]、(11,13] 当插入新的记录12时，则锁定的范围会变成:(10,11]、(11,12],(12,13]。 然而当查询的索引含有**唯一属性**时，InnoDB会对Next-Key Lock进行优化降级为Record Lock，即仅仅锁住索引本身，而不是范围。
 3. 解决Phantom Problem
    - **Phantom Problem定义：** 是指在同一事务下，连续执行俩次同样的SQL语句可能导致不同的结果，第二次执行的SQL语句可能会返回之前不存在的行。
    - **在默认的事务隔离级别下，即REPEATABLE READ下**，InnoDB存储引擎采用Next-Key Locking机制来避免幻象问题。
    
    ```
    举例说明：表t由1，2，5三个值组成，事务T1执行：select * from t where a>2 for update;
    这时，T1并没有提交操作，此时结果返回5，与此同时，另一个事务T2插入了4这个值，在执行一遍sql
    会返回4，5，即俩次的结果不一致。InnoDB的next-key locking算法避免Phantom Problem,是对（2，+8）这个范围加了X锁，因此对于任何这个范围插入的操作都是不被允许的。
        
    ```

#### 锁问题

 1. 脏读
   
    - 脏数据和脏页的关系：脏页指的是在缓冲池中已经被修改的页，但是还没有刷新到磁盘中，即数据库实例内存中的页和磁盘的页的数据是不一致的。当然在刷新到磁盘之前，日志都已经被写入到了重做日志文件中，而脏数据是指**事务对缓冲池中的行记录的修改，并且还没有被提交。** 一个事务可以读到另一个事务中为提交的数据，这显然违反了数据库的个隔离性。
 2. 不可重复读
    - 重点在于修改。是指在**一个事务内**多次读取同一数据集合，在这个事务还没有结束时，另外一个事务也访问该同一数据集合，并做了一些DML操作，因此在第一个事务中的俩次读数据之间，由于第二个事务的修改，导致俩次读取到的结果集合可能出现不一致的情况。
    - 不可重复读和脏读的区别是，脏读是读到未提交的数据，而不可重复读读到的却是已经提及的数据，但是其违反了数据库事务一致性的要求。
    - 一般来说不可重复读还是可以接受的，不少数据库厂商（Oracle、MicroSoft SQL SERVER）将其数据库事务的默认隔离级别设置为Read COMMITTED
    - InnoDB存储引擎中，通过使用**Next-Key Lock**算法来避免不可重复读的问题。Mysql官方文档中将不可重复读的问题定义为Phantom Problem，即幻象问题
 3. 幻读
   
    - 重点在于新增或删除。当事务不是独立执行时发生的一种现象，第一个事务对一个表中的数据进行了修改，涉及到全部的数据行。同时第二个事务也修改了这个表中的数据，这种修改是向表中插入一行新数据。那么之后第一个事务重新读取数据的时候就会出现幻读的情况
    - **Next-Key Lock**算法解决幻读问题
 4. 丢失更新
     - 丢失更新是另一个锁导致的问题，简单来说就是一个事务的更新操作会被另外一个事务的更新操作所覆盖，从而导致数据的不一致。
        - 1）事务T1将行记录r更新为v1,但是事务T1并未提交
        - 2）与此同时事务T2将行记录更新为V2，事务T2未提交
        - 3）事务T1提交
        - 4）事务T2提交 
     
     但是在任何隔离级别下，都不会导致数据库理论意义上的丢失更新问题。但是生产应用中还有一种逻辑意义的丢失更新，而导致该问题的并不是因为数据库本身的问题。
      - 实际上，在所有多用户计算机系统环境下都有可能产生这个问题：
       
      - 1) 事务T1查询一行数据，放入本地内存，并显示给一个终端用户User1
      - 2) 事务T2也查询该行数据，放入本地内存，并显示给一个终端用户User2
      - 3) user1修改该行记录，更新数据库并提交
      - 4) User2也修改该行记录，更新数据库并提交
    
     要避免丢失更新的发生，就需要让事务在这种情况下的**操作变成串行化，而不是并行操作**。于是在步骤一的过程中，用户读取数据加上一个**排他锁**，这样用户2读取的时候也必须加上排他锁，否则就等待锁的释放。
 4. 阻塞
    - 因为不同锁之间的兼容性关系，在有些时刻一个事务中的锁需要等待另一个事务中的锁释放它所占用的资源，这就是阻塞。阻塞不是一件坏事，是为了确保事务可以并发且正常地运行。
    - 在InnoDB中，参数 **innodb_lock_wait_timeout**用来控制等待的时间**（默认是50秒）**，innodb_rollback_on_timeout用来设定是否在等待超时时对进行中的事务进行回滚操作，默认不回滚。可以通过代码调整： 
    
     ```
    set @@innodb_lock_wait_time=60
    innodb_rollback_on_timeout是静态的，不可在启动时进行修改。
    默认情况下，innoDB存储引擎不会回滚超时引发的错误异常，其实InnoDB存储引擎在大部分情况下，都不会对异常进行回滚。
        
     ```

#### 死锁

 1. 死锁的概念
 2. 死锁的概率
 3. 死锁的示例
 4. 锁升级
