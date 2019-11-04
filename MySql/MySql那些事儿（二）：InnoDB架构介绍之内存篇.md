# MySql那些事儿（二）：InnoDB架构介绍之内存篇
作者：阿茂

### InnoDB介绍
书接上回，我们基本说完了mysql的逻辑架构与物理架构。今天我们来说说当下比较火的存储引擎InnoDb。MySQL 5.5以前InnoDB引擎是需要手动通过Plugin方式引入，
内置的是MyISAM。MySQL 5.5以后随着数据库默认引擎的更换，InnobaseOy的InnoDB就开始大放异彩(以下我们将用MySQL5.7举例)。我们先说说它有什么样的特性让诸位英雄都说香：
* 免费：这个在当时Oracle,DB2,Sybase MSSQL垄断的时代，有一款免费还相对稳定的的数据库，再加上后期社区非常活跃，能不香吗？
* 完善的崩溃恢复机制：不管是服务器硬件故障还是挖掘机挖断电线导致的关机，重启数据库后都无需执行任何操作（根据设置参数有关）
崩溃恢复会自动完成崩溃之前已提交的所有更改，并撤消正在处理但尚未提交的所有更改。只需重新启动，然后从上次中断的地方继续即可。
* 高效的缓存查询机制：通过一些算法和设计（后面会说到）让常用的数据最大可能的基于内存Buffer处理减少IO，通常这块Buffer将高达数据库服务器内存的80%分配给它。
* 完整性外键机制：将数据拆分到不同数据表中使用外键机制会自动更新或删除关联表中的数据（现在基本没有人这么玩了，一旦设计不全面，
数据的修改将不受程序的控制，数据耦合度太强，适用于一些强制保证数据完整规范的系统，例如工作流这类的）
* 数据完整性校验机制：数据在磁盘或内存中损坏，则校验机制会在使用前提醒您注意虚假数据
* 自动引入主键列：有适当数据库主键列时会自动在Where子句，Order By子句，Group By字句后自动引入主键。
* 基于数据库缓存Buffer写数据：不仅允许对同一表的并发读写访问，而且还缓存更改的数据以减少磁盘IO。
* 自适应哈希索引 ：当频繁的从巨表中查询相同行的数据时候就会使用此方式，让它们像从Hash表中查出来的一样。
* 表和索引压缩：这个不用大多说了，它借助于zlib库，采用L777压缩算法。就是让你在原来两个数据页中的数据现在一页就能显示下，但是过度压缩会带来重组页的影响，
而且不是对所有类型数据列都有良好的压缩比。
* 动态创建或删除索引，保证数据的可用性（这个后面单独说）
* 截断表空间非常快，并且可以释放空间给系统重用。
* 支持存储引擎混用，最常见的就是我们关联表查询的时候产生的中间临时表，默认就是使用MEMORY引擎来实现的。
>更多特点请参考阅读最下方：Benefits of Using InnoDB Tables

###InnoDB内存结构
接下来我们来看一张InnoDB的逻辑架构图：

