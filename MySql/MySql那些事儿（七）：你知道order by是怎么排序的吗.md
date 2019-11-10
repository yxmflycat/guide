#MySql那些事儿（七）：你知道order by是怎么排序的吗？
作者：阿茂

从这一片开始我们就说一下平时是遇到的问题以及其原理解析，看到标题有没有朋友遇到过sql加了order by速度很慢的呢？下面我们来看了例子，建表语句入下：
```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `city` varchar(16) NOT NULL,
  `name` varchar(16) NOT NULL,
  `age` int(11) NOT NULL,
  `addr` varchar(128) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `city` (`city`)
) ENGINE=InnoDB;
```
我们的排序语句入下
```mysql
select city,name,age from t where city='杭州' order by name ;
```
##全字段排序
作为一个优秀的crud程序员，为了避免全部扫描我们都会在city字段上建立索引，然后我们用explain来看看我们的执行计划

 ![](../resource/order%20by.png)
 
 Extra这个字段中的“Using filesort”表示的就是需要排序，MySQL会给每个线程分配一块内存用于排序，称为sort_buffer。
 下面我们梳理下执行流程
 > 1. 初始化sort_buffer，确定放入name、city、age这三个字段；
 > 2. 从索引city找到第一个满足city='杭州’条件的主键id；
 > 3. 到主键id索引取出整行，取name、city、age三个字段的值，存入sort_buffer中；
 > 4. 从索引city取下一个记录的主键id；
 > 5. 重复步骤3、4直到city的值不满足查询条件为止；
 > 6. 对sort_buffer中的数据按照字段name做快速排序；
 > 7. 返回结果集给客户端
 
“按name排序”这个动作，可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数sort_buffer_size。它就是MySQL为排序开辟的内存（sort_buffer）的大小。如果要排序的数据量小于sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

## rowid排序
通过上面的例子大家都应该能看的出来，当查询行的数据量很大的时候，需要每次都都要从查询出来的数据放到sort_buffer中，这样内存中只能放下有限数量的数据，要分很多的临时文件用来辅助排序，性能就会很差。总结下来就是：当查询单行数据量较大的时候这个方法就不说最优选择。那么用什么方式呢？这就不得不说到max_length_for_sort_data 这个参数了
>max_length_for_sort_data：MySQL中专门控制用于排序的行数据的长度的一个参数。如果单行的长度超过这个值，MySQL就认为单行太大，要换一个算法。

city ->varchar(16)、name -> varchar(16) 、age ->int(11) 这三个字段的定义总长度是43，我把max_length_for_sort_data设置为16，我们再来看看计算过程有什么改变 :新的算法放入sort_buffer的字段，只有要排序的列（即name字段）和主键id。排序的结果就因为少了city和age字段的值，不能直接返回了，整个执行流程就变成如下所示的样子：
> 1. 初始化sort_buffer，确定放入两个字段，即name和id；
> 2. 从索引city找到第一个满足city='杭州’条件的主键id；
> 3. 到主键id索引取出整行，取name、id这两个字段，存入sort_buffer中；
> 4. 从索引city取下一个记录的主键id；
> 5. 重复步骤3、4直到不满足city='杭州’条件为止;
> 6. 对sort_buffer中的数据按照字段name进行排序；
> 7. <font color=#FF0000 > 按照id的值回到原表中取出city、name和age三个字段返回给客户端结果集。</font>

全字段排序算法中你会发现，rowid排序多访问了一次表t的主键索引。需要说明的是，最后的“结果集”是一个逻辑概念，实际上MySQL服务端从排序后的sort_buffer中依次取出id，然后到原表查到city、name和age这三个字段的结果，不需要在服务端再耗费内存存储结果，是直接返回给客户端的。