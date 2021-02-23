
GO相关：
- 1. go里 new和make的区别 
  - 输入参数，输出参数
- 2. 值类型和引用类型的区别
  - 最大区别是深拷贝还是浅拷贝。

3. GMP模型 --- GO的调度 --- GO的调度器

- 4. GO中同步的机制有哪些
  - go自带的：chan, sync.WaitGroup, sync.Cond.
  - 锁：sync.Lock, sync.RWLock, 原子锁.

- 5. GO中垃圾回收
  - gc
  - 怎么从root对象开始？ （白色--》灰色--》黑色）
- 6. 线程和进程的区别，对线程的理解，（堆栈相关）
  - 共用的堆，各自独立的栈。
  - 拥有各自的信号
  - 线程是系统最小的逻辑单元，进程是系统的最小的执行单元
- 8. 逃逸分析
  - 可以参考golang.md的逃逸分析

- 9. pprof相关调试
  - heap, groutine, memory, cpu?
  - 收集完数据，进入里面去以后，类似gdb的list

- 10. go chan相关 （close之后对chan的各种操作会导致什么）
  - 空读写阻塞，关闭恐慌
  - 关闭读为0，关闭写恐慌


其他方面：
1. 数据库索引的存储方式
  - 非叶子节点无数据，数据存储在叶子节点.
  - 叶子节点数据通过指针相连,便于范围查找。
2. redis几种数据结构，Redis的存储方式，redis Hash的具体实现
  - string, set, zset, hash, list
  - redisObject[type, encoding, ptr]
    - ziplist and hashtable(dict)
      - key/value的~~大小和长度~~(key/value字符串长度都小于64Byte)还有个数有关。
      - hash
        - hash的算法
        - hash的冲突的解决
        - ```负载因子 = 哈希表已保存节点数量 / 哈希表大小``` ~~存储的key/value与hash桶比值过高~~，需要迁移。
3. 数据库设计三大范式
  - 需求>性能>表结构
    - 列不可再分
    - 第二范式（2NF）和第三范式（3NF）的概念很容易混淆，区分它们的关键点在于，2NF：非主键列是否完全依赖于主键，还是依赖于主键的一部分；3NF：非主键列是直接依赖于主键，还是直接依赖于非主键列。
      - 2NF: 首先是 1NF，另外包含两部分内容，**一是表必须有一个主键；二是没有包含在主键中的列必须完全依赖于主键，而不能只依赖于主键的一部分。**
        - 考虑一个订单明细表：```【OrderDetail】（OrderID，ProductID，UnitPrice，Discount，Quantity，ProductName）。```
          - 因为我们知道在**一个订单**中可以订购**多种产品**，所以单单一个 OrderID 是不足以成为主键的，```主键应该是（OrderID，ProductID）```。显而易见 Discount（折扣），Quantity（数量）完全依赖（取决）于主键（OderID，ProductID），~~而 UnitPrice，ProductName 只依赖于 ProductID。所以 OrderDetail 表不符合 2NF。不符合 2NF 的设计容易产生冗余数据。~~
            - 可以把【OrderDetail】表拆分为```【OrderDetail】（OrderID，ProductID，Discount，Quantity）```和```【Product】（ProductID，UnitPrice，ProductName）```来消除原订单表中```UnitPrice，ProductName```多次重复的情况。
      - 3NF: 首先是 2NF，另外非主键列必须直接依赖于主键，不能存在传递依赖。即不能存在：非主键列 A 依赖于非主键列 B，非主键列 B 依赖于主键的情况。
        - 考虑一个订单表```【Order】（OrderID，OrderDate，CustomerID，CustomerName，CustomerAddr，CustomerCity）```主键是```（OrderID）````。
          - 其中 ```OrderDate，CustomerID，CustomerName，CustomerAddr，CustomerCity```等非主键列都完全依赖于主键（OrderID），所以符合 2NF。
            - 不过问题是 ```CustomerName，CustomerAddr，CustomerCity``` 直接依赖的是 CustomerID（非主键列），而不是直接依赖于主键，它是通过传递才依赖于主键，所以不符合 3NF。
通过拆分【Order】为【Order】（OrderID，OrderDate，CustomerID）和【Customer】（CustomerID，CustomerName，CustomerAddr，CustomerCity）从而达到 3NF。
  

算法/数据结构
- 1. 排序有多少种，各自的时间/空间复杂度怎么样
  - O(n^2): 冒泡，插入，
  - O(logN): 归并，快排，
  - O(N):桶排序.

2. 实现单链表