![](https://dev.mysql.com/doc/refman/5.7/en/images/innodb-architecture.png)

#### Buffer Pool #### 
InnoDB使用malloc()操作在启动时为整个缓冲池分配缓存，缓冲池存储着常访问的表数据和索引，以页面为单位（一个页面可能包含多个行 ）组成一个链接列表，
使用LRU的变体算法将缓冲池作为列表管理。当需要空间将新页面加入到缓冲池时，将淘汰掉最近最少使用的页面，将其加入到列表的中间，
通过此插入点将列表分为两段：最前面的一段为最新访问过的页面，末尾是最近访问的旧页面，如下图：该算法大致如下：

![](https://dev.mysql.com/doc/refman/5.7/en/images/innodb-buffer-pool-list.png)

>默认情况下，3/8的缓冲池专用于旧的数据页列表（Old SubList），当然也可以（通过innodb_old_blocks_pct参数默认是37，5-95范围值）调整其比例大小。
列表的中点（Midpoint）是新数据页（New SubList）列表的尾部与旧数据页（Old Sublist）列表的头相交的边界。当一个页面读入缓冲池时，
它首先会插入旧列表（Old Sublist）的头部如果再次读取到旧列表（Old Sublist）的数据页时（在一定时间内，由参数innodb_old_blocks_time来决定，
默认1000ms）将其移动到新列表的的头部。如果是用户启动的操作需要读取页面，立即将其“年轻化”如果是预读操作读取该页面第一次，不发生任何变化。
随着数据库的逐渐运行，中点数据的不断插入，新老页面都会随着其他页面的更新而老化，而最终未被使用到的页面将到达旧列表的尾部被淘汰掉。

默认情况下，查询读取的页面会立即移入新的列表，它们在缓冲池中停留时间更长。例如，针对mysqldump操作或不带WHERE的SELECT子句执行的表扫描可以将大量数据载入缓冲池，
并老化同等数量的旧数据，即使不再使用新数据也是如此。预读后台线程加载且仅访问一次的页面将移到新列表的开头。这些情况可能会将热数据挤压到旧的列表或者淘汰。
我们可以通过以下几种方法来优化：
* 调整innodb_old_blocks_pct参数：为较小的值可以使仅读取一次的数据不会占用缓冲池的很大一部分。扫描小型表时，在缓冲池中移动页面的开销较小，可以设置为较大值。
* 防止缓冲池被预读搅动：可以避免由于表或索引扫描而引起的类似问题，它们的特征是：通常快速连续地访问数据页面几次，而再也不会被访问。调整innodb_old_blocks_time （第一次访问数据页的时间窗口，单位MS，在该时间窗口内可以访问页面而不将其移到New SubList的头部），增加此值会使越来越多的块从缓冲池中更快地老化。
* 线程预读：根据顺序访问的缓冲池中的页面来预测很快需要哪些页面。调整触发异步读取请求所需的顺序页面访问数（innodb_read_ahead_threshold）参数来执行预读操作。
InnoDB仅当读取当前扩展区的最后一页时，才计算是否对下一个扩展区发出异步预取请求。如果扩展区顺序读取的页面数大于或等于此参数则启动整个后续扩展区的异步读取操作。
设值范围为0-64之间，默认为56,。值越高访问检查越严格。
* 随机预读：根据缓冲池中已有的页面来预测何时可能需要该页面，而不管这些页面的读取顺序如何。需要设置 innodb_random_read_ahead为on。

如果服务器资源利用率不高也可以通过调整Buffer Pool参数来提高资源利用率这里我们就大致说几种（有兴趣的话我们单独开一篇说说mysql内存怎么使用）：
* 增加缓冲池实例大小：在大内存机器上调整 innodb_buffer_pool_instances来提高并发性。

* 增加缓冲池大小：太小会造成数据搅动，太大会造成争用内存而导致交换。通常建议将 innodb_buffer_pool_size其配置为系统内存的50％到75％。
缓冲池大小必须始终等于innodb_buffer_pool_chunk_size* innodb_buffer_pool_instances的倍数否则Buffer Pool会自动调整为此规则。
* 调整刷盘函数：在某些Linux/Unix版本的系统使用fsync()刷盘时性能异常差，存在这样的问题可以使用innodb_flush_method参数设置为进行基准测试O_DSYNC。
* noop调度或时间调度程序与本机AIO一起使用：innodb_use_native_aio（关于系统调度程序请参考本文尾部资料：*调整 Linux I/O 调度器优化系统性能*）
* 配置缓冲池刷新：缓冲池刷新是由页面清理程序线程执行的。清理程序线程的数量由innodb_page_cleaners变量控制默认4。
当脏页百分比达到innodb_max_dirty_pages_pct_lwm变量定义的低水位标记值时，启动缓冲池刷新。默认低水位为0，这会禁用早期的刷新。
尽量提高脏页刷盘的利用率。还有innodb_flush_neighbors参数是指：从缓冲池刷新页面是否也刷新其他脏页面，0为不刷新，默认为1刷新与其连续的脏页，2为刷新相同extend上的脏页。
* 配置innodb_lru_scan_depth参数：针对每个缓冲池实例指定页面清理线程扫描以查找要刷新的脏页面的缓冲池LRU列表的下行距离。

我们可以通过SHOW ENGINE INNODB STATUS来查看Buffer Pool的状态：
```shell

----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 2198863872
Dictionary memory allocated 776332
Buffer pool size   131072
Free buffers       124908
Database pages     5720
Old database pages 2071
Modified db pages  910
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 4, not young 0
0.10 youngs/s, 0.00 non-youngs/s
Pages read 197, created 5523, written 5060
0.00 reads/s, 190.89 creates/s, 244.94 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 5720, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]

```
>提示：基于InnoDB上次打印输出以来经过的时间每秒平均值

#### Change Buffer ####
仅非主键索引的二级索引页不在Buffer Pool中时，会更改这部分缓存，将合并后读入Buffer Pool再刷到磁盘。先来看个图吧：

![](https://dev.mysql.com/doc/refman/5.7/en/images/innodb-change-buffer.png)

非主键索引中的插入顺序相对随机，删除和修改可能会影响不相邻的非主键索引页，最后通过异步线程将受影响的页读入Buffer Pool时批量合并更改并写入磁盘，
这将减少每次更改读取IO的消耗。（如果索引包含降序索引列或主键包含降序索引列，则非主键索引不会进Change Buffer）。
通常情况下当我们在表上操作Insert时非主键索引列的值通常都是未排序的，需要大量IO保证最新状态。引入Change Buffer后 非主键索引的更改将进入其中，
而不是立即刷新到磁盘。当页面加载到Buffer Pool时会合并更改然后刷盘（适合大量的MDL操作：批量插入等）。
当然对于合并率较低的库库表操作我们可以通过参数innodb_change_buffering选择关闭或者调整Change Buffer来提升性能。它的设值有以下几种：

>* all：默认值：缓冲区插入，删除标记操作和清除。
>* none：关闭缓冲任何操作。
>* inserts：缓冲区插入操作。
>* deletes：缓冲区删除标记操作。
>* changes：缓冲插入和删除标记操作。
>* purges：缓冲在后台发生的物理删除操作。

当然根据库表的使用情况，通过innodb_change_buffer_max_size来设置Change Buffer总大小的百分比，默认25，最大50。
当有大量写入的操作时还可以调整innodb_change_buffer_max_size，否则就出现合并更改跟不上新产生的的条目，从而导致Change Buffer内存达到上限。
（可以根据实际情况动态设置）我们可以通过如下命名查看Change Buffer的状态：

```shell
mysql> SHOW ENGINE INNODB STATUS\G
```

Change Buffer状态位于INSERT BUFFER AND ADAPTIVE HASH INDEX 标题下，内容大致如下
```shell

-------------------------------------
INSERT BUFFER AND​ ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:insert 0, delete mark 0, delete 0
discarded operations:insert 0, delete mark 0, delete 0
Hash table size 4425293, used cells 32, node heap has 1 buffer(s)
13577.57 hash searches/s, 202.47 non-hash searches/s
```
#### Log Buffer ####
保存要写入磁盘上的日志文件的数据。日志缓冲区大小由innodb_log_buffer_size变量定义 。默认大小为16MB。Log Buffer的内容会定期刷新到磁盘。
较大的日志缓冲区使大型事务可以运行，而无需在事务提交之前将redo log数据写入磁盘。如果您有很多行的写事务，则增加Log Buffer的大小可以节省磁盘
的IO开销。innodb_flush_log_at_trx_commit 变量控制如何将Log Buffer区的内容写入并刷新到磁盘。该 innodb_flush_log_at_timeout 变量控制日志刷新频率。

#### Adaptive Hash Index  ####
该功能可以在InnoDB不牺牲事务功能或可靠性的情况下,在工作线程Buffer Pool有足够内存的的系统上充当内存数据库使用，使用innodb_adaptive_hash_index 变量启用。
自适应哈希索引是根据观察者模式查找由索引关键字前缀构建的哈希索引，该前缀可以是任何长度，可以是Hash Tree索引中仅B Tree中的某些值出现。
它是根据经常访问的索引页的需求而建立的 ，模糊查找与范围查找不适用，在并发情况下回造成资源争抢。由于很难预先预测自适应哈希索引功能是否适合特定的系统和工作负载，
因此请考虑启用和禁用该功能的基准测试。



参考阅读资料：
* 调整 Linux I/O 调度器优化系统性能:`https://www.ibm.com/developerworks/cn/linux/l-lo-io-scheduler-optimize-performance/index.html`
* Benefits of Using InnoDB Tables ：`https://dev.mysql.com/doc/refman/5.7/en/innodb-benefits.html`


