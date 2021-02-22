
- redis中有一个「核心的对象」叫做redisObject
  - ![20210220112119](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210220112119.png)
    - 是用来表示所有的key和value的，用redisObject结构体来表示String、Hash、List、Set、ZSet五种数据类型。
    - type表示属于哪种数据类型，encoding表示该数据的存储方式
      - ![20210220112710](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210220112710.png)
  - String类型的数据结构存储方式有三种int、raw、embstr。
    - int
      - Redis中规定假如存储的是「整数型值」，比如set num 123这样的类型，就会使用 int的存储方式进行存储，在redisObject的「ptr属性」中就会保存该值。![20210220104436](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210220104436.png)
    - raw: 字符串是一个字符串值并且长度大于32个字节
      - 使用SDS（simple dynamic string）方式进行存储
      - 且encoding设置为raw
    - embstr: 字符串长度小于等于32个字节
      - 使用SDS（simple dynamic string）方式进行存储
      - 且encoding设置为embstr
        - SDS（simple dynamic string）
          - len保存了字符串的长度，
          - buf数组则是保存字符串的每一个字符元素。
          - free表示buf数组中未使用的字节数量
          - ![Redsi中存储一个字符串Hello时](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210220111805.png) 
            - c语言字符串与SDS对比:
              - 获取长度的时间复杂度为O(n) | 获取长度的时间复杂度为O(1)
              - 不是二进制安全的 | 是二进制安全的
              - 只能保存字符串 | 还可以保存二进制数据
              - n次增长字符串必然会带来n次的内存分配 | n次增长字符串内存分配的次数<=n
  - hash
    - ziplist: 一组连续的内存空间的使用，节省空间.
      - 编码的哈希对象使用压缩列表作为底层实现， **每当有新的键值对要加入到哈希对象时， 程序会先将保存了键的压缩列表节点推入到压缩列表表尾， 然后再将保存了值的压缩列表节点推入到压缩列表表尾**
      - ![20210220144824](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210220144824.png)
    - hashtable
      - 编码的哈希对象使用字典作为底层实现， 哈希对象中的每个键值对都使用一个字典键值对来保存：
        - 字典的每个键都是一个字符串对象， 对象中保存了键值对的键；
        - 字典的每个值都是一个字符串对象， 对象中保存了键值对的值。
      - ![20210220145440](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210220145440.png)
        - Redis 的**字典**使用**哈希表**作为底层实现， 一个哈希表里面可以有**多个哈希表节点**， 而每个哈希表节点就保存了字典中的**一个键值对**。
          - 哈希表:```typedef struct dictht {```
          - 哈希表节点: ```typedef struct dictEntry {```
          - 字典: ```typedef struct dict {```
            - hash算法:
              - 使用字典设置的哈希函数，计算键 key 的哈希值```hash = dict->type->hashFunction(key);```
              - 使用哈希表的 sizemask 属性和哈希值，计算出索引值; 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]```index = hash & dict->ht[x].sizemask;```
            - 解决键冲突: **拉链法**
              - ![20210220152248](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210220152248.png)
            - hash rehash:
              - 一般情况下， 字典只使用```ht[0]```哈希表，```ht[1]```哈希表只会在对```ht[0]```哈希表进行 rehash 时使用。
              - 除了```ht[1]```之外， 另一个和 rehash 有关的属性就是 rehashidx ： 它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 -1 。
              - 步骤：
                - 1. 为字典的 ht[1] 哈希表分配空间， 这个哈希表的空间大小取决于要执行的操作，以及 ht[0] 当前包含的键值对数量（也即是ht[0].used 属性的值）：
                  - 如果执行的是扩展操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used * 2 的 2^n （2 的 n 次方幂）；
                  - 如果执行的是收缩操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n 。
                - 2. 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面： rehash 指的是重新计算键的哈希值和索引值， 然后将键值对放置到 ht[1] 哈希表的指定位置上。
                - 3. 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后 （ht[0] 变为空表）， 释放 ht[0] ， 将 ht[1] 设置为 ht[0] ， 并在 ht[1] 新创建一个空白哈希表， 为下一次 rehash 做准备。
              - 当以下条件中的任意一个被满足时， 程序会自动开始对哈希表执行扩展操作：
                - 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 1 ；
                - 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 5 ；
                - 负载因子: ```负载因子 = 哈希表已保存节点数量 / 哈希表大小   ====>   load_factor = ht[0].used / ht[0].size```
              - 哈希表渐进式 rehash 的详细步骤：
                - 1. 为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
                - 2. 在字典中维持一个索引计数器变量 ```rehashidx``` ， 并将它的值设置为 0(**当没有rehash的时候，它是-1**) ， 表示 rehash 工作正式开始。
                - 3. 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 **```rehashidx```(这个变量当前的index是多是，比如0,1,2...) 索引上的所有键值对 rehash 到 ht[1]** ， 当 rehash 工作完成之后， 程序将``` rehashidx ```属性的值增一。
                - 4. 随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成。
    - 编码转换
      - 当哈希对象可以**同时满足**以下两个条件时， 哈希对象使用 ziplist 编码：
        - 哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节；
        - 哈希对象保存的键值对数量小于 512 个；
          - 这两个条件的上限值是可以修改的， 具体请看配置文件中关于 ```hash-max-ziplist-value``` 选项和 ```hash-max-ziplist-entries``` 选项的说明。
      - 不能满足这两个条件的哈希对象需要使用 hashtable 编码。




