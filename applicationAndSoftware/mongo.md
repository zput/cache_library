
- 1. 概述mongo
  - what:
    - ![20201214165457](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201214165457.png)
    - MongoDB中的一条记录是一个document一个对象，它是一个由字段和值对（field:value）组成的数据结构。
      - MongoDB文档类似于JSON对象，即一个文档认为就是一个对象。**字段的数据类型**是**字符型**，它的**值**除了使用基本的一些类型外，还可以包括其他文档、普通数组和文档数组。
      - 它的那个**值**(field:value)，value是什么类型，mongo不需要插入数据的时候先把表的字段，字段类型等确定，那它是什么时候确定的呢？
        - **它的值（value）保存是BSON里面定义的值**
          - what:BSON?
            - BSON（Binary Serialized Document Format）是一种类json的一种二进制形式的存储格式，简称Binary JSON。
              - [bson type](https://docs.mongodb.com/manual/reference/bson-types/)
                - ![20201214173652](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201214173652.png)
          - 但是从BSON读出来的时候可以有NumberLong这种?
            - example:![20201214184047](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201214184047.png)
            - 那它的定义就像是```32-bit integers```的另一表示方法```NumberInt("0")```
  - why:
    - 应对而MongoDB可应对“三高”需求: 高并发、高性能、高可用
      -（1）高性能：MongoDB提供高性能的数据持久性。特别是,对嵌入式数据模型的支持减少了数据库系统上的I/O活动。
      -（2）高可用性：MongoDB的复制工具称为副本集（replica set），它可提供自动故障转移和数据冗余。
      -（3）高扩展性：MongoDB提供了**水平**可扩展性作为其核心功能的一部分。分片将数据分布在一组集群的机器上。
        - how?
  - how:
    - interview:
      - 使用mgo的第三方库。
      - 使用version 4.0。
    - [crud](https://docs.mongodb.com/manual/crud/)
      - 正则的复杂条件查询```db.collection.find({field:/正则表达式/}) 或db.集合.find({字段:/正则表达式/})```
      - ![20201214190948](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201214190948.png)


- 2. [索引-Index](https://docs.mongodb.com/manual/indexes/)
  - what:
    - 单字段索引
    - 复合索引
    - 地理空间索引（Geospatial Index）、文本索引（Text Indexes）、哈希索引（Hashed Indexes）。
  - why:
  - how:
    - 索引的管理操作:
      - 索引的查看```db.collection.getIndexes()```
      - 索引的创建```db.collection.createIndex(keys, options)```
      - 索引的移除```db.collection.dropIndex(index)```
    - 索引的使用:
      - 分析查询性能（Analyze Query Performance）
        - 使用执行计划（解释计划、Explain Plan）来查看查询的情况，如查询耗费的时间、是否基于索引查询等。
        - ```db.collection.find(query,options).explain([options])```
      - 涵盖的查询
        - 当查询条件和查询的投影仅包含索引字段时，MongoDB直接从索引返回结果，而不扫描任何文档或将文档带入内存。 这些覆盖的查询可以非常有效。
        - ![20201214194756](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201214194756.png)

- 3.副本集
  - what:
    - 主从复制和副本集区别
      - 主从集群和副本集最大的区别就是**副本集**没有固定的“主节点”；整个集群会选出一个“主节点”，当其挂掉后，又在剩下的从节点中选中其他节点为“主节点”，副本集总有一个活跃点(主、primary)和一个或多个备份节点(从、secondary)。
        - 很像redis的哨兵模式(sentry)
    - 副本集有两种类型三种角色
      - 两种类型：
        - 主节点（Primary）类型：数据操作的主要连接点，可读写。
        - 次要（辅助、从）节点（Secondaries）类型：数据冗余备份节点，可以读或选举。
      - 三种角色（PSA Style (Primary - Secondary - Arbiter)）：
        - 主要成员（Primary）：主要接收所有写操作。就是主节点。
        - 副本成员（Replicate）：从主节点通过复制操作以维护相同的数据集，即备份数据，不可写操作，但可以读操作（但需要配置）。是默认的一种从节点类型。
        - 仲裁者（Arbiter）：**不保留任何数据的副本**，只具有投票选举作用。当然也可以将仲裁服务器维护为副本集的一部分，即副本成员同时也可以是仲裁者。也是一种从节点类型。
          - 您可以将额外的mongod实例添加到副本集作为仲裁者。 仲裁者不维护数据集。 仲裁者的目的是通过响应其他副本集成员的心跳和选举请求来维护副本集中的仲裁。 因为它们不存储数据集，所以仲裁器可以是提供副本集仲裁功能的好方法，其资源成本比具有数据集的全功能副本集成员更便宜。如果您的副本集具有偶数个成员，请添加仲裁者以获得主要选举中的“大多数”投票。 仲裁者不需要专用硬件。仲裁者将永远是仲裁者，而主要人员可能会退出并成为次要人员，而次要人员可能成为选举期间的主要人员。
          - 如果你的副本+主节点的个数是偶数，建议加一个仲裁者，形成奇数，容易满足大多数的投票。
          - **如果你的副本+主节点的个数是奇数，可以不加仲裁者**。
    - 主节点的选举原则
      - 触发条件
        - 1） 主节点故障
        - 2） 主节点网络不可达（默认心跳信息为10秒）
        - 3） 人工干预（rs.stepDown(600)）
      - 选举规则是根据票数来决定谁获胜：
        - 票数最高，而且获得了“大多数”成员的投票支持的节点获胜(两个都需要满足)。
          - 票数是怎么得出来的？ ===> 在获得票数的时候，优先级（priority）参数影响重大。
            - 可以通过设置优先级（priority）来设置额外票数(默认是一人一票)。优先级即权重，取值为0-1000，相当于可额外增加0-1000的票数，优先级的值越大，就越可能获得多数成员的投票（votes）数。指定较高的值可使成员更有资格成为主要成员，更低的值可使成员更不符合条件。默认情况下，优先级的值是1
          - “大多数”的定义为：假设复制集内投票成员数量为N，则大多数为 N/2 + 1。
            - 例如：3个投票成员，则大多数的值是2。当复制集内存活成员数量不足大多数时，整个复制集将无法选举出Primary，复制集将无法提供写服务，处于只读状态。
        - 若票数相同，且都获得了“大多数”成员的投票支持的，数据新的节点获胜。数据的新旧是通过操作日志oplog来对比的。

  - why:

  - how:
    - 副本集的数据读写操作
      - 主节点，写入和读取数据。
      - 从节点是没有读写权限的，可以增加读的权限。
        - 设置成slave节点:```rs.slaveOk() or rs.slaveOk(true)```
      - 仲裁者节点，不存放任何业务数据。


- 4. 分片集群-Sharded Cluster
  - what:
    - 分片（sharding）是一种跨多台机器分布数据的方法， MongoDB使用分片来支持具有非常大的数据集和高吞吐量操作的部署。
      - 就像我们上次扩展的两种方式，一种是把column分， 一种是水平分（把数据一部分分到这边，其他的分到其他地方，每个地方的column都是一样）
    - MongoDB分片群集包含以下组件：
      - 分片（存储）：每个分片包含分片数据的子集。 每个分片都可以部署为副本集。
      - mongos（路由）：mongos充当查询路由器，在客户端应用程序和分片集群之间提供接口。
      - config servers（“调度”的配置）：配置服务器存储群集的元数据和配置设置。 从MongoDB 3.4开始，必须将配置服务器部署为副本集（CSRS）。
  - why:
  - how:
    - ![20201215162409](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201215162409.png)
    - ![20201215162627](https://raw.githubusercontent.com/zput/myPicLib/master/zput.github.io/20201215162627.png)




- 5. 访问控制
  - what:
    - MongoDB使用的是基于角色的访问控制(Role-Based Access Control,RBAC)来管理用户对实例的访问。
      - 通过对用户授予一个或多个角色来控制用户访问数据库资源的权限和数据库操作的权限，在对用户分配角色之前，用户无法访问实例
  - why:
  - how:

