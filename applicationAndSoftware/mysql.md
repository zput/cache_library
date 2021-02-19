# 0.重要的知识点

1. MySQL逻辑架构
  1.1. 连接层
  1.2. 服务层
  1.3. 引擎层(存储引擎是基于表的，而不是数据库)
  1.4. 存储层

2. 如何修改字符集(查看1.3)

3. 日志
  3.1. 查询日志
  3.2. 错误日志
  3.3. 二进制日志

- 4. 分析慢SQL的步骤
  - ~~first:找到是那些sql慢.~~
    - 4.1. 慢查询的开启，设置阈值(如超过5秒钟的就是慢SQL)并捕获。
  - ~~second:开始分析这些sql.~~
    - 4.2. explain + 慢SQL分析。
      - 查询语句写的差。
        - 关联 查询太多`join`（设计缺陷或者不得已的需求）。
      - 索引失效：索引建了，但是没有用上。
    - 4.3. show Profile查询SQL在MySQL数据库中的执行细节和生命周期情况。
    - 4.4. MySQL数据库服务器的参数调优。

5. SQL执行顺序

```shell
select              # 5 ---->
	... 
from                # 1
	... 
where               # 2
	.... 
group by            # 3
	... 
having              # 4 ---->
	... 
order by            # 6
	... 
limit               # 7
	[offset]
```

6. join
![如何写出join语句](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201203160509.png)


- 7. 索引
  - what: [A database index is a data structure that improves the speed of operations in a table](https://www.tutorialspoint.com/mysql/mysql-indexes.htm)
    - 索引分类：
      - 单值索引：一个索引只包含单个列，一个表可以有多个单列索引。
      - 唯一索引：索引列的值必须唯一，但是允许**空值**。
      - 复合索引：一个索引包含多个字段。
  - why:
    - 索引的优势和劣势
      - 优势：
        - 查找：类似大学图书馆的书目索引，提高数据检索的效率，降低数据库的IO成本。
        - 排序：通过索引対数据进行排序，降低数据排序的成本，降低了CPU的消耗。
      - 劣势：
        - 实际上索引也是一张表，该表保存了主键与索引字段，并指向实体表的记录，所以索引列也是要占用空间的。
        - 虽然索引大大提高了查询速度，但是同时会降低表的更新速度，例如对表频繁的进行`INSERT`、`UPDATE`和`DELETE`。因为更新表的时候，MySQL不仅要保存数据，还要保存一下索引文件每次更新添加的索引列的字段，都会调整因为更新所带来的键值变化后的索引信息。
        - 索引只是提高效率的一个因素，如果MySQL有大数据量的表，就需要花时间研究建立最优秀的索引。

  - how: 
    - **重点：索引会影响到MySQL查找(WHERE的查询条件)和排序(ORDER BY)两大功能！**
    - 什么时候需要建立索引：
      - 主键:主键自动建立主键索引（唯一 + 非空）。
      - where/order by:
        - 频繁作为查询条件的字段应该创建索引。
        - 查询中统计或者分组字段（group by也和索引有关）。
        - 查询中排序的字段，排序字段若通过索引去访问将大大提高排序速度。
      - 查询中与其他表关联的字段，**外键关系**建立索引。
    - 索引分析
      - 从9.1.单表索引分析-->范围之后的索引会失效。
      - 从9.2.两表索引分析--->左连接将索引创建在右表上更合适。
      - 10.1.索引失效的情况:
        - 全值匹配我最爱。
        - 最佳左前缀法则。
        - 不在**索引列**上做任何操作（计算、函数、(自动or手动)类型转换），会导致索引失效而转向全表扫描。
        - 索引中**范围条件**右边的字段会全部失效。
        - 尽量使用覆盖索引（只访问索引的查询，索引列和查询列一致），减少`SELECT *`。
        - MySQL在使用`!=`或者`<>`的时候无法使用索引会导致全表扫描。
        - `is null`、`is not null`也无法使用索引。
        - `like`以通配符开头`%abc`索引失效会变成全表扫描。
        - 字符串不加单引号索引失效。
        - 少用`or`，用它来连接时会索引失效。

- 8. 锁
  - what: 
    - InnoDB锁:
      - 1. 共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。
      - 2. 排他锁（X）：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。
      >>为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB 还有两种内部使用的意向锁（Intention Locks），这两种意向锁都是表锁：
      - 3. 意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的 IS 锁。
      - 4. 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的 IX 锁。
  
  >> ![20201204193446](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201204193446.png)
  >>（如果一个事务请求的锁模式与当前的锁兼容， InnoDB 就将请求的锁授予该事务； 反之， 如果两者不兼容，该事务就要等待锁释放。）
  
  - why:
    - 防止并发修改数据。
  - how:
    - InnoDB加锁方法：
      - 对于普通```SELECT```语句，InnoDB 不会加**任何锁**；
      - 对于 ```UPDATE、 DELETE 和 INSERT``` 语句， InnoDB会自动给涉及数据集加**排他锁（X)**；
      - 事务可以通过以下语句显式给记录集加共享锁或排他锁：
        - 共享锁（S）：```SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE```。 其他 session 仍然可以查询记录，并也可以对该记录加 share mode 的共享锁。但是如果当前事务需要对该记录进行更新操作，则很有可能造成死锁。
        - 排他锁（X)：```SELECT * FROM table_name WHERE ... FOR UPDATE```。其他 session 可以查询该记录，但是不能对该记录加共享锁或排他锁，而是等待获得锁
      - ~~意向锁是 InnoDB 自动加的， 不需用户干预。~~
  