- 跳跃表 ( skiplist )
  - what:
    - 一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。
    -  Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员 ( member ) 是比较长的字符串时，Redis 就会使用跳跃表来作为有序集合键的底层实现。
  - why:
    - 首先，因为 zset 要支持随机的插入和删除，所以它 不宜使用数组来实现，关于排序问题，我们也很容易就想到 红黑树/ 平衡树 这样的树形结构，为什么 Redis 不使用这样一些结构呢？
      - 性能考虑： 在高并发的情况下，树形结构需要执行一些类似于 rebalance 这样的可能涉及整棵树的操作，相对来说跳跃表的变化只涉及局部 (下面详细说)；
      - 实现考虑： 在复杂度与红黑树相同的情况下，跳跃表实现起来更简单，看起来也更加直观；
  - how:
    - ![20210220203234](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210220203234.png)
    - https://lotabout.me/2018/skip-list/#%E8%B7%B3%E8%A1%A8%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%80%9D%E6%83%B3



## other

- SDS与c语言字符串对比:
   - Redis使用SDS作为存储字符串的类型肯定是有自己的优势，SDS与c语言的字符串相比，SDS对c语言的字符串做了自己的设计和优化，具体优势有以下几点：
     - 1. c语言中的字符串并不会记录自己的长度，因此「每次获取字符串的长度都会遍历得到，时间的复杂度是O(n)」，而Redis中获取字符串只要读取len的值就可，时间复杂度变为O(1)。
     - 2. 「c语言」中两个字符串拼接，若是没有分配足够长度的内存空间就「会出现缓冲区溢出的情况」；而「SDS」会先根据len属性判断空间是否满足要求，若是空间不够，就会进行相应的空间扩展，所以「不会出现缓冲区溢出的情况」。
     - 3. SDS还提供「空间预分配」和「惰性空间释放」两种策略。
       - 在为字符串分配空间时，分配的空间比实际要多，这样就能「减少连续的执行字符串增长带来内存重新分配的次数」。
         - 具体的空间预分配原则是：(这里说的是在保证字符串的长度后，还需要预先分配额外的空间大小)
           - **「当修改字符串后的长度len小于1MB(字符串的长度小于1MB)，就会预分配额外和len一样长度的空间，即len=free；若是len大于1MB，free分配额外的空间大小就为1MB」。**
       - 当字符串被缩短的时候，SDS也不会立即回收不适用的空间，而是通过free属性将不使用的空间记录下来，等后面使用的时候再释放。
     - 4. SDS是**二进制安全的**，除了可以**储存字符串以外还可以储存二进制文件**（如图片、音频，视频等文件的二进制数据）；而c语言中的```字符串是以空字符串作为结束符，一些图片中含有结束符，因此不是二进制安全的```。





 https://www.w3cschool.cn/hdclil/lun1dozt.html









一致性，高可用这两个方面来弄。

