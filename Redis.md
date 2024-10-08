k-v形式存储对象，key指向string对象，v可以指向string对象，可以指向其他集合类型对象如list、set、hash等。
https://ls8sck0zrg.feishu.cn/wiki/IqzrwXjITiFZ6mkqQoHcsPq8nkg

1. #### Redis  版本5.0.5

   1. 主要应用场景：缓存与分布式锁

   2. 模块：对象、执行流程、持久化、场景、集群 （少）

      - 对象：String、Hash、Set、SortSet、List（五个基本类型）
      - 执行流程：内存结构、线程
      - 持久化：RDB、AOF
      - 场景：缓存、分布式锁、秒杀、限流、事务
      - 集群：主从、哨兵、集群

   3. 分布式系统的CAP
      三者最多同时满足两个
      [分布式系统的 CAP 定理与 BASE 理论-51CTO.COM](https://www.51cto.com/article/649412.html)

      - 一致性：分布式系统在完成某个写操作后的任何读操作，都应该获取到那个写入的最新值，即系统中每个节点时刻保持数据一致性
      - 可用性：系统可以一直进行读写操作，即用户可以正常访问并得到响应
      - 分区容错性：分布式网络中，当某个节点或网络分区故障了，整个系统还能对外提供满足一致性和可用性的服务（用户仍可以继续访问）
      - CA 即系统不再是扩展的，违背初衷
      - AP 放弃强一致性，保证最终一致性
      - CP 放弃可用性，在数据一致性比较严格的场合。一旦发生网络故障或者消息丢失，就会牺牲用户体验，等恢复之后用户才逐渐能访问

   4. BASE理论
      是对CAP理论中一致性和可用性的妥协，允许数据在一段时间内是不一致的，而最终达到一致状态。

      BA 基本可用：某个模块出现问题，而其他模块可以正常访问

      S 软状态：允许数据出现中间状态，即允许系统在多个不同节点的数据副本出现延迟

      E 最终一致性：所有客户端对系统的数据访问最终都能够获取到最新的值，而这个时间期限取决于网络延时，系统负载，数据复制方案等因素。


#### 1  数据类型

1. **redisobject对象**  **redis有6种对象，其底层依赖不同数据结构**
   type数据类型、encoding编码方式、ptr内容指针


##### **1.1 String**

- 多用于缓存，如Json、数字、二进制。计数，string可以自增自减（**INCR key命令进行自增**）

- 最大512M，但是超过100M的字符串就是大Key了，要考虑进行处理

- <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240121165850751.png" alt="image-20240121165850751" style="zoom:50%;" />

- setnx 即若数据存在，则不进行覆盖（锁），返回0，否则进行覆盖并返回1，通过set 的nx参数 `set key value ex seconds nx`可以完全取代setnx，ex为过期时间，xx只有键存在才进行操作

- mget一次获取多个值

- unlink可以进行异步删除（对于大对象效率性能好）

- `TTL key`查询过期时间，若已过期返回-2

- 底层编码实现（若数据发生变动三者之间会自动转变）：

  1. INT：存整型，（float等非整型按照长度使用下两个编码）

  2. EMBSTR：长度小于阈值就用EMBSTR，对象头和内容连续内存，一次性分配空间，使用特殊编码节省空间，**任何写操作（更新）后EMBSTR都会转变为RAW，因为重新分配空间效率太低，设计为只读。**

  3. RAW：长度大于阈值就用RAW，对象头和内容分开内存，适用于更新多的场景，使用字符数组存储

     <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240121173340248.png" alt="image-20240121173340248" style="zoom:67%;" />

     为什么要有sds：（**保存了字符串长度、二进制安全、预留了空间**）sds中有len，查长度时时间复杂度为1。alloc为分配的空间，alloc-len为空余空间，为后续追加数据留余地，**二进制安全**，兼容‘\0’特殊字符

##### 1.2 **List 双端链表**      

`lpush list1 s1 s2 s3` list1为list的表名

- 个数限制：2^64-1 基本无上限
- <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240122185951749.png" alt="image-20240122185951749" style="zoom:67%;" />
- LPUSH list a b c d 插入顺序是从左到右（即先插入a，最后插b），从头插入
- RPUSH 从尾部插入
- LREM list count value，**移除指定值**，count=0即移除所有，count>0从左往右，count<0从右往左，返回值为移除的数量 `LREM list 2 a`，遍历去删除，时间复杂度为o(N)
- DEL 同步删除，阻塞客户端，**删除list对象**
- UNLINK异步删除，单独开个异步线程去删除，即非阻塞的，**只是取消key在键空间的关联，不再能被查到，删除异步执行**
- LRANGE list start stop **读操作** 当 0 -1时查询所有数据
- **编码实现**(5.0.5编码全用QUICKLIST)
  1. ZIPLIST ：（条件：**对象中每个字符串长度小于64字节，对象元素小于512个**）类似于集合（在频繁插入删除时效率低，要移动索引），一次性分配空间,小于阈值使用ZIPLIST，**便于数据少时节约内存**,**压缩列表的数据在内存中紧密相连**，小数据量的时候遍历访问性能好
     **查询长度**：ziplist头部保存了zllen字段保存了节点的数量
     **查询指定数据的节点**：遍历压缩链表，每个entry中保存上一个节点数据长度prevlen（因此可以从后向前遍历）,encoding中还保存了本entry的长度（用于正向遍历）
     **增删数据**:从尾部添加节点时不会导致数据后移，在头部添加节点时会导致节点移动，而且可能会**连锁更新**（即每个节点向后移动后，要更新节点中的prevlen字段，而当一个节点大于254时下一个节点的prevlen可能会从1个字节变成5个字节，导致这个entry大小又产生变化，节点又需要移动更新，虽然概率很小）
  2. LINKEDLIST 双向链表，添加很方便，为了数据多时提高更新效率（**加快处理速度，牺牲一部分内存**）
  3. QUICKLIST 融合了ZIPLIST和LINKEDLIST。双向链表连接节点，每个节点又是一个ZIPLIST。**数据少的时候即只有一个节点，相当于ziplist，数据多时就用多个节点**
  4. listpack 用于解决ziplist可能导致的连锁更新。  
     **ziplist将存储上一个节点长度的prevlen放在本节点中**，所以上一个节点发生更新也会导致本节点连锁更新，因此**变为将长度存放在自己当前节点的尾部，只需要下一个节点向前查找字节找到element-tot-len的结束标志位置**，就不会发生连锁更新。element-tot-len存储本节点除了本字段的长度。当需要找当前元素的上一个元素时，从后向前查找每个字节，找到上一个节点的element-tot-len，既可以找到上一个节点的起始位置（**即找到上一个元素的element-tot-len，通过该偏移量定位到上一个元素的起始位置**）

##### 1.3 **Set**

无序不重复的字符串集合
场景：如关注的公众号集合，set还提供了查交并集的功能，可以实现查找共同关注。

- 增 `sadd setName aa bb cc`
- 删 `srem setName aa`  
- 读 查看元素是否存在 `sismember setName aa` 、查看set集合大小 `scard setName `、查看集合所有元素 `smember setName` 迭代器 `sscan setName cursor [Match pattern ] [Count i]`，cursor为起始位置，返回迭代出来的元素与下一次迭代开始的位置（游标），下次迭代时可以将本次返回的游标作为起始位置。set是无序的，不是根据顺序进行迭代。首次迭代时cursor值为0。`sinter setName1 setName2`返回交集。`sunion setName1 setName2`返回两个set并集。`sdiff setName1 setName2`返回第一个set中有，而第二个集合中没有的元素。
- 编码方式：intset hashtable
  1. intset:如果都是整数，且不超过512个，有序存储，使用二分法查找。
  2. hashtable:使用拉链法，查找性能很高，O(1)时间复杂度，hashtable的键值对中的值为null。

##### 1.4 **hash**  

k-v都为string的hash表。

- 增 `hset mapName k1 v1 k2 v2`  `hsetnx  mapName k1 v1`k1不存在时添加数据
- 删 `hdel mapName k1 k2` 删除对象中数据` del mapName`删除对象
- 查 `hgetall mapName` 查找所有数据 `hget mapName key`查找对应元素的value `hlen mapName`元素个数 `hscan mapName cursor [Match pattern] [Count i]`从指定游标位置迭代元素（当编码方式为ziplist时，hscan会返回所有元素）
- 编码方式：ziplist hashtable
  1. ziplist （**当元素小于64个字节，所有元素个数小于512个**）压缩列表，内存紧凑（节约内存，且因为数量少查询效率高），将k-v都当做entry存放在ziplist中，使用ziplist都不用计算哈希值，也不会遇到哈希冲突

1. **set与hash中的底层hashtable **  **即数组+链表的拉链法**（头插法而不是hashmap中的尾插法）

   快速索引，使用O(1)时间复杂度查找元素的key。
   dictht结构：**table**指向实际地址，可以看为一个数组。**size**元素个数（通常为2的幂）。**sizemask**为size-1（方便获取全为1的二进制数与hash值进行与位运算得到实际索引，即hash/sizemask取余运算），**used**表示hashtale中被使用的节点数量。**也可以看做为hashmap中的数组结构**

   **渐进式扩容**：hashtable对dictht进行了进一步封装为dict，便于渐进式扩容（当中创建dictht数组），即dict中有2个hashtable结构（dictht中数组的entry为一个链表结构），一般只使用其中的一个，另一个hashtable结构size为0，当触发扩容后两个hashtable同时使用。迁移过程：先为新的hashtable分配空间（新表的大小为大于原来表used两倍的第一个2次方幂），接着将迁移偏移量从-1设置为0（即将原来表的0索引位置开始迁移）表示迁移正式开始，每次增删改查操作都会将偏移量位置的数据迁移到新表，并且更新偏移量（**即每次操作的时候顺带迁移数据**）。如果没有操作就不发生数据迁移。
   **扩容时机**：hashtable中有负载因子（为used/size）当负载因子大于1时，如果此时服务器没有进行BGSAVE复制操作就发生扩容，当负载因子大于5时一定扩容。
   **缩容**：也是渐进式的缩容，也用负载因子进行控制，当负载因子小于0.1时，缩容的新表大小为第一个大于used的2次方幂。

   

##### 1.5 zset

[Skip List--跳表（全网最详细的跳表文章没有之一） - 简书 (jianshu.com)](https://www.jianshu.com/p/9d8296562806)  在插入数据的同时就建立了多机索引

[Redis基本类型学习之Sorted Set - 掘金 (juejin.cn)](https://juejin.cn/post/6994792326455361566)

#### 2. Redis如何运作

##### 2.1 在内存中如何存储

redis中数据以键值对的形式存放在内存中。

- redisDB结构：redisDB为数据库对象（有dict字典、expire字典），指向数据字典dictht（即hash对象结构），字典中存放的就是k-v（k为字符串即对象名字，v为任意类型的redis对象如string、list、hash），当添加对象时`set key value`key就存放在dictht的field中，value存放在val中，更新即更新val中对象，DEL删除即把key和val都删除
- 过期键：redis的过期键存放在expire字典中，该字典的key为String对象指针，val为对象的过期的时间戳
  <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240327152231148.png" alt="image-20240327152231148" style="zoom:50%;" />
- set a  b如何存储？ redis中使用字典来存放键值对，a存放在字典对应的索引上

##### 2.2 redis单线程模型

**redis的单线程是指接受客户端请求、解析请求、进行数据读写操作、发送数据给客户端这个过程是由一个主线程来完成的。**

- redis是单线程还是多线程？：Redis本身并不是单线程的，Redis在启动时会启动后台线程（2.0前有两个后台线程分别用于**关闭文件**、**AOF刷盘**，4.0后新增一个线程用于**异步释放Redis内存**（如unlink、flushall async非阻塞的删除））

  异步删除的好处是不会导致主线程阻塞，因此大key不要用del，而使用unlink。为这三个操作单独使用线程的原因就是这些操作很耗时，使用单独的线程处理避免主线程阻塞。

- redis为何选择单线程？：（**核心为redis在内存中处理很快、多线程太麻烦**）cpu并不是制约redis性能的瓶颈，redis在内存中读写操作执行速度很快，更多是受到内存大小和网络I/O的限制，多线程并不会对处理逻辑的速度带来很大的提升。而且引入多线程是很复杂的，要**保证线程安全，如加锁控制执行顺序**，带来额外的开销，**如多线程带来额外的上下文切换**。
- redis为什么单线程还这么快？：单线程Redis度吞吐量可以达到10w/s。**redis是在内存中操作，本身就很快**。**redis的数据结构非常高效，并且做了优化**，如list在数据量少的时候使用ziplist，数据量多的时候使用quicklist。**redis采用了多路复用能并发处理大量的客户端请求**（使用epoll_wait监听socket的状态），实现高吞吐
- 介绍IO多路复用：
  - socket：当客户端请求到来时使用accept建立连接，使用recv获取套接字中的请求，**socket模型的accept和recv都是默认阻塞模式的**，当建立连接时间过长，或者recv时数据一直没有到达，整个redis服务都会被阻塞住，
  - IO多路复用：因此需要使用非阻塞模式的IO多路复用进行监听IO的状态，**Redis对IO多路复用进行了包装为Reactor模型，即监听IO状态，当事件发生将事件分发给不同的处理器（连接处理、写处理、读处理）**，如果没有请求到来也并不会阻塞，而是去做其他的事情。不会因为一个连接阻塞导致其他连接没法处理，使得Redis有了一定的并发能力。
- redis6.0为什么引入了多线程？：（负责读取、解析、回包）网络IO成为了Redis的瓶颈，因此引入了多线程来处理网络的IO（accept创建socket、recv解析请求参数、发送回包），而reids命令执行逻辑依然使用单线程。多线程处理读写请求，将命令通过队列传递给主线程处理。主线程和子线程都进行连接、接受、解析、回包，执行redis命令只有主线程进行。多线程比单线程性能提高了一倍左右（20w）
  子线程默认是关闭的，需要设置多线程数量开启多线程，多线程默认用来socket写请求给客户端，读请求需要手动开启thread-do-reads yes，在redis.conf配置中修改。

##### 2.3 内存淘汰机制

当redis存储超过maxmemory阈值，则会触发内存淘汰机制。

1. 默认为不开启淘汰策略，内存满了再写入会报错，淘汰的分为LRU（回收长时间没使用的）、LFU（回收最不常用的）、random（随机回收）、TTL（回收存活时间短的），这四种每个又分为针对全部回收、针对设置了淘汰机制的回收，因此共8种。
2. 淘汰时机：**每次读写命令的时候会去查看内存是否到达了阈值，并进行淘汰一部分内存**。
3. LRU：**删除最久没访问的key**，需要维护每个key最近访问时间。
   传统的LRU：基于链表，元素按照访问顺序从前向后排列，当元素被访问，则元素被移动到链表投，当内存淘汰时，删除链表尾部的元素。但是维护这个链表空间成本高，大量数据的移动操作很耗时。
   redis采用近似LRU算法：**在redis对象结构体中添加最近一次访问时间的字段**，**淘汰时随机取5个值，淘汰其中最久没使用的，若淘汰后内存依然不足，那么继续淘汰**。不用额外维护一个链表。
   淘汰池优化：随机抽取的数可能不是全局最久未访问的，因此优化的LRU算法维护一个大小为16的候选池（为一个数组，按照空闲时间排序），当pool没满时，将key填充到池子中直到池子满了，当满了时，抽取到的key的lru要小于pool中最大的lru才放入pool中，并将池子中的置换出来（**即维护的都是最长时间没访问的数据**），**然后每次都淘汰池子中最后一个键**，淘汰完了再看是否内存充足，否则继续采样。（效果接近与严格lru）
4. LFU **删除最不常使用的**（使用频率低的）
   lru可能会存在问题，缓存污染（即大量的不常用的数据进入缓存，在淘汰时可能最常用的但是最近没有访问到的数据被淘汰）
   redis对象头中有lru字段（24位），**lru和lfu共用该字段（lfu时只用前8位表示访问计数，高16位为上次访问时间戳，精度到分钟）**，由这两个字段综合考虑，当上次访问时间太久时，访问计数衰减，**每超过1分钟，访问计数减1**。
   当某个key被访问时更新lfu，第一步计算计数衰减，第二步一定概率增加访问计数（避免过度关注访问次数，即因为访问频率过低直接被删除，避免热点数据过多占用缓存空间），第三步更新访问时间与访问计数，淘汰策略与lru的淘汰池一样。

##### 2.4 过期删除机制

对于设置了过期时间的key。

- 定期删除：每隔一段时间随机从过期字典（键为key，值为过期时间）中取出20个key检查，如果其中过期的键占比大于25%（即5个），那么还会再抽出来20个检查，但这个循环不会超过25ms）。Redis中采用的是**定期删除+惰性删除结合**。
- 定时删除：在设置key的过期时间的同时，创建一个定时时间，当时间到达时由事件处理器执行key的删除操作
- 惰性删除：不主动删除，每次访问key时才检测key是否过期，如果过期则删除key。

#### 3. 持久化

RDB和AOF，**RDB记录某个时刻压缩的数据快照** **，AOF以日志形式记录命令**（重启后重做命令来恢复数据）。
二者对比：**RDB的体积小和恢复速度更快，AOF的数据完整性更好** ，当数据完整性要求高时使用AOF与RDB结合，接受几分钟的数据丢失则使用RDB，**RDB是默认打开的**，不建议单独使用AOF（因为数据快照总是最有效的）。**==AOF与RDB都使用后台线程并通过写时复制技术保证持久化时主进程依然可以执行命令==**（即数据依然能被修改）

1. 为什么要持久化？：为了在服务重启的时候能够保留缓存数据，保证数据的完整性，持久化可以用于数据恢复，防止重启后缓存不再，请求都打到数据库中了。


##### 3.1 RDB

配置 `save 300 10` 每隔几分钟（通常大于5min），fork一个子进程去持久化刷盘，**在redis.conf中配置多个save interval num,即每间隔interval秒，有num个写操作，就触发持久化**，默认开启了持久化。
持久化时机：存在`save interval num`的默认配置触发RDB持久化

- 生成RDB文件：

  - save使用主线程生成RDB文件，如果执行时间太长会导致主线程阻塞，除了关闭redis时基本不用。
  - bgsave后台使用子线程生成RDB文件，避免主线程阻塞，
  - `save interval num`自动触发RDB持久化（使用的时bgsave后台线程）

  达到redis.conf中的阈值，即周期函数检查在interval间隔内有num个写操作，会触发bgsave

  正常关闭redis会使用save

- 子进程RDB时，主线程依然可以继续处理操作，**通过写时复制子进程和父进程共享一片内存区域，只有当发生了内存数据修改时，主线程会复制一份内存，以此实现快照，==子线程持久化的依然是修改前的数据==**，在复制期间，读数据相互不影响，若发生写，则主线程复制一份内存，在复制的内存中修改数据，并修改主进程页表的映射。

##### 3.2 AOF

（流程、刷盘、重写）

每执行完一个写操作命令，就把该命令以追加写的方式写到日志文件中。（先执行命令再记录日志），由主线程完成

  - 刷盘策略： 每次刷盘 每秒刷盘 系统刷盘
    <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240329101552737.png" alt="image-20240329101552737" style="zoom:50%;" />
  - **写AOF流程**：
    - 将每个命令执行完后写入AOF缓存、
    - write系统调用将缓存刷到内核缓冲区（在每个时间处理完后、周期函数、关闭服务器或AOF时都会刷）、
    - fsync根据刷盘策略将缓存区的数据刷盘（Always每个命令、Everysec每秒、NO由系统决定）

  - ==AOF重写==：为了防止AOF不断变大，会对AOF进行重写，即对相同key的操作进行合并，因为重写会读取所有的键值对为每个键值对生成一条命令是一个耗时的过程，不能在主线程进行，**因此会开启一个后台进程**，根据当前数据库中的键值对状态转换成命令，**写一个新的AOF文件并替换原来的AOF文件**。fork出一个子进程，遍历Redis中数据中数据以命令的形式写入新的AOF文件中，**为了防止写时复制后主进程与子进程的数据不一致，所以在重写期间的写命令除了会写到AOF缓存还会写到AOF重写缓冲中，新的AOF写完后将重写缓冲中的命令追加到新的AOF中。在重写的期间对数据的修改会通过写时复制（因为重写时物理内存为只读的，所以对物理内存进行复制后主进程再写），而不是在原数据上修改**
  - **写时复制**：子进程会复制一份父进程的页表（是虚拟地址与物理地址的映射），而不是将物理内存复制一份，二者实际上用的是同一个物理内存，只有在写操作时主进程才会复制物理内存进行修改并更该主进程页表的映射，而没修改的物理内存还是与子进程共享。复制页表比复制物理内存阻塞的时间小很多。使得在子进程重写的过程中，主进程依然可以正常处理命令。
    <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240510190102280.png" alt="image-20240510190102280" style="zoom:50%;" />

4. AOF与RDB对过期键如何处理？

   重载RDB时，主服务器会过滤掉已经过期的键（即过期的键不会被加载到主服务器中），从服务器在加载RDB时，不会对过期键进行过滤，而是先加载到服务器中，再通过同从同步，删除过期键。

   AOF文件是追加写，如果过期键被删除，会追加DEl命令，当加载AOF文件时会过滤过期键

##### 3.3 大key对持久化影响

- 对AOF影响：AOF在主线程执行。根据AOF刷盘策略，当使用Always时会等到fsync刷盘完成后才会返回，因为数据刷盘过程是耗时的所以阻塞时间会比较久。Everysec时为异步执行fsync的（不是每次都刷盘，隔一秒才刷盘）所以不会影响主线程。当使用No时同理不会影响主线程。
- 对AOF重写和RDB影响：AOF重写和RDB都是使用子进程来执行的。在写时复制时，即如果父进程对共享内存中的大key进行了修改，会复制一份物理内存，这个过程是阻塞的，那么对于大key物理内存是比较大的，复制物理内存时就会阻塞比较长时间。
- 除了对持久化有影响：redis执行命令是单线程的，执行大key的命令会比较耗时，就会阻塞redis。网络阻塞。
- 如何解决：在设计阶段将大key拆分成一个个小key。定期检查是否有大key，若大key是可以删除的，使用unlink异步删除。

#### 4. 系统

##### 4.1 **socket模型**

插口，**用于跨主机的进程间的通信**，服务端先调用**socket()**函数，指定创建网络协议IPV4，传输协议为TCP的socket，调用**bind()**函数，将socket绑定Ip地址和端口（用于指定接受的网卡地址，与程序接受数据的端口），接着调用**listen()**函数来进行监听端口（即监听socket），服务端再调用**accept()**函数来获取客户端连接（会阻塞直到客户端连接到来），客户端创建好socket之后，调用**connect()**函数发起连接（其中参数指定了服务端的Ip地址与端口号），**监听的socket收到服务段的connect请求后会创建新的socket与客户端建立连接（即TCP三次握手）**。服务端中维护了两个socket队列(TCP半连接队列、TCP全连接队列)，accept()函数会从TCP全连接队列中拿出一个已经完成三次握手的socket返回给应用程序，后续数据的传输都使用这个socket。socket分为监听socket和已连接socket。**连接建立后双方就可以通过read()和write()进行读写数据**

##### 4.2如何服务更多的用户

（即建立更多的连接）socket模型中连接的过程是阻塞的，连接请求的瓶颈在文件描述符不能超过1024，系统内存会被TCP的连接占满，但真正限制连接更多请求的地方在于IO模型。

###### 4.2.1 多进程模型

在阻塞的IO中，要服务器连接多个客户端，使用多进程模型，主进程监听连接请求，accept()函数生成一个已连接Socket，再通过fork()生成一个子进程，将主进程的东西都复制一份，使用子进程与客户端通信。当客户端连接数很多时，因为对每个客户端都产生一个进程，占据系统资源，且进程的上下文切换很性能很低的。

###### 4.2.2 多线程模型

线程是进程中的逻辑流，但进程可以运行多个线程，同一个进程中的线程会共享进程资源，因此多线程在切换上下文时开销很小。同时使用线程池避免频繁的创建与销毁线程，当accept()新建已连接Socket时，将socket放入队列中，线程池中线程再从队列中取socket与客户端通信，这个队列是需要加锁的，但是当客户端过多时，创建太多的线程操作系统也是扛不住的

###### 4.2.3  IO多路复用

**一个进程同时监听多个IO事件**，不会阻塞在某个请求中，**如使用select、poll、epoll多路复用接口从内核中获取有事件发生的Socket进行处理**（select、poll、epoll区别在于获准备好数据socket文件句柄的方式）

- select：将已连接socket放入一个文件描述符集合，调用select()函数将这个集合复制到内核中，让内核来检查是否有IO事件产生，**内核通过遍历的方式检查**，如果有事件，则标记该socket，遍历完再把整合集合复制回用户态中，用户态遍历集合找到可读可写的socket进行处理（**发生两次集合的遍历和拷贝**），**调用select是阻塞的，直到有socket文件描述符就绪或者超时**。返回就绪的socket的个数，程序还是得通过遍历集合而才知道具体就绪的socket。
  `int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);`n表示长度，三种类型文件描述符集合，超时时间
- poll与select的区别是，select使用固定长度的集合，而poll使用链表的形式，动态变化长度
  **缺点**：select和poll十分相似，只是文件描述符集合不同，内核中也是采用轮询的方式标记socket，**二者的缺点是socket文件描述符集合要在用户空间和内核空间进行复制（因为内核会修改状态位来标记），开销会随着集合数量增长而变大，导致性能下降**
  `int poll (struct pollfd *fds, unsigned int nfds, int timeout);`fds为结构体数组，nfds为集合中文件描述符的数量
- epoll：**epoll的系统调用有三个函数epoll_create、epoll_ctl、epoll_wait**，使用epoll_create创建epoll对象（**在内核建立epoll的红黑树和链表**），再使用epoll_ctl将socket对象放入epoll中监听或删除或修改（**将socket文件描述符放到红黑树上，并注册该socket的回调函数，当监听到socket就绪，调用回调函数将socket文件描述符放到链表中**），再调用epoll_wait等待发生了事件的socket（**只监听链表，将链表中就绪的socket拷贝回用户空间，少量的复制**）。epoll中使用红黑树来保存待检测的socket（**由于不修改socket的状态位，不用每次都传入整个集合，减少了拷贝与内存的消耗，直到使用epoll_ctl删除了该socket**），epoll中维护一个链表来记录就绪的事件，当某个socket有事件发生，将其加入到链表中，用户调用epoll_wait时不需要扫描整个socket集合，提高检测效率。epoll不会因为连接的Socket数量上升而导致性能降低。
  优点：
  - 与poll一样没有socket数量限制，
  - select与poll每次系统调用都要拷贝socket集合到内核空间（因为内核中会修改socket状态为，每次都要重新拷贝），十分低效，而epoll_ctl每次只需要向内核中加入新的socket句柄
  - select和poll内核都是主动轮询集合或者链表判断是否就绪的socket，而epoll是被动的通过回调函数将就绪的socket放到链表，epoll_wait只需要监听链表就可以判断是否有socket就绪

#### 5. 场景题

##### 5.1缓存异常

**穿透**：数据库和缓存中都没有。接口校验、缓存null值（设置短的过期时间，防止数据存在而无法查询）、使用布隆过滤器（在发现缓存没有后查数据库之前进行过滤，不能完全避免穿透）
**击穿**：热点数据缓存中没有而数据库中有（某个热点数据过期了（如秒杀活动的库存数据），导致大量请求访问该热点数据打到了数据库，数据库容易被高并发请求冲垮）。
		热点数据支持续期（只要有请求访问，就增加过期时间，避免热键过期）
		加互斥锁
**雪崩**：大量缓存同时过期或者Reids宕机，请求直接打到数据库上。
		均匀过期时间，在设置过期时间时加上一个随机数（防止大量数据同时过期），
		业务处理请求时如果数据不在Redis中，加互斥锁，保证同一时间只有一个请求能构建缓存（即只有一个请求访问数据库，将数据更新到缓存中），构建完成再释放锁，其他请求加个睡眠循环获得缓存。
		加上消息队列，发现缓存没有数据就发送给消息队列要求更新缓存，消费者先检查缓存是否存在，不存在则查数据库并更新缓存。

<img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240511115918508.png" alt="image-20240511115918508" style="zoom:50%;" />

##### 5.2 布隆过滤器

用于如：重复元素判断、去重、缓存穿透
使用一个很长的二进制数组，key使用不同的n个hash函数进行映射，将映射到对应下标值设置为1，若检测时存在映射到的某位置为0则该数据一定不再集合中，可以检测**绝对不在集合中的元素与可能在集合中的元素**（即过滤器决定不存在则集合中一定没有，过滤器觉得存在则集合中可能有）

##### 5.3 缓存一致性

即MySQL数据库和Redis缓存中数据不一致，

- （**更新数据无论数据库与缓存哪个先都有一致性问题**）无论是**先更新MySQL再更新Redis**还是**先更新Redis再更新Mysql**，在高并发时都可能会出现数据不一致的问题，而且更新缓存成本高（**有的缓存由好几张表组成，更新一次缓存可能要查好几张表**，而且**并不是所有的缓存都是频繁访问的，可能更新了缓存后很长时间都会访问**），==因此应当删除缓存==。

  - 在某些要求缓存命中有很高要求的场景中，如果删除缓存会导致缓存未命中，因此还是需要更新缓存。所以为了控制写操作的顺序，在**更新时可以先加分布式锁，保证同一数据同一时刻只有一个写请求运行**，但是会对性能产生影响。还可以**给缓存加上较短的过期时间**，即使数据不一致很快也会过期

  ​	<img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240413141417019.png" alt="image-20240413141417019" style="zoom:33%;" /><img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240413141431860.png" alt="image-20240413141431860" style="zoom:33%;" />

- （**删除缓存**，先操作数据库再删除缓存，本身缓存有过期时间，删除缓存是为了减少不一致的时间，==即旁路缓存==）因此不再更新缓存，而是删除缓存。写时先写数据库，再删除缓存，读时先读缓存，缓存中没有再读数据库并写入缓存，返回给用户。

  - 在写的过程中先删缓存还是先写数据库呢？<img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240413155058347.png" alt="image-20240413155058347" style="zoom:33%;" />
    **在读写并发时会出现数据库中数据为新值，缓存中为旧值**
    - 解决办法：延迟双删，删除缓存，更新完数据库后sleep一段时间，再将缓存重新删除，sleep保证读操作完成，再重新将读操作造成的旧数据的缓存删除
  - 因此先更新数据库再删除缓存，在读写并发时，因为写缓存往往会比写数据库快，所以很少会发生B已经删除完缓存，A才写缓存的现象、

- 如果先更新数据库，在删除缓存中的删除缓存失败了如何避免（如何保证两个操作都成功）？
  <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240511111931066.png" alt="image-20240511111931066" style="zoom:33%;" />
  
  - 可以引入**消息队列**，将要删除的数据加入消息队列，由消费者来操作删除缓存，如果删除失败，消费者会从消费队列中重新读数据，再次删除缓存，**重试机制**，删除成功则把数据从消息队列中移除。
  
  - 另起一个专门更新Redis的服务（canal组件将binlog传递到MQ中，服务订阅该MQ进行操作Redis），作为子节点订阅Mysql的binlog日志，再解析日志更新Redis（**和业务完全解耦，更新Mysql时业务不用操作Redis，没有时序性问题**）适用于缓存基本不过期的场景，也很少变化，如视频网站和对于热点数据流量很高的。

##### 5.4 分布式锁

​	用于在分布式环境下控制并发访问，控制某个资源在同一时刻只能被一个应用访问。redis的set命令有NX参数表示key不存在时才插入，若key不存在，插入成功（表示加锁成功），若key已经存在了，插入失败（表示枷锁失败）。

加锁

- 要保证读锁变量、检查锁变量、设置锁变量三个操作时原子性的，所以使用set加NX来实现。

- 要给锁设置过期时间，防止客户端拿到锁之后，发生异常导致锁无法释放。所以在set中加EX/PX来设置过期时间

- 要防止将别人的锁解了，因此加锁的时候每个客户端设置的Value是唯一的，标识客户端。

  `SET lock_key unique_value NX PX 10000 `

解锁

- 解锁先判断是不是自己加的锁，是的话才解锁。这两个操作要保证原子性，使用Lua脚本保证解锁的原子性，

  ```c
  // 释放锁时，先比较 unique_value 是否相等，避免锁的误释放
  if redis.call("get",KEYS[1]) == ARGV[1] then
      return redis.call("del",KEYS[1])
  else
      return 0
  end
  ```

如果锁超时释放了不就两个客户端同时操作一个数据吗？

- 给锁设置合理的超时时间，在锁要过期时使用守护线程（Redisson中有看门狗机制，在客户端启动守护线程在到达过期时间2/3时使用Lua脚本重置过期时间，如果客户端宕机了那守护线程自然也就没有了，时间到了就释放锁）

可靠性？

- 主次容灾，主节点挂了从节点顶包，但是会出现问题，因为主从复制是异步的存在延时。
  ![image-20240413185750009](C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240413185750009.png)

- 多机部署

  同一份数据在多个机器上操作（加锁也对多个机器的redis一起加），比如RedLock，获取锁与解锁时，只有超过一半的机器返回成功才算加锁成功，（加锁失败）否则要向每个redis发送解锁命令，解锁时向每个redis发送解锁请求。好处时如果挂了几台机器，整个集群还可以用

- 就一定可靠了吗？
  NPC问题 P：如果线程暂停如GC，那么看门狗机制会失效，超时锁就被释放了，此时会出现两个服务同时获得锁。N：网络延迟，当返回包的时间过长，可能加锁成功，锁很快就过期了，如果锁时间为10s，但回包了11s，锁实际上失效了。 C：时钟漂移，客户端和reids服务器的时间不一样，可能导致A不知道自己的锁过期了，B又获得了锁，此时AB都可以去访问了。

##### 5.5 高可用

[主从复制是怎么实现的？ | 小林coding (xiaolincoding.com)](https://xiaolincoding.com/redis/cluster/master_slave_replication.html#第一次同步)

1. 主从复制
   机器故障了怎么办？ 主从模式，将数据备份到从机上，主节点坏了可以用从节点提供服务。**主从复制保证主从节点的数据一致性**，主从读写分离，写只在主节点上，主节点再将数据同步到从节点，从节点只提供读服务。主从复制的模式：**全量复制（连接、RDB同步、缓冲同步）、基于长连接的命令传播、增量复制**

   - 配置主从模式，即主从复制将主节点数据同步给从节点，当主节点挂了从节点上还有数据，将某个节点设置为其他节点的从节点 `127.0.0.1:6380> salveof 127.0.01:6379` 将某个从节点取消 `127.0.0.1:6380> salveof no one`，接着就会进行第一次同步

   主从如何同步数据？ 

   - **第一次同步**：

     - 第一阶段：建立连接、协商同步。从服务器发送psync 命令。这个命令包括两个参数从服务器的ID，复制进度-1表示进行全量复制，主服务器收到命令后，使用FULLRESYNC响应，包含参数主服务器的ID和主服务器复制进度
     - 第二阶段：全量复制，主服务器启动bgsave生成RDB文件发给从服务器（这个过程是异步的），从服务器会清空当前数据，载入RDB文件，为了保证主从数据一致，主服务器在生成RDB文件的同时会将新收到的写命令写到缓冲区replication buffer中。
     - 第三阶段：主服务器将缓冲的写命令发送给从服务器。从服务器将RDB文件载入完成后，会恢复一个确认消息给主服务器，主服务器再将缓冲区replication buffer中的数据发送给从服务器
       <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240511122514675.png" alt="image-20240511122514675" style="zoom:50%;" />

   - **长链接命令传播**：在第一次同步后，会维护一个TCP长连接，主服务器之后收到写操作发给从服务器

   - **增量复制**：**当网络断开又恢复后，即网络延时，如何保证主从节点一致**，**增量复制将网络断开期间主节点收到的写操作同步给从节点**。网络恢复后从节点会发送psync命令给主节点，参数为从节点id和复制进度offset（此时不为-1），主节点收到后，使用continue命令响应从节点表示接下来使用增量复制同步，接下来将断开期间的数据发给从节点

     主节点如何知道发送哪些增量数据？主服务器在命令传播到从节点时，还会将命令写到replication backlog buffer中，这是一个循环写的缓冲区，当从节点重写连接时，会将其复制进度偏移量发送给主节点，**主节点会通过自身的offset偏移量和从节点的偏移量进行比较**，如果在backlog该数据已经被新的写命令覆盖了即backlog缓冲区没有该数据了，就会采用**全量同步**的方式，否则存在的话就会使用**增量同步**将数据写到repilcation缓冲区，传播给从服务器。backlog大小应当至少设定为重连接的平均时间*主节点每秒收到的写命令数量，避免被覆盖造成全量同步
     <img src="C:\Users\10122\AppData\Roaming\Typora\typora-user-images\image-20240414225434840.png" alt="image-20240414225434840" style="zoom:50%;" />

   - 分摊主节点压力：当有很多个从节点全量同步时，对每个从节点生成RDB文件和传输RDB会对主服务器产生压力，fork创建子进程是阻塞的，传输RDB会占用主服务器的网络带宽，因此可以将一个从服务器当成经理的角色，其他从节点都连接这个经理从节点，将创建传输RDB的压力可以分摊到经理从节点上，即将从服务器 `salveof <经理从节点><端口号>`。

   - 怎么判断Redis某个节点是否正常工作？Redis主节点每10s ping一次从节点，判断从节点是否还存活，Redis从节点每1s发送一次offset给主节点，检查复制是否丢失。
   - replication buffer和backlog buffer区别？ replication buffer是在全量复制时缓存新的写命令与增量复制，每个从节点都会对应一个replication buffer，backlog buffer在主节点只会有一个。backlog buffer是循环写的结构，会覆盖。replication buffer写满了就会重新连接，从节点重新全量复制。
   - 主从不一致怎么办？即客户端从从节点读到的数据和主节点中最新值不一样，因为主从复制是异步进行的，无法保证强一致性，可以开启一个程序监控INFO replication命令查看主节点和从节点的复制进度，如果二者的差值超过我们设定的阈值，可以让客户端停止请求该从节点，减少不一致的情况。
   - 主从切换如何减少数据丢失？：**异步复制丢失**，主节点还没来得及同步给从节点的数据会丢失，配置参数min-slaves-max-lag x，主从数据复制和同步的延迟**不能超过 x 秒**使得当从节点与主节点的数据差（即主节点同步给从节点所需要的时间）超过配置的阈值，主节点就拒绝写请求，客户端发现主节点不可用时可以将数据缓存起来或写到消息队列中，等到主节点恢复再从队列中消费请求。**脑裂产生数据丢失**，如果主节点与从节点连接丢失了，但是主节点和客户端还连接正常，客户端会正常向主节点写请求，但是无法同步给从节点，哨兵选举出了新的主节点，原来的主节点恢复后会变成从节点，进行全量复制从而清理了自己本地的数据，从而导致数据丢失。可以当主节点发现从节点失去连接数量过多或延迟太大时，禁止客户端的写操作，min-slaves-max-lag x，主从数据复制和同步的延迟**不能超过 x 秒**，min-slaves-to-write x，主节点必须要有**至少 x 个从节点连接**
   - 主从如何主动故障切换？使用哨兵模式，哨兵发现主节点故障会重新选举出一个新的主节点并通知给客户端。

   

2. 哨兵模式

   主节点挂了是手动设置从节点为主节点吗？即从节点如何升级为主节点，**Redis中有哨兵模式，检测Redis节点故障，并进行故障转移，==本质也是一个Redis进程==**，（集群多机部署中有多个主节点，每个主节点的主从复制都可以有哨兵集群），**主要工作为：监控、选主、通知**

   - 如何判断主节点挂了：哨兵会检测主节点是否存活(每一秒发送ping命令给所有节点)，若发现节点在规定时间内没有响应哨兵的ping命令，哨兵就会将该节点标记为主观下线，一般会有多个哨兵，当某个哨兵将主节点主观下线了，就会给其他哨兵发送命令，其他哨兵会进行投票，当超过一半的哨兵赞成，那么主节点就会被标记为客观下线
   - 由哪个哨兵进行故障转移（选主）：哨兵集群中选出一个leader对Redis主从切换，将主节点判断为客观下线的哨兵为候选者，候选者给其他哨兵发送命令要求投票，候选者可以投给自己，当收到大于一半的赞成就选主成功，
   - 主从切换并通知：选举出哨兵leader后，就可以进行主从故障转移。
     - 先选出新的主节点：向一个状态良好的从节点（优先级、复制进度、ID）发送 `slaveof no one `将其变为主节点
     - 从节点指向新主节点： 向其他从节点发送 `salveof 新主节点`命令
     - 通知客户端：通过发布订阅的方式，每个客户端都订阅了哨兵的消息。客户端收到消息后就会用新的IP地址和端口进行通信了。
     - 将旧主节点变为从节点：监控旧主节点，当其重新上线时，哨兵将其变为从节点，
   - 如何进行哨兵集群与发现从节点：配置哨兵命令 `sentinel monitor mastername ip port quorum`只需要主节点名字、主节点IP、端口号、quorum。哨兵节点之间通过主节点上的发布订阅机制来相互发现组成集群进行通信。同时哨兵会每10s给主节点发送INFO命令来获得从节点链表以监控从节点。

##### 5.6 秒杀场景

redis做秒杀场景的应用方式主要有将redis作为消息队列进行削峰处理（作为轻量级的消息队列，但是不如kafka可靠），二是将redis记录库存，进行库存的加减。**即redis扮演减少库存的角色**
高并发：**使用消息队列或redis实现，将抢和购解耦，不至于让Mysql过度承压，可以先将库存预加载到redis中，然后在redis中减扣，减扣成功后再通过消息队列传递到Mysql中做真正的订单生成**。redis的QPS可以达到上万，当请求数量超过了6w/s就要考虑使用多个redis进行分流（使用Nginx进行负载均衡，均匀的将请求分布到不同的redis上）
拒绝超卖：抢购时需要先判断库存是否还有，然后再减扣库存。但是需要保证这两个步骤是原子性的（不然可能先判断还有库存，但是减扣的时候实际上库存已经没有了）。因此使用lua脚本将这两个操作整合。
拒绝少卖：可能库存实际上减少了，但是并没有进去生成订单的流程，也可能redis操作成功，但是向消息队列发送失败，也会白白减少库存。即需要保证redis库存和kafka消费的最终一致性。在生成kafka消息失败时可以进行重试保证一定会被kafka消费，甚至可以将这条消息落盘慢慢重试保证安全。

##### 5.7 消息队列

消息队列一般用于异步流程、消息分发、流量削峰等。接入维护一个kafka比较繁重，且其实有时候并不需要非常完善与可靠的消息队列，redis适合用来做轻量级的消息队列。

- 基础版本：用list做消息队列，生产者RPUSH list v1 v2;，消费者LPOP list，实现先入先出的生产消费。
- 阻塞式消费：但是消费者无法知道什么时候LPOP，只能不停的轮询，因此使用BLPOP，为阻塞版本的pop直到到达超时时间BRPOP list 10.
- ACK确认机制：消费者取消息后，消息就出队列了，但是如果消费失败还得把消息重写放回队列，且不支持多人消费。先LRANGE查队列中消息，消费完成后再pop，就可以实现ACK与多人消费。但是还是没法实现消费组（即消息只能被该消费组中的消费者消费）
- pub/sub发布订阅模式：redis提供了发布订阅模式来实现消息的传递，`subscribe channel`来订阅某个频道。 `publish channel message`用来发布消息到某个频道。但是同样没有ACK功能（消费成功了进行ACK后才能消费下个消息）

##### 5.8 限流器

限制请求流量，防止数据库被冲垮。
[基于Redis的分布式限流器Java实现-CSDN博客](https://blog.csdn.net/GaleZhang/article/details/126493108)

- 计数器：记录请求次数，设定每秒请求的阈值，超过阈值就拒绝请求，直到下一秒再重新计数。单机可以使用AtomicInteger整数+时间戳（通过拦截器的方式对每个请求判断）。分布式限流中，使用LUA脚本保证检查阈值与阈值+1的原子性。使用set key val nx ex。但是会有请求突刺（在1.5-2.5这一秒内超过了阈值，但是在0-1,1-2中并没有超过）
- 滑动窗口，是计数器的升级版，解决了请求突刺，基于当前时间前的一段时间内是否超过阈值，而不是将时间分为一段一段（0-1,1-2,2-3......）。使用zset有序集合，key为操作数，score为时间戳，每次根据当前时间使用zcount key start end来查前一段时间区间内得请求数量。
- 漏桶，桶的流入如果超过限制即桶满了就不解释请求了，桶的流出是匀速的，从而做到均匀流量，但是可能无法充分用满性能资源。无法面对突发流量。
- 令牌桶：桶里均匀产生令牌，如果令牌把桶塞满了，就不产生令牌，请求需要先获得令牌再进行处理。如果桶里没有令牌了就得等待或放弃，令牌桶可以在流量小的时候积累令牌，当爆发请求时，可以一次性将桶中令牌拿完，允许不超过桶中令牌数的突发请求。







 











得物





