# 表空间的格式

![InnoDB的物理结构](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210218140846.png)
segment和extent是InnoDB内部用于分配管理页的逻辑结构，用于分配与回收页，对于写入数据的性能至关重要。
 
但这张图有所局限性，可能会产生误解：
 
1）图中是系统表空间，因此存在rollback segment，独立表空间则没有。
2）leaf node segment实际是InnoDB的inode概念，一个segment可能包含最多32个碎片page、0个extent（用于小表优化），或者是非常多的extent，我猜测作者图中画了4个extent是在描述表超过32MB大小的时候一次申请4个extent。
3）一个extent在默认16k的page大小下，由64个page组成，page大小由UNIV_PAGE_SIZE定义，所以extent不一定由64个page组成。

表的所有行数据都存在页类型为INDEX的索引页（page）上，为了管理表空间，还需要很多其他的辅助页，例如文件管理页FSP_HDR/XDES、插入缓冲IBUF_BITMAP页、INODE页等。

## 页的结构

![innodb_files](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/innodb_files.png)

MySQL一次IO的最小单位是页（page），也可以理解为一次原子操作都是以page为单位的，默认大小16k。刚刚列出的所有物理文件结构上都是以Page构成的，只是page内部的结构不同。


### 文件管理页

- 文件管理页的页类型是FSP_HDR和XDES（extent descriptor），用于分配、管理extent和page。
  - FSP_HDR和XDES的唯一区别，
    - 它们两者除了```FSP Header```这个位置只在FSP_HDR有值，在XDES中是用0填充的；其他field都一样。都是类似的一堆结构(一个占40字节的XDES entry描述维护extent的)
      - ```FSP Header```这个field，它在XDES页中是空的```(zero-filled for XDES pages) ---> (112)```; page 0(FSP_HDR页面)中FSP_HDR中有值。
      - FSP_HDR页都是page 0，XDES页一般出现在page 16384, 32768等固定的位置。一个FSP_HDR或者XDES页大小同样是16K。
  - 一般情况下，每个extent都有一个占40字节的XDES entry描述维护，因此1个(FSP_HDR/XDES)页最多管理256个extent（也就是256M，16384个page）。那么随着表空间文件越来越大，就需要更多的XDES页。(所以才会出现在16384, 32768的位置)
    - FSP header:
      - 而FSP Header里面最重要的信息就是四个链表头尾数据（FLST_BASE_NODE结构，FLST意思是first and last），FLST_BASE_NODE如下。
        - 1）当一个Extent中所有page都未被使用时，挂在FSP_FREE list base node上，可以用于随后的分配；
        - 2）有一部分page被写入的extent，挂在FREE_FRAG list base node上；
        - 3）全满的extent，挂在FULL_FRAG list base node上；
        - 4）归属于某个segment时候挂在FSEG list base node上。
      - **当InnoDB写入数据的时候，会从这些链表上分配或者回收extent和page，这些extent也都是在这几个链表上移动的。**
    - XDES entry
      - XDES entry存储所管理的extent状态：
        - 1）FREE（空）
        - 2）FREE_FRAG（至少一个被占用）
        - 3）FULL_FRAG（满）
        - 4）归某个segment管理的信息
      - XDES entry还存储了每个extent内部page是否free（有空间）信息（用bitmap表示）。XDES entry组成了一个双向链表，同一种extent状态的收尾连在一起，便于管理。


