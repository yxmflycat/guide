# Redis数据结构详解
作者：阿茂

## 基础数据结构
Redis有5钟基础数据结构，也是最常用的，分别是：string，list，hash，set，zset。下面我们就面试中经常会被问到的问题进行解释

- 字符串（String）：Redis的字符串是动态字符串，是可以修改的。采用预分配冗余空间的方式来减少内存的频繁的分配，当字符串长度小于1M时扩容都是加倍现有空间，如果超过1M扩容的时候只会增加1M空间，字符串类型结构最大长度为512M。
- 列表（list）：它的内部实现是链表，插入和删除速度相当快，查找元素就的时间复杂度就是O(n)了，当list弹出最后一个元素，自身将被删除并回收内存。可以根据ltrim的两个参数（ start_index 和 end_index ）来实现一个定长链表。
- 字典（hash）：相当于Java里面的HashMap，使用数组+链表的方式实现，它是无序的。不同的是Redis的hash结构值只能是字符串，另外它的rehash的方式也不一样。因为Java的HashMap的rehash的时候要全量复制，而且是一个阻塞的动作，所以Redis明显不能这么干。Redis采用渐进式rehash，渐进式rehash会在rehash的同时保留新旧两个hash结构，查询的时会同时查询两个hash结构，然后在后续的定时任务以及hash的子指令中逐渐地将旧数据一点点迁移到新的hash结构中。它不同于字符串需要一次性全部序列化整个对象，可以单独对每个字段单独存储。所以hash的存储消耗是高于单个字符串的，在使用的时候根据场景进行考量。
- set集合：它相当于Java里的HashSet，内部键值是无序唯一的。它相当于一个特殊的Hash，Hash中的value都是NULL,问到较多的就是求交集，求并集，求差集。
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
不同的对象具有不同的类型type(4bit)，同一个类型的type会有不同的存储形式encoding(4bit)，为了记录对象的LRU信息，使用了24个bit来记录LRU信息。每个对象都有个引用计数，当引用计数为零时，对象就会被销毁，内存被回收。说到这里大家可以去复习下《什么是缓存LRU淘汰机制》这篇文章，帮助大家加深对LRU的理解。ptr指针指向对象具体的存储位置。这样一个Rdis的对象头部就占用了16byte。我们再看下SDS结构体的大小，字符串较小的时候：capacity+3。这就意味着字符串最小空间为19byte（16+3）。embstr它是将RedisObject和SDS对象连续存在一起，使用malloc方法一次分配。而raw需要两次，两个对象头不在一个连续的地址上。我们知道不管是jemalloc还是tcmalloc等分配内存的单位都是2^n。为了能容纳一个完整的embstr，jemalloc最少会分配32byte的内存，如果再长就是64，超过64Redis就要换存储方式了。在内存分配处的临界点是64，那么44是怎么来呢？我们刚刚说了Redis对象结构，其实留给内容的（content）的最大长度只有45（64-19），字符串又是以\0结尾的，所以证明出了embstr最大容纳字符串长度为44。

- 字典（dict）：字典结构内部包含两个hashtable，通常情况下只有一个hashtable 是有值的。但是在字典扩容缩容时，需要分配新的hashtable，然后进行渐进式搬迁，这时候两个hashtable 存储的分别是旧的 hashtable 和新的 hashtable。待搬迁结束后，旧的 hashtable 被删除，新的hashtable 取而代之。
- 压缩列表(ziplist)：zset和hash 容器对象在元素个数较少的时候，采用压缩列表进行存储。压缩列表是一块连续的内存空间，元素之间紧挨着存储，没有任何冗余空间。压缩列表为了支持双向遍历，所以才会有ztail_offset 这个字段，用来快速定位到最后一个元素，然后倒着遍历。
```uastcontextlanguage
    struct ziplist<T> {
        int32 zlbytes; // 整个压缩列表占用字节数
        int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
        int16 zllength; // 元素个数
        T[] entries; // 元素内容列表，挨个挨个紧凑存储
        int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
    }
    
    struct entry {
        int<var> prevlen; // 前一个 entry 的字节长度
        int<var> encoding; // 元素类型编码
        optional byte[] content; // 元素内容
    }
```
entry 块随着容纳的元素类型不同，也会有不一样的结构。它的prevlen字段表示前一个entry的字节长度，当压缩列表倒着遍历时，需要通过这个字段来快速定位到下一个元素的位置，它是一个变长的整数，当字符串长度小于254 时，使用一个字节表示；如果达到或超出254那就使用5个字节来表示。第一个字节是254，剩余四个字节表示字符串长度。encoding字段存储了元素内容的编码类型信息，ziplist通过这个字段来决定后面的content内容的形式。content字段在结构体中定义为optional类型，表示这个字段是可选的，对于很小的整数而言，它的内容已经内联到encoding 字段的尾部了。

关于压缩列表增加元素：因为压缩列表都是紧凑类型存储的，加入新元素就意味着扩内存并将就内容复制到新分配的内存上然后再追加。如果压缩列表占用内存太大，太多，频繁的重新分配内存与复制就会有很大的性能消耗。还有一个比较严重的问题：每个entry都有一个prevlen存储前一个enytry的长度。如果内容小于254，prevlen用1字节存储，否则就是 5 字节。这意味着如果某个 entry 经过了修改操作从 253 字节变成了 254 字节，那么它的下一个 entry 的 prevlen 字段就要更新，从 1个字节扩展到 5 个字节；如果这个 entry 的长度本来也是 253 字节，那么后面 entry 的prevlen字段还得继续更新，就会造成级联更新,我们再假设一种极端的情况，当压缩列表上都存储的是253byte的内容，那么第一个发生更新，想想一下后果。所以ziplist并不是节省空间的银弹。