- 1. redis基础知识
  - 数据库的个数?
    - 1. Redis默认有16个数据库；可以用SELECT来切换数据库；
    - 2. 当redis-cli连上server，默认是0号数据库，个个数据库之间相互隔离，允许相同的key存在。
  - 清空数据
    - flushdb 清空当前数据库
    - flushall 清空所有数据库的数据
  - CPU/MemeryIO/NetIO
    - Redis是基于内存操作的，CPU不是Redis的性能瓶颈，Redis的瓶颈是根据机器的内存和网络带宽
      - redis快的原因？
        - redis所有的数据全部放在内存，是直接操作内存的。
        - 它的瓶颈不是CPU，用单线程避免CPU上下文切换。
        - Redis是C语言写的。
- 2. redis五大类型，三种特殊类型
  - 五
    - string(字符串)
    - list(列表)
    - hash
    - set(集合)
    - zset(有序集合)
  - 三
    - Geospatial(地理位置)
    - HyperLogLog(基数统计)
    - Bitmaps(位存储)

- 3. **Redis单条命令是保证原子性的，但是Redis的事务是不保证原子性的！**
  - **Redis事务没有隔离级别的概念。**
  - **Redis事务：**
    - **开启事务(MULTI)**
    - **命令入队**
      - 如果命令入队的时候就错了（比如命令错了，命令参数错了），所有的都不会执行.
    - **执行事务(EXEC)**
      - 但是执行的时候，出错了，拿它之前的命令已经是执行完毕了，不能够回滚。
    - TODO / watch unwatch