![innodb_file_struct_manager](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/innodb_file_struct_manager.png)


### INODE页

- what:
  - ![innodb_file_struct_inode](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/innodb_file_struct_inode.png)
- why:

- how:
  - segment是表空间管理的逻辑单位。INODE页就是用于管理segment的，每个Inode entry负责一个segment。
    - 一个segment由32个碎片页（fragment array），FSEG_FREE、FSEG_NOT_FULL、FSEG_FULL组成，这些信息记录在Inode entry里，可以简单理解为Inode就是segment元信息的载体。
      - FREE、NOT_FULL、FULL三个FLST_BASE_NODE对象和FSP_HDR/XDES页里面的FSP_FREE、FREE_FRAG、FULL_FRAG、FSEG概念类似。这些链表被InnoDB使用，用于高效的管理页分配和回收。
      - 至于碎片页上（fragment array），用于优化小表空间分配，先从全局的碎片分配Page，当fragment array填满（32个槽位）时，之后每次分配一个完整的Extent，如果表大于32MB，则一次分配4个extent。

  - MySQL的数据是按照B+ tree聚簇索引（clustered index）组织数据的，每个B+ tree使用两个segment来管理page，分别是leaf node segment（叶子节点segment）和non-leaf node segment（非叶子节点segment）。
    - 这两个segment的Inode entry地址记录在B+ tree的root page中FSEG_HEADER里面，而root page又被分配在non-leaf segment第一个碎片页上（fragment array）。

### INDEX数据索引页


#### 聚簇和非聚簇

- what:
  - Clustered/Unclustered
    - Clustered
      - Index determines the location of indexed records
      - Typically, clustered index is one where values are data records (but not necessary)
    - Unclustered
      - Index cannot reorder data, does not determine data location
      - In these indexes: value = pointer to data record
    - ![20210218151054](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210218151054.png)
    - ![20210218151136](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210218151136.png)
- why:
- how:
  - 就是它的实际数据也是按照顺序排的，那么可以向上和向下查找，但是如果是非聚族的，它如果想找比它大一点点的值，你不可能从Data file里面就能找到，你必须回归到Data entries？
  - 使用B+树聚簇索引（B+ tree clustered index）的好处在于，
    - 1）数据和索引顺序一致，充分利用磁盘顺序IO性能普遍高于随机IO的特性。
    - 2）对于局部性查询也会大有裨益。
    - 3）采用B+树，叶子节点（leaf node）存储数据，**非叶子节点（non-leaf node）只是索引，这样非叶子节点就会足够的小(一个页面就能存储很多的节点，查找的时候，减少了磁盘io)**，因此数据很“热”，便于更好的缓存。
    - 4）对于覆盖索引，可以直接利用叶子节点的主键值。
  - 二级索引，就可以理解为非聚簇索引，也是一颗B+树，只不过这棵树的叶子节点是指向聚簇索引主键的，可以看做“行指针”，因此查询的时候需要“回表”。


#### index

- what:
  ![innodb_file_struct_index](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/innodb_file_struct_index.png)
  ![20210218185953](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210218185953.png)
  - 主键、二级索引、行和列
    - B+树的**每个节点(包括叶子和非叶子节点)**都是一个INDEX索引页，其结构都是相同的；
    - 对于聚簇索引，非叶子节点包含主键和child page number，叶子节点包含主键和具体的行；
    - 对于非聚簇索引，也就是二级索引，非叶子节点包含二级索引和child page number，叶子节点包含二级索引和主键值。
  - 叶子和非叶子都在index索引页，那么inode里面保存的segment信息，是不是这个segment用的所有页都能从这里找到，（先用的32个碎片页，然后也能从管理页中申请extent?）
