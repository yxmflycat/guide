# Redis数据结构详解
作者：阿茂

## 基础数据结构
Redis有5钟基础数据结构，也是最常用的，分别是：string，list，hash，set，zset。下面我们就面试中经常会被问到的问题进行解释

- 字符串（String）：Redis的字符串是动态字符串，是可以修改的。采用预分配冗余空间的方式来减少内存的频繁的分配，当字符串长度小于1M时扩容都是加倍现有空间，如果超过1M扩容的时候只会增加1M空间，字符串类型结构最大长度为512M。
- 列表（list）：它的内部实现是链表，插入和删除速度相当快，查找元素就的时间复杂度就是O(n)了，当list弹出最后一个元素，自身将被删除并回收内存。可以根据ltrim的两个参数（ start_index 和 end_index ）来实现一个定长链表。
- 字典（hash）：相当于Java里面的HashMap，使用数组+链表的方式实现，它是无序的。不同的是Redis的hash结构值只能是字符串，另外它的rehash的方式也不一样。因为Java的HashMap的rehash的时候要全量复制，而且是一个阻塞的动作，所以Redis明显不能这么干。Redis采用渐进式rehash，渐进式rehash会在rehash的同时保留新旧两个hash结构，查询的时会同时查询两个hash结构，然后在后续的定时任务以及hash的子指令中逐渐地将旧数据一点点迁移到新的hash结构中。它不同于字符串需要一次性全部序列化整个对象，可以单独对每个字段单独存储。所以hash的存储消耗是高于单个字符串的，在使用的时候根据场景进行考量。
- set集合：它相当于Java里的HashSet，内部键值是无序唯一的。它相当于一个特殊的Hash，Hash中的value都是NULL
- zset：zset数据结构面试中是最多会问到的，因为它可以实现很多场景。它类似于Java中的SortedSet和HashMap的结合。用Set的特性保证了value的唯一性，还提供了给每个value添加一个权重字段score。使用score字段可以对value进行自定义排序。它内部使用跳跃列表数据结构实现。
### 数据结构进阶篇
下面我们将更深入的了解一下这些数据结构还有一些其他我们常见到的数据结构，我这里也只是看书学习总结过来的，有说的不对的地方请大家指正

- 字符串：我能知道在C中标准的字符串是以NULL作为结束符的，想要获取字符串长度需要使用到strlen标准库函数，但是这个函数的时间复杂度是O(n),需要对整个字符数组进行遍历，作为单线程的Redis明细接受不了，所以就诞生了Simple Dynamic String，它是结构中自带长度信息，结构如下：
```uastcontextlanguage
struct SDS<T> {
    T capacity; // 数组容量
    T len; // 数组长度
    byte flags; // 特殊标识位
    byte[] content; // 数组内容
}
```
content里面存储的真正的内容，capacity-len就是冗余空间长度，如果数组没有冗余空间，那么append操作就会触发扩容。它是将分配一个新数组，然后将就数组复制过来，再append新内容，如果字符串非常大，那么内存分配的操作是相当消耗资源的。大部分情况下我们创建的字符串len和capacity是相等的，因为我们绝大情况下是不会使用append操作的。Redis的字符串有两种存储方式，在长度小于等于44的时候会使用enbeded方式存储，当长度超过44的时候会使用raw方式存储。为什么这两张方式的分界线是44呢？这个我们得先来了解下Redis对象头的结构体：
```uastcontextlanguage
struct RedisObject {
    int4 type; // 4bits
    int4 encoding; // 4bits
    int24 lru; // 24bits
    int32 refcount; // 4bytes
    void *ptr; // 8bytes，64-bit system
} robj;
```
不同的对象具有不同的类型type(4bit)，同一个类型的type会有不同的存储形式encoding(4bit)，为了记录对象的LRU信息，使用了24个bit来记录LRU信息。每个对象都有个引用计数，当引用计数为零时，对象就会被销毁，内存被回收。说到这里大家可以去复习下《什么是缓存LRU淘汰机制》这篇文章，帮助大家加深理解。