- 4. redis持久化
  - RDB(Redis DataBase):
    - what:
      - RDB(Redis Database) 是通过快照(snapshot)的形式**将数据保存到磁盘中**。
        - 所谓快照，可以理解为在某一时间点将数据集拍照并保存下来。Redis 通过这种方式可以在指定的时间间隔或者执行特定命令时将当前系统中的数据保存备份，以二进制的形式写入磁盘中，默认文件名为dump.rdb。
    - why:
      - 使用RDB的优劣
        - 优势：
          - 文件格式的角度：RDB 是一个非常紧凑（compact）的文件（保存二进制数据），它保存了 Redis 在某个时间点上的数据集
            - 适合用于进行备份:比如说，你可以在最近的 24 小时内，每小时备份一次 RDB 文件，并且在每个月的每一天，也备份一个 RDB 文件。
            - 非常适用于灾难恢复（disaster recovery）：它只有一个文件，并且内容都非常紧凑，可以（在加密后）将它传送到别的数据中心。
          - 保存的方式的角度：
            - RDB 可以最大化 Redis 的性能：父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作；
        - 劣势：
          - 发生故障停机， 就可能会丢失好几分钟的数据
            - 虽然 Redis 允许在设置不同的保存点（save point）来控制保存 RDB 文件的频率， 但是， 由于 RDB 文件需要保存整个数据集的状态， 所以这个过程并不快，可能会至少 5 分钟才能完成一次 RDB 文件保存
    - how:
      - how it works?
        - fork一个子进程
          - Redis为了不阻塞线上业务，所以需要一边持久化一边响应客户端的请求，因此fork出一个子进程来处理这些保存工作。
        - COW机制
          - 那么具体这个fork出来的子进程是如何做到使得Redis可以一边做持久化操作，一边做响应工作呢？这就涉及到COW (Copy On Write)机制。
            - Redis在持久化的时候会去调用glibc的函数fork出一个子进程，快照持久化完成交由子进程来处理，父进程继续响应客户端的请求。而在子进程刚刚产生时，它其实使用的是父进程中的代码段和数据段。所以fork之后，kernel会将**父进程**中所有的内存页的权限都设置为read-only，然后子进程的地址空间指向父进程的地址空间。当父进程写内存时，CPU硬件检测到内存页是read-only的，就会触发页异常中断（page-fault），陷入 kernel 的一个中断例程。中断例程中，kernel就会把触发的异常的页复制一份，于是**父子进程各自持有独立的一份**。而此时子进程相应的数据还是没有发生变化，依旧是进程产生时那一瞬间的数据，故而子进程可以安心地遍历数据，进行序列化写入磁盘了。
            - 随着父进程修改操作的持续进行，越来越多的共享页面将会被分离出来，内存就会持续增长，但是也不会超过原有数据内存的两倍大小（Redis实例里的冷数据占的比例往往是比较高的，所以很少出现所有页面都被分离的情况）。
            - COW机制的好处很明显：首先可以减少分配和复制时带来的瞬时延迟，还可以减少不必要的资源分配。但是缺点也很明显：如果父进程接收到大量的写操作，那么将会产生大量的分页错误（页异常中断page-fault）。
      - how to use?
        - save触发
          - Redis是单线程程序，这个线程要同时负责多个客户端套接字的并发读写操作和内存结构的逻辑读写。而save命令会阻塞当前的Redis服务器，在执行该命令期间，Redis无法处理其他的命令，直到整个RDB过程完成为止，当这条指令执行完毕，将RDB文件保存下来后，才能继续去响应请求。这种方式用于新机器上数据的备份还好，如果用在生产上，那么简直是灾难，数据量过于庞大，阻塞的时间点过长。这种方式并不可取。
        - bgsave触发
          - 为了不阻塞线上的业务，那么Redis就必须一边持久化，一边响应客户端的请求。所以在执行bgsave时可以通过fork一个子进程，然后通过这个子进程来处理接下来所有的保存工作，父进程就可以继续响应请求而无需去关心I/O操作。
        - redis.config 配置
          - ```save <seconds> <changes>```
            - 自动化的触发机制,使用bgsave
          - ```save ""```禁止掉数据持久化
        - 执行shutdown命令关闭服务器时，如果没有开启AOF持久化功能，那么会自动执行一次bgsave
        - 主从同步（slave和master建立同步机制）
          - ![20201210144331](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201210144331.png)

  - AOF(Append Only File):
    - what: AOF日志是持续增量的备份，是基于**写命令**存储的可读的文本文件。
      - AOF日志:
        - AOF只记录对内存进行修改的指令记录。
        - 先执行指令**再将日志存盘**。
        - 弱一致性。
    - why:
      - 为什么要有AOF：RDB 持久化是全量备份，比较耗时。
      - 使用AOF的优劣
        - 优势：
          - 文件格式的角度：
            - 写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂
            - AOF 文件是一个只进行追加操作的日志文件（append only log）， 因此对 AOF 文件的写入不需要进行 seek ， 即使日志因为某些原因而包含了未写入完整的命令（比如写入时磁盘已满，写入中途停机，等等）， redis-check-aof 工具也可以轻易地修复这种问题。
            - 重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。
          - 保存的方式的角度：
            - AOF 持久化的默认策略为每秒钟 fsync 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多也只会丢失掉一秒钟内的数据；
        - 劣势：
          - 对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。
          - 根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB。
            - 在一般情况下， 每秒fsync的性能依然非常高， 而关闭fsync可以让AOF的速度和RDB一样快，即使在高负荷之下也是如此。不过在处理巨大的写入载入时，RDB可以提供更有保证的最大延迟时间（latency）。
    - how:
      - how to use?
        - 打开```appendonly yes```。
        - ```appendfsync always/everysec/no```默认是```appendfsync everysec```。
          - always：每次发生数据修改就会立即记录到磁盘文件中，这种方案的完整性好但是IO开销很大，性能较差；
          - everysec：在每一秒中进行同步，速度有所提升。但是如果在一秒内宕机的话可能失去这一秒内的数据；
            - 再将AOF配置为appendfsync everysec之后，Redis在处理一条命令后，并不直接立即调用write将数据写入 AOF 文件，而是先将数据写入AOF buffer（server.aof_buf）。
          - no：默认配置，即不使用 AOF 持久化方案。
        - 重写（rewrite）机制
          - what:**AOF Rewrite**“压缩”AOF文件的过程
            - **AOF Rewrite**并非采用**基于原AOF文件**来重写或压缩，而是采取了类似RDB快照的方式：
              - 基于Copy On Write，全量遍历内存中数据，然后**逐个序列**到AOF文件中。因此AOF rewrite能够正确反应当前内存数据的状态。
              - 重写过程中，对于新的变更操作将仍然被写入到原AOF文件中，同时这些新的变更操作也会被Redis收集起来。当内存中的数据被全部写入到新的AOF文件之后，收集的新的变更操作也将被一并追加到新的AOF文件中。然后将新AOF文件重命名为appendonly.aof，使用新AOF文件替换老文件，此后所有的操作都将被写入新的AOF文件。
          - why:AOF日志会在持续运行中持续增大，需要定期进行AOF重写，对AOF日志进行瘦身。
            - fsync函数：配置为appendfsync everysec之后，Redis在处理一条命令后，并不直接立即调用write将数据写入 AOF 文件，而是先将数据写入AOF buffer（server.aof_buf）。调用write和命令处理是分开的，Redis只在每次进入epoll_wait之前做 write 操作。
          - how:
            - 手动触发
              - ```redis-cli -h ip -p port bgrewriteaof```。
            - 自动触发
              - 打开```no-appendfsync-on-rewrite yes```。
              - ```auto-aof-rewrite-min-size```:表示运行AOF重写时文件最小体积，默认为64MB（我们线上是512MB）。
              - ```auto-aof-rewrite-percentage```:代表当前AOF文件空间（aof_current_size）和上一次重写后AOF文件空间（aof_base_size）的值
  - Redis 4.0 混合持久化
    - what: Redis 4.0引入。
      - 将 rdb 文件的内容和增量的 AOF 日志文件存在一起。这里的**AOF 日志不再是全量**的日志，而是自持久化开始到持久化结束的这段时间发生的**增量 AOF**日志，通常这部分 AOF 日志很小。相当于：
        - ![20201210172441](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201210172441.png)
        - 大量数据使用粗粒度（时间上）的rdb快照方式，性能高，恢复时间快。
        - 增量数据使用细粒度（时间上）的AOF日志方式，尽量保证数据的不丢失。
    - why:
      - 仅使用RDB快照方式恢复数据，由于快照时间粒度较大，时回丢失大量数据。
      - 仅使用AOF重放方式恢复数据，日志性能相对 rdb 来说要慢。在 Redis 实例很大的情况下，启动需要花费很长的时间。
    - how:
      - 在 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 AOF 日志就可以完全替代之前的 AOF 全量文件重放，重启效率因此大幅得到提升。
    - 混合持久化是最佳方式吗？
      - Master使用AOF，Slave使用RDB快照，master需要首先确保数据完整性，它作为数据备份的第一选择；slave提供只读服务或仅作为备机，它的主要目的就是快速响应客户端read请求或灾切换。