- why:

- how:
  - page directory从Fil Trailer开始从后往前写，里面包含槽位slots，每个slot 2个字节，存储了某些record在该页中的物理偏移量，例如图中最后面是infimum record的offset，最前面是supremum record的offset，中间从后往前是r4，r8，r12 record的offset，一般总是每隔4-8个record添加一个slot，这样slots就等同于一个稀疏索引（sparse index），加速页内查询的办法就是通过二分查找，查询key的时间复杂度可以降为O(logN)，由于页都在内存里面，所以查询性能可控，内存访问延迟100ns左右，如果在CPU缓存中则可能更快。



##### Row Format

###### what:

row format可通过innodb_default_row_format参数指定，也可以在建表的时候指定。

```sql
CREATE TABLE choose_row_format (
   id INT,
) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;
```

- REDUNDANT: 是比较老的格式，
- COMPACT: 5.6版本默认，COMPACT比REDUNDANT要更节省空间，大概在20%左右。
- DYNAMIC: 5.7版本默认，DYNAMIC在变长存储上做了更大的空间优化，对于VARBINARY, VARCHAR, BLOB和TEXT类型更友好，
- COMPRESSED是压缩页。


###### what:
###### how:

> row的格式在上面图中简单介绍过，由可选的两个标识+record header+body组成，具体如下。

![innodb_file_struct_row_record](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/innodb_file_struct_row_record.png)

![20210218190555](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210218190555.png)



4）索引：序列化后存储于此，例如int类型索引主键就占用4个字节。
对于聚簇索引的叶子节点，存储行。
对于二级索引的叶子节点，存储行的主键值。
对于聚簇索引和二级索引的非叶子节点，存储child page最小的key。
上面提到的infimum和supremum中就只存字符串在行数据里。



