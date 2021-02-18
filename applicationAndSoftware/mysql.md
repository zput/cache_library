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





如何锁住一个范围。

