- 小整数集合（IntSet）：当 set 集合容纳的元素都是整数并且元素个数较小时，Redis 会使用 intset 来存储结合元素。intset是紧凑的数组结构，同时支持16位、32位和 64位整数。结构如下：
```uastcontextlanguage
    struct intset<T> {
        int32 encoding; // 决定整数位宽是 16 位、32 位还是 64 位
        int32 length; // 元素个数
        int<T> contents; // 整数数组，可以是 16 位、32 位和 64 位
    }
```
- 快速列表（quicklist）：quicklist是ziplist和linkedlist的混合体，它将linkedlist按段切分，每一段使用ziplist来紧凑存储，多个 ziplist之间使用双向指针串接起来。结构如下：
```uastcontextlanguage
    struct quicklistNode {
        quicklistNode* prev;
        quicklistNode* next;
        ziplist* zl; // 指向压缩列表
        int32 size; // ziplist 的字节总数
        int16 count; // ziplist 中的元素数量
        int2 encoding; // 存储形式 2bit，原生字节数组还是 LZF 压缩存储
        ...
    }
    
    struct quicklist {
        quicklistNode* head;
        quicklistNode* tail;
        long count; // 元素总数
        int nodes; // ziplist 节点的个数
        int compressDepth; // LZF 算法压缩深度
        ...
    }
```
为了进一步节约空间，Redis 还会对ziplist 进行压缩存储，使用LZF算法压缩，可以选择压缩深度。quicklist内部默认单个 ziplist长度为8k字节，超出了这个字节数，就会新起一个 ziplist。ziplist的长度由配置参数list-max-ziplist-size决定。quicklist默认的压缩深度是0，也就是不压缩。

-跳跃列表：Redis的跳跃表共有64层，意味着最多可以容纳 2^64 次方个元素。每一个 kv 块对应的结构如下面的代码中的zslnode 结构，kv header 也是这个结构，只不过value字段是null值——无效的，score是Double.MIN_VALUE，用来垫底的。kv 之间使用指针串起来形成了双向链表结构，它们是有序 排列的，从小到大。不同的 kv 层高可能不一样，层数越高的 kv 越少。同一层的 kv会使用指针串起来。每一个层元素的遍历都是从kv header出发。
```uastcontextlanguage
    struct zslnode {
        string value;
        double score;
        zslnode*[] forwards; // 多层连接指针
        zslnode* backward; // 回溯指针
    }
    
    struct zsl {
        zslnode* header; // 跳跃列表头指针
        int maxLevel; // 跳跃列表当前的最高层
        map<string, zslnode*> ht; // hash 结构的所有键值对
    }
```
我们要定位一个kv,从header的最高层开始遍历找到第一个节点 (最后一个比「我」小的元素)，然后从这个节点开始降一层再遍历找到第二个节点(最后一个比「我」小的元素)，然后一直降到最底层进行遍历就找到了期望的节点 (最底层的最后一个比我「小」的元素)。我们将中间经过的一系列节点称之「搜索路径」，它是从最高层一直到最底层的每一层最后一个比「我」小的元素节点列表。有了这个搜索路径，我们就可以插入这个新节点了。不过这个插入过程也不是特别简单。因为新插入的节点到底有多少层，得有个算法来分配一下，跳跃列表使用的是随机算法。

- 紧凑了表（listpack）： Redis 5.0引入了一个新的数据结构listpack，它是对ziplist的结构的一种改造，在存储空间上更加节省，结构也精简了不少：
```uastcontextlanguage
    struct listpack<T> {
        int32 total_bytes; // 占用的总字节数
        int16 size; // 元素个数
        T[] entries; // 紧凑排列的元素列表
        int8 end; // 同 zlend 一样，恒为 0xFF
    }
    
    struct lpentry {
        int<var> encoding;
        optional byte[] content;
        int<var> length;
    }
```
首先这个listpack 跟ziplist 的结构几乎一摸一样，只是少了一个zltail_offset 字段。ziplist 通过这个字段来定位出最后一个元素的位置，用于逆序遍历。不过 listpack 可以通过其它方式来定位出最后一个元素的位置，所以 zltail_offset 字段就省掉了。元素的结构和 ziplist 的元素结构也很类似，都是包含三个字段。不同的是长度字段放在了元素的尾部，而且存储的不是上一个元素的长度，是当前元素的长度。正是因为长度放在了尾部，所以可以省去了 zltail_offset 字段来标记最后一个元素的位置，这个位置可以通过total_bytes 字段和最后一个元素的长度字段计算出来。长度字段使用 varint 进行编码，不同于ziplist 元素长度的编码为1个字节或者5个字节，listpack 元素长度的编码可以是1、2、3、4、5个字节。同UTF8编码一样，它通过字节的最高位是否为1来决定编码的长度。listpack的设计彻底消灭了ziplist存在的级联更新行为，元素与元素之间完全独立，不会因为一个元素的长度变长就导致后续的元素内容会受到影响。

### 结尾
以上就是Redis的大部分数据结构与原理，查资料实在太费时间了，这里我们给大家留个动手题吧：自己去翻阅一下资料看看：基数树 Radix Tree 这种Redis 的数据结构是怎么实现的？文中引用《Redis深度历险 ：核心原理和应用实践》内容，有兴趣一起阅读的朋友可以通过订阅号私信给我获取书籍。