[ppt](https://courses.cs.washington.edu/courses/cse444/09sp/lectures/lecture15.pdf)
[blog](http://neoremind.com/2020/01/inside_innodb_file/)




## 讨论innodb数据结构

- [索引最终选择B+树的原因](https://zhuanlan.zhihu.com/p/113917726):
  - hash很快，但每次IO只能取一个数(范围查找不适合);
    - 哈希索引只需要计算一次就可以获取到对应的数据，检索速度非常快。但是 Mysql 并没有采取哈希作为其底层算法，这是为什么呢？因为考虑到数据检索有一个常用手段就是范围查找，比如以下这个 SQL 语句：```select \* from user where id \>3;```针对以上这个语句，我们希望做的是找出 id>3 的数据，这是很典型的范围查找。如果使用哈希算法实现的索引，范围查找怎么做呢？一个简单的思路就是一次把所有数据找出来加载到内存，然后再在内存里筛选筛选目标范围内的数据。但是这个范围查找的方法也太笨重了，没有一点效率而言。
      - 所以，使用哈希算法实现的索引虽然可以做到快速检索数据，但是没办法做数据高效范围查找，因此哈希索引是不适合作为 Mysql 的底层索引的数据结构。
  - AVL和红黑树，在大量数据的情况下，IO操作还是太多;
  - B树每个节点内存储的是数据，因此每个节点存储的分支太少;
  - B+节点存储的是索引+指针(引用指向下一个节点)，可以存储大量索引，同时最终数据存储在叶子节点，并且有引用横向链接，可以在2-3次的IO操作内完成千万级别的表操作;
  - 建议索引是是自增长数字，这样适合范围查找.


## log

- log 
  - redo log
    - what:
      - **物理格式**日志，记录的是物理数据页面的修改的信息，其redo log是顺序写入redo log file的物理文件中去的。
    - why:
      - 确保事务的持久性。
      - 防止在发生故障的时间点，尚有脏页未写入磁盘，在重启mysql服务的时候，根据redo log进行重做，从而达到事务的持久性这一特性。
    - how:
      - 什么时候产生：
        - 事务开始之后就产生```redo log```，```redo log```的落盘并不是当事务提交时才写入的，而是在事务的执行过程中，便开始写入```redo log```文件中。
          - 重做日志有一个缓存区Innodb_log_buffer，Innodb_log_buffer的默认大小为8M(这里设置的16M),Innodb存储引擎先将重做日志写入innodb_log_buffer中。
          - 然后会通过以下三种方式将innodb日志缓冲区的日志刷新到磁盘
            - 1. Master Thread 每秒一次执行刷新Innodb_log_buffer到重做日志文件。
            - 2. 每个事务提交时会将重做日志刷新到重做日志文件。
            - 3. 当重做日志缓存可用空间 少于一半时，重做日志缓存被刷新到重做日志文件
      - 什么时候释放：
        - 当对应事务的脏页写入到磁盘之后，redo log的使命也就完成了，重做日志占用的空间就可以重用（被覆盖）。
      - 对应的物理文件：
        - 默认情况下，对应的物理文件位于数据库的data目录下的```ib_logfile1&ib_logfile2```
          - innodb_log_group_home_dir 指定日志文件组所在的路径，默认./ ，表示在数据库的数据目录下。
          - innodb_log_files_in_group 指定重做日志文件组中文件的数量，默认2
        - 关于文件的大小和数量，由一下两个参数配置
          - innodb_log_file_size 重做日志文件的大小。
          - innodb_mirrored_log_groups 指定了日志镜像文件组的数量，默认1

  - undo log回滚日志
    - what:
      - **逻辑格式**的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，**而不是从物理页面上操作实现**的，这一点是不同于redo log的。
    - why:
      - 保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读
    - how:
      - 什么时候产生：
        - 事务开始之前，将当前是的版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性
      - 什么时候释放：
        - 当事务提交之后，undo log并不能立马被删除，
        - 而是放入待清理的链表，由purge线程判断是否由其他事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间。
      - 对应的物理文件：
        - MySQL5.6之前，undo表空间位于共享表空间的回滚段中，共享表空间的默认的名称是ibdata，位于数据文件目录中。
        - MySQL5.6之后，undo表空间可以配置成独立的文件，但是提前需要在配置文件中配置，完成数据库初始化后生效且不可改变undo log文件的个数如果初始化数据库之前没有进行相关配置，那么就无法配置成独立的表空间了。
        - 关于MySQL5.7之后的独立undo 表空间配置参数如下
          - innodb_undo_directory = /data/undospace/ --undo独立表空间的存放目录
          - innodb_undo_logs = 128 --回滚段为128KB
          - innodb_undo_tablespaces = 4 --指定有4个undo log文件
        - 如果undo使用的共享表空间，这个共享表空间中又不仅仅是存储了undo的信息，共享表空间的默认为与MySQL的数据目录下面，其属性由参数innodb_data_file_path配置。
      - 其他：
        - undo是在事务开始之前保存的被修改数据的一个版本，产生undo日志的时候，同样会伴随类似于保护事务持久化机制的redolog的产生。
        - 默认情况下undo文件是保持在共享表空间的，也即ibdatafile文件中，当数据库中发生一些大的事务性操作的时候，要生成大量的undo信息，全部保存在共享表空间中的。
        - 因此共享表空间可能会变的很大，默认情况下，也就是undo 日志使用共享表空间的时候，被“撑大”的共享表空间是不会也不能自动收缩的.mysql5.7之后的“独立undo 表空间”的配置就显得很有必要了。


  - binlog二进制日志
    - what:
      - 逻辑格式的日志，可以简单认为就是执行过的事务中的sql语句。但又不完全是sql语句这么简单，而是包括了执行的sql语句（增删改）反向的信息，也就意味着delete对应着delete本身和其反向的insert；update对应着update执行前后的版本的信息；insert对应着delete和insert本身的信息。
      - 在使用mysqlbinlog解析binlog之后一些都会真相大白。
    - why:
      - 1. 用于复制，在主从复制中，从库利用主库上的binlog进行重播，实现主从同步。
      - 2. 用于数据库的基于时间点的还原。
    - how:
      - 什么时候产生：
        - 事务提交的时候，一次性将事务中的sql语句（一个事物可能对应多个sql语句）按照一定的格式记录到binlog中。
        - 这里与redo log很明显的差异就是redo log并不一定是在事务提交的时候刷新到磁盘，redo log是在事务开始之后就开始逐步写入磁盘。
        - 因此对于事务的提交，即便是较大的事务，提交（commit）都是很快的，但是在开启了bin_log的情况下，对于较大事务的提交，可能会变得比较慢一些。
          - 这是因为binlog是在事务提交的时候一次性写入的造成的，这些可以通过测试验证。
      - 什么时候释放：
        - binlog的默认是保持时间由参数expire_logs_days配置，也就是说对于非活动的日志文件，在生成时间超过expire_logs_days配置的天数之后，会被自动删除。
      - 对应的物理文件：
        - 配置文件的路径为log_bin_basename，binlog日志文件按照指定大小，当日志文件达到指定的最大的大小之后，进行滚动更新，生成新的日志文件。
        - 对于每个binlog日志文件，通过一个统一的index文件来组织。
      - 二进制日志的作用之一是还原数据库的，这与redo log很类似，很多人混淆过，但是两者有本质的不同
        - 1. 作用不同：redo log是保证事务的持久性的，是事务层面的，binlog作为还原的功能，是数据库层面的（当然也可以精确到事务层面的），虽然都有还原的意思，但是其保护数据的层次是不一样的。
        - 2. 内容不同：redo log是物理日志，是数据页面的修改之后的物理记录，binlog是逻辑日志，可以简单认为记录的就是sql语句
        - 3. 另外，两者日志产生的时间，可以释放的时间，在可释放的情况下清理机制，都是完全不同的。
        - 4. 恢复数据时候的效率，基于物理日志的redo log恢复数据的效率要高于语句逻辑日志的binlog

- MySQL通过**两阶段提交**过程来完成事务的一致性的，也即redo log和binlog的一致性的，理论上是先写redo log，再写binlog，两个日志都提交成功（刷入磁盘），事务才算真正的完成。



## mvcc

mvcc(Multi-Version Concurrency Control) --- Lock-Based Concurrency Control

- what:

\ | 脏读 | 不可重复读 | 幻读
--|----|-------|---
READ UNCOMMITTED | 1 | 1 | 1
READ COMMITTED | 0 | 1 | 1
REPEATABLE READ | 0 | 0 | 1
SERIALIZABLE | 0 | 0 | 0
  - MySQL的实现：MySQL默认采用RR隔离级别，SQL标准是要求RR解决不可重复读的问题，但是因为MySQL采用了gap lock，所以实际上MySQL的RR隔离级别也解决了幻读的问题。
    - 1. 脏读的情况：对于两个事务T1与T2，T1读取了已经被T2更新但是还没有提交的字段之后，若此时T2回滚，T1读取的内容就是临时并且无效的
    - 2. 不可重复读: 对于两个事务T1和T2，T1读取了一个字段，然后T2更新了该字段并提交之后，T1再次提取同一个字段，值便不相等了。
    - 3. 幻读: 对于两个事务T1、T2，T1从表中读取数据，然后T2进行了INSERT操作并提交，当T1'再次读取的时候，结果不一致的情况发生。

- why:



- how:
  - **隔离级别有关的加锁操作**
    - 在Serializable隔离级别下，所有的操作都会加锁
      - 在Serializable隔离级别下，无论是查询语句也会加锁，也就是说快照读不存在了，MVCC降级为Lock-Based CC。
    - 在RC，RR隔离级别下, 对于查询语句（比如```select * from T1 where id = 10```）来说，都是快照读，不加锁。
      - **当前读**
        - next-key锁(行记录锁+Gap间隙锁)
          - 记录锁（Record Lock）：
            - 记录锁锁定索引中的一条记录。
          - 间隙锁（Gap Lock）：
            - 间隙锁要么锁住索引记录之间的值，要么锁住第一个索引记录前面的值或最后一个索引记录后面的值。
          - Next-Key Lock：
            - 索引记录上的记录锁和在记录之前的间隙锁的组合。
            - next-key锁(行记录锁+Gap间隙锁)
        - 触发动作
          - 对主键或唯一索引，如果当前读时，where条件全部精确命中(=或者in)，这种场景本身就不会出现幻读，所以只会加行记录锁。
          - 非唯一索引列，如果where条件部分命中(>、<、like等)或者全未命中，则会加附近Gap间隙锁。例如，某表数据如下，非唯一索引2,6,9,9,11,15。如下语句要操作非唯一索引列9的数据，gap锁将会锁定的列是(6,11]，该区间内无法插入数据。
          - 没有索引的列，当前读操作时，会加全表gap锁，生产环境要注意。
      - **快照读**
        - 单纯的select操作，不包括上述 select ... lock in share mode, select ... for update。
        - Read Committed隔离级别：每次select都生成一个快照读。--->TODO 这里应该是开始确定一个快照ID吧
        - Read Repeatable隔离级别：开启事务后第一个select语句才是快照读的地方，而不是一开启事务就快照读。--->TODO 这里应该是开始确定一个快照ID吧
          - 下图右侧绿色的是数据：一行数据记录，```主键ID是10，name='Jack'，age=10,``` 被update更新set为```name= 'Tom'，age=23。```
            - 事务会先使用“排他锁”锁定改行，将该行当前的值复制到undo log中，然后再真正地修改当前行的值，最后填写事务的DB_TRX_ID，使用回滚指针DB_ROLL_PTR指向undo log中修改前的行DB_ROW_ID。
            - ![20210219150303](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20210219150303.png)
              - DB_TRX_ID: 6字节DB_TRX_ID字段，表示最后更新的事务id(update,delete,insert)。此外，删除在内部被视为更新，其中行中的特殊位被设置为将其标记为已软删除。
              - DB_ROLL_PTR: 7字节回滚指针，指向前一个版本的undolog记录，组成undo链表。如果更新了行，则撤消日志记录包含在更新行之前重建行内容所需的信息。
              - DB_ROW_ID: 6字节的DB_ROW_ID字段，包含一个随着新行插入而单调递增的行ID, 当由innodb自动产生聚集索引时，聚集索引会包括这个行ID的值，否则这个行ID不会出现在任何索引中。如果表中没有主键或合适的唯一索引, 也就是无法生成聚簇索引的时候, InnoDB会帮我们自动生成聚集索引, 聚簇索引会使用DB_ROW_ID的值来作为主键; 如果表中有主键或者合适的唯一索引, 那么聚簇索引中也就不会包含 DB_ROW_ID了 。  
            - ```insert undo log```只在事务回滚时需要,事务提交就可以删掉了。```update undo log```包括```update``` 和 ```delete```, 回滚和快照读都需要。














## FQA

- [如何锁住一个范围]()


1. 数据库事务ACID特性
数据库事务的4个特性：

原子性(Atomic): 事务中的多个操作，不可分割，要么都成功，要么都失败； All or Nothing.

一致性(Consistency): 事务操作之后, 数据库所处的状态和业务规则是一致的; 比如a,b账户相互转账之后，总金额不变；

隔离性(Isolation): 多个事务之间就像是串行执行一样，不相互影响;

持久性(Durability): 事务提交后被持久化到永久存储.


https://www.cnblogs.com/kisun168/p/11320549.html
https://zhuanlan.zhihu.com/p/161933980



5. MySQL 中RC和RR隔离级别的区别
MySQL数据库中默认隔离级别为RR，但是实际情况是使用RC 和 RR隔离级别的都不少。好像淘宝、网易都是使用的 RC 隔离级别。那么在MySQL中 RC 和 RR有什么区别呢？我们该如何选择呢？为什么MySQL将RR作为默认的隔离级别呢？


5.1 RC 与 RR 在锁方面的区别
1> 显然 RR 支持 gap lock(next-key lock)，而RC则没有gap lock。因为MySQL的RR需要gap lock来解决幻读问题。而RC隔离级别则是允许存在不可重复读和幻读的。所以RC的并发一般要好于RR；

2> RC 隔离级别，通过 where 条件过滤之后，不符合条件的记录上的行锁，会释放掉(虽然这里破坏了“两阶段加锁原则”)；但是RR隔离级别，即使不符合where条件的记录，也不会是否行锁和gap lock；所以从锁方面来看，RC的并发应该要好于RR；另外 insert into t select ... from s where 语句在s表上的锁也是不一样的。