- 5. 高可用-高并发
  - 主从复制是哨兵和集群能够实时的基础，因此主从复制是Redis高可用的基础！
    - 主从复制:![20201210144331](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201210144331.png)
      - 数据冗余:主从复制实现了数据的热备份，是持久化之外的一种数据冗余方式。
      - 服务冗余:当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复。
      - 负载均衡：在主从复制的基础上，配合读写分离，可以由主节点提供写服务，从节点提供读服务，分担服务器负载。
    - 哨兵(sentry)
      - <img src="https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201210213656.png" alt="哨兵模式" style="zoom:80%;" />
      - **哨兵模式就是主从模式的升级，从手动到自动，更加健壮！**
    - 集群


- 6. 缓存现象
  - 6.1.缓存穿透(查不到)
    - 缓存层(N)-执行层(N); **缓存层找不到，执行层也找不到**，那么其实是浪费查询资源。
    - **single-key**
      - what: 假设用户想要查询一个数据，发现Redis内存数据库没有，也就是缓存没有命中，于是向关系型数据库查询。发现关系型数据库中也没有，于是本次查询失败。当用户很多的时候，缓存都没有命中，于是请求都到了持久层数据库。这会给持久层数据库造成很大的压力。这时候就相当于出现了缓存穿透。
      - why:
      - how:
        - how to solve?
          - 每次系统 A 从数据库中只要没查到，就写一个空值到缓存里去，比如 set -999 UNKNOWN。
            - 然后设置一个过期时间，这样的话，下次有相同的 key 来访问的时候，在缓存失效之前，都可以直接从缓存中取数据。
          - redis布隆过滤器
            - what:之前的布隆过滤器可以使用Redis中的位图操作实现，直到Redis4.0版本提供了插件功能，Redis官方提供的布隆过滤器才正式登场。布隆过滤器作为一个插件加载到Redis Server中，就会给Redis提供了强大的布隆去重功能。
            - why:

            - how:
              - how to use?
                - bf.add：添加元素到布隆过滤器中，类似于集合的sadd命令，不过bf.add命令只能一次添加一个元素，如果想一次添加多个元素，可以使用bf.madd命令。
                - bf.exists：判断某个元素是否在过滤器中，类似于集合的sismember命令，不过bf.exists命令只能一次查询一个元素，如果想一次查询多个元素，可以使用bf.mexists命令。
                - 在使用bf.add命令添加元素之前，使用bf.reserve命令创建**一个自定义的布隆过滤器**。bf.reserve命令有三个参数，分别是：
                  - key：键
                  - error_rate：期望错误率，期望错误率越低，需要的空间就越大。
                  - capacity：初始容量，当实际元素的数量超过这个初始化容量时，误判率上升。
                    - 布隆过滤器的error_rate越小，需要的存储空间就越大，对于不需要过于精确的场景，error_rate设置稍大一点也可以。布隆过滤器的capacity设置的过大，会浪费存储空间，设置的过小，就会影响准确率，所以在使用之前一定要尽可能地精确估计好元素数量，还需要加上一定的冗余空间以避免实际元素可能会意外高出设置值很多。总之，error_rate和 capacity都需要设置一个合适的数值。
                      - **默认的error_rate是 0.01，capacity是 100**

            ```shell
            #如果对应的key已经存在时，在执行bf.reserve命令就会报错。
            #如果不使用bf.reserve命令创建，而是使用Redis自动创建的布隆过滤器。
            #默认的error_rate是 0.01，capacity是 100。
            > bf.reserve one-more-filter 0.0001 1000000 # 这个如果没有这一句，只有下面的，那么使用redis自动创建的不隆过滤器。
            OK

            > bf.add one-more-filter fans1
            (integer) 1
            > bf.add one-more-filter fans2
            (integer) 1
            > bf.add one-more-filter fans3
            (integer) 1
            > bf.exists one-more-filter fans1
            (integer) 1
            > bf.exists one-more-filter fans2
            (integer) 1
            > bf.exists one-more-filter fans3
            (integer) 1
            > bf.exists one-more-filter fans4
            (integer) 0
            > bf.madd one-more-filter fans4 fans5 fans6
            1) (integer) 1
            2) (integer) 1
            3) (integer) 1
            > bf.mexists one-more-filter fans4 fans5 fans6 fans7
            1) (integer) 1
            2) (integer) 1
            3) (integer) 1
            4) (integer) 0
            ```

  - 6.2.缓存击穿(高并发,缓存过期)
    - 缓存层(N)-执行层(Y)；**缓存层找不到，执行层能找到**，那么压力都在执行层里面。
    - **single-key**
      - what:
      - why:
      - how:
        - how to solve?
          - **1.设置热点数据永不过期**
            - 从缓存层来看，没有设置过期时间，所以不会出现热点key过期后产生的问题。
          - **2.加互斥锁**
            - 分布式锁：使用分布式锁，保证对于每个key同时只有一个线程去查询后端微服务，其他线程没有获得分布式锁的权限，因此只需要等待即可。这种方式将高并发的压力转移到了分布式锁，因此对分布式锁的考验极大。

  - 7.3.缓存雪崩(缓存集体失效)
    - 缓存层(N)-执行层(Y)；**缓存层找不到，执行层能找到**，但是很多key都有这种情况，是一段时间内一堆key都失效了，那么压力都在执行层里面。
    - **multi-key**
      - what:
      - why:
      - how:
        - how to solve?
          - 事前：redis 高可用，主从+哨兵，redis cluster，避免全盘崩溃。
            - **Redis高可用**
              - 这个思想的含义是，既然Redis有可能挂掉，那么就多增加几台Redis服务器，这样一台服务器挂掉之后其他的还可以继续工作，其实就是搭建集群(异地多活)。
          - 事中：本地 ehcache 缓存 + hystrix 限流&降级，避免 MySQL 被打死。
            - **限流降级**
              - SpringCloud的服务降级，SpringCloudAlibaba的服务限流。
                - 限流组件，可以设置每秒的请求，有多少能通过组件，剩余的未通过的请求，怎么办？
                  - 走降级！可以返回一些默认的值，或者友情提示，或者空白的值。
          - 事后：redis 持久化，一旦重启，自动从磁盘上加载数据，快速恢复缓存数据。
            - **数据预热**
              - 在即将发生高并发访问前手动触发加载缓存不同的key，设置不同的过期时间，让缓存失效的时间尽量均匀！

[redis api document](http://redisdoc.com/pubsub/index.html)

[面试题redis-缓存](https://zhuanlan.zhihu.com/p/82980434)


