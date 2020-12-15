
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


