# Java进阶——分布式架构

## 消息队列

### 核心问题

- #### 为什么要使用消息队列？

  - 解耦

    通过接口调用是强耦合性的，为不同的消息发送要调用不同的接口。

    如果使用MQ，消息的生产者只需要考虑把消息送进队列，消费者只需要从队列消费消息。

    通过MQ，Pub/Sub发布订阅消息模型，实现消息生产者系统和消费者系统解耦。

  - 异步

    消息生产者不需要阻塞等待消费者，缩短响应时间。（有可能造成消息被积压）

  - 削峰

    高峰期容忍消息挤压，平峰期慢慢消费消息。

    避免下层系统被打死。

- #### 消息队列的优缺点？

  - 系统可用性降低

    MQ存在单点问题，加入MQ挂了会导致整个系统故障。

  - 系统复杂度提高

    如何保证以下一堆问题。

  - 一致性问题

    比如BCD三个系统消费，BC写库成功，C失败，导致数据不一致

- #### Kafka, ActiveMQ, RabbitMQ, RocketMQ 都有什么区别？、

  | 特性                      | ActiveMQ               | RabbitMQ                                         | RocketMQ                                                     | Kafka                                                        |
  | ------------------------- | ---------------------- | ------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | 单机吞吐量                | 万级                   | 万级                                             | 10万级                                                       | 10万级，一般配合大数据类的系统来进行实时数据计算、日志采集等场景。 |
  | topic数量对于吞吐量的影响 |                        |                                                  | topic可以达到几百/几千的级别，吞吐量**小幅下降**。RocketMQ的一大优势，在同等机器下支撑大量topic。 | topic从几十到几百个的时候，吞吐量会**大幅度下降**。在同等机器下，kafka需要保证topic数量不要过多，如果要支撑大量topic，需要增加机器资源。 |
  | 时效性                    | 毫秒级                 | 微秒级                                           | 毫秒级                                                       | 毫秒级                                                       |
  | 可用性                    | 高，主从架构实现高可用 | 高，主从架构实现高可用                           | 非常高，分布式架构                                           | 非常高，分布式，一个数据多个副本，少量机器宕机不会丢失数据   |
  | 消息可靠性                | 有较低概率丢失         | 基本不丢失                                       | 经过参数优化可以做到0丢失                                    | 经过参数优化可以做到0丢失                                    |
  | 功能支持                  | MQ领域的功能极其完备   | 基于erlang开发，并发能力很强，性能极好，延时很低 | MQ功能较为完善，分布式拓展性高                               | 功能比较简单，主要支撑简单MQ功能，在大数据实时计算以及日志采集被大规模使用 |

- #### 如何保证消息队列的高可用？

  ##### RabbitMQ的高可用

  RabbitMQ是基于主从做高可用的

  - 单机模式

  - 普通集群模式

    在这个模式中，部署的队列只会被放在一个实例中，其他实例只有队列的元数据，假如从另一个实例拉取数据，这个实例会去从这个队列所在的实例拉取数据。

    从随机实例拉取数据：数据拉取开销

    从主实例拉取数据：单实例性能瓶颈

  - 镜像集群模式（高可用）

    在这个模式中，部署的队列的元数据和实际数据会被同步到所有节点。

    好处：高可用

    坏处：性能开销极大，没有办法线性拓展高负载的队列。

  ##### Kafka的高可用性

  Kafka的架构：由多个broker组成，每个broker是一个节点；创建一个topic，这个topic可以划分为多个partition，每个partition可以存在于不同的broker上。

  RabbitMQ之流并不算是分布式消息队列，一个队列始终只能存在一个节点上，要么就是完整镜像到另一个节点。

  Kafka 0.8之前也是没有高可用机制的，一个broker宕机会导致所存的partition不可读写。

  Kafka 0.8之后提供了一个replica机制。每个partition的数据都会同步到其他机器上，形成自己的多个副本。所有replica会选出一个leader，其他的replica是follower，消费者之会跟leader打交道，读直接读leader的数据，写会被leader同步到follower上。

  只有leader可读写？

  如果任何一个replica都可以读写，那就要考虑数据一致性问题，导致系统复杂度过高，容错性降低。

  **高可用：**某个broker宕机了存在备份冗余，leader down了重新选举leader，follower down了就down了。

  **写数据：**生产者只写给leader，leader先落地本地磁盘。接着其他follower主动来pull数据，一旦follower同步完成，就会发送ack给leader。leader收到所有ack之后，返回操作成功给生产者。

  **读数据：**消费者只从leader读取，但是只有这个消息被所有follower同步成功返回ack之后，才能被消费者读取到。

- #### 如何保证消息不被重复消费？如何保证消费的时候是幂等的？

  ##### 保证消息不被重复消费

  MQ无法保证消息不被重复消费，只能由消费者来保证

  ##### 保证消费的时候是幂等的

  1. 比如你拿个数据要写库，你先根据主键查一下，如果这数据都有了，你就别插入了， update 一下好吧。 
  2. 比如你是写 Redis，那没问题了，反正每次都是 set，天然幂等性。
  3.  比如你不是上面两个场景，那做的稍微复杂一点，你需要让生产者发送每条数据的时候， 里面加一个全局唯一的 id，类似订单 id 之类的东西，然后你这里消费到了之后，先根据这个 id 去比如 Redis 里查一下，之前消费过吗？如果没有消费过，你就处理，然后这个 id 写 Redis。如果消费过了，那你就别处理了，保证别重复处理相同的消息即可。
  4.  比如基于数据库的唯一键来保证重复数据不会重复插入多条。因为有唯一键约束了，重复 数据插入只会报错，不会导致数据库中出现脏数据。

  如何保证 MQ 的消费是幂等性的，需要结合具体的业务来看。

- #### 如何保证消息的可靠性传输？如果消息丢失了怎么办？

  ##### RabbitMQ

  - 生产者弄丢了数据
    - 事务机制（同步，性能低）
    - confirm机制 （异步）
  - MQ弄丢了数据
    - 数据落盘，开启持久化
  - 消费者弄丢了数据
    - 关闭自动ACK，处理完再ACK

  ##### Kafka

  - 生产者弄丢数据
    - 只要设置了`acks=all`就不可能丢失，必须得不断重试写入每个replica
  - Kafka弄丢数据
    - 给topic设置`replication.factor`参数，必须大于1，每个partition至少有两个副本
    - 给Kafka服务器端设置`min.insync.replicas`参数，必须大于1，每个leader至少感知到有一个follower和自己保持联系
    - 给producer端设置`acks=all`，每条数据必须是写入所有replica后才能认为写入成功
    - 给producer端设置`retries=MAX`，一旦写入失败，就无限重试
  - 消费者弄丢数据
    - 关闭自动提交offset

- #### 如何保证消息的顺序性？

  ##### RabbitMQ

  - 拆分队列，每一个队列对应一个消费者，有序消息进入同一个队列

  ##### Kafka

  - 到同一个内存队列

- #### 如何解决消息队列的延时以及过期失效问题？消息队列满了以后该怎么处理？

  - 大量消息挤压
    - 原因预测：消费者挂了，或无法消费
    - 紧急扩容：
      1. 新建一个topic，partition是原来的十倍
      2. 写一个临时分发程序，把消息均匀轮询分发到十倍的partition
      3. 征用10倍的机器资源来部署comsumer

- #### 如果让你写一个消息队列，该如何进行架构设计啊？

  - 可伸缩性
  - 持久化
  - 高可用
  - 数据0丢失

## 搜索引擎

### ES分布式架构原理

#### Index -> Type -> mapping -> document -> field

#### Replica Shard

### ES写入数据的工作原理

#### 程序级别原理

write request -> coordinate node --router--> node(with primary shard) --synchronize--> replica node -> response

read request -> any node -> hashed doc id --router(load balanced with round robin)--> shard --response(to coodinate node)--> response

#### 操作系统级别原理

write request-> memory buffer --(full or a period of time)--> os cache (can be searched since here) -> segment file

translog

### 大数据场景下（数十亿级别）提升查询效率

1. 万变不离其中
   - 无脑加内存，把更多的内存留给filesystem cache
   - 把索引留在内存里，查到对应记录之后根据doc id去hbase/mysql读取完整数据
2. 数据预热
   - 写个程序把热点数据都刷到filesystem cache里
3. 冷热分离
4. 别用关联查询
5. 分页性能优化

## 缓存

### 项目中缓存是如何使用的

#### 高性能

因为redis基于内存存储，所以可以提高读写速度。

#### 高并发

同上。

### 使用了缓存的不良后果

- 缓存数据库双写不一致
- 缓存雪崩（缓存系统突然宕机，高峰期请求全落数据库，把数据库打死）
- 缓存穿透（比如被恶意攻击，查的全是不存在的key，请求也会全部落到数据库）
- 缓存击穿（缓存失效的一瞬间，热点数据的全部请求都落到数据库，数据库被打死）

### Redis 和 Memcached 有什么区别？Redis 的线程模型是什么？为什么 Redis 单线程却能支撑高并发？

- Key：Redis是个单线程工作模型
- Redis和Memcached的区别：
  - Redis支持复杂的数据结构
  - Redis原生支持集群模式（Redis Cluster）
  - Redis单核，存储小数据<100k的性能更高；Memcached多核，在存储大数据时性能更高。
- Redis的线程模型
  - Redis内部使用的文件事件处理器叫`file event handler`，这个文件处理器是单线程的，它采用了IO Multiplexing机制同时监听多个socket，将产生事件的socket压入内存队列，事件分派器根据文件事件类型来选择对应的事件处理器。
  - 文件事件处理器的结构：
    - 多个socket
    - IO多路复用程序
    - 文件事件分派器
    - 事件处理器（连接应答处理器，命令请求处理器，命令回复处理器）
- 为啥Redis单线程支撑高并发
  - 纯内存操作（暴力，直接提升读写速度）
  - 核心基于Non-blocking IO Multiplexing（底层上采用支持并发的IO模型）
  - c语言实现（程序效率高）
  - 单线程反而避免了多线程的频繁上下文切换，预防了多线程可能产生的竞争问题（？这个就有待考证了）
  - 注：redis 6.0开始引入多线程，socket的IO操作占用了大量的cpu时间片，不得不把socket IO操作拿出来做成多线程，但是执行命令还是单线程（避免系统复杂度过高）

### Redis的数据类型以及适用场景

- Strings

  最简单的类型，KV缓存

  ```bash
  set college zjut
  get college
  ```

- Hashes

  存储的是一个map，或者叫一个对象（没有嵌套其他对象）

  ```bash
  hset person name salvation
  hset person age 22
  hset person id 1
  hget person name
  ```

- Lists

  有序列表

  ```bash
  lrange mylist 0 -1 #lrange表示范围读取，下标类型于python
  lpush mylist 1
  lpush mylist 2 3 4
  rpop mylist #1
  ```

- Sets

  无序集合，实现在多台机器上一个自动去重的操作，求交集并集

  ```bash
  sadd myset 1 #add
  smembers myset #select all
  sismember myset 3 #if is a member of
  srem myset 1 #remove
  scard myset #count
  spop myset #remove one randomly
  smove youset myset 2 #move elements from one set to anonther
  sinter youset myset #figure out the same part
  sunion youset myset #combine
  sdiff youset myset #elements in youset but not in myset
  ```

- Sorted Sets
  有序集合，实现排序

  ```bash
  zadd board zhangsan
  zadd board lisi
  zadd board wangwu
  zadd board zhaoliu
  # 获取排名前三的用户（默认是升序，所以需要 rev 改为降序）
  zrevrange board 0 3
  # 获取某用户的排名
  zrank board zhaoliu
  ```

### Redis的过期策略、内存策略、手撕LRU

#### Redis过期策略

**定期删除+惰性删除**

- 定期删除

  默认每隔100ms就随机抽查设置了过期时间的key，过期了就删除。不能全查，开销太大。

- 惰性删除

  获取key的时候，若已经过期，就删除，不返回

如果在以上机制的基础上，redis还是快耗尽了内存，就要走内存淘汰机制

#### 内存淘汰机制

- noeviction：写不下就不写了
- allkeys-lru：在全部键空间中，删除最近最少使用的key
- allkeys-random：在全部键空间中，随机删
- volatile-lru：在设置了过期时间的键空间中，删除最近最少使用的key
- volatile-random：在设置了过期时间的键空间中，随机删
- volatile-ttl：在设置了过期时间的键空间中，优先删最早过期的

#### 手撕LRU

两种基本思路：在读取的时候要改变被读取项的位置或者标记

### 保证Redis的高并发和高可用、主从复制原理、哨兵机制

#### 主从架构、读写分离——保证读高并发

读请求走从节点，写请求走主节点同步给从节点。轻松实现水平扩容，增加从节点的数量就可以支撑读高并发。

#### Redis replication的核心机制

- 主节点采用异步方式复制数据到从节点
- 从节点在做复制的时候不会阻塞自己的查询操作，会暂时使用旧的数据继续提供服务。待数据复制完成之后会删除旧的数据集，加载新的数据集，此时会暂时停止服务。

注意：

- 必须开启master node的持久化机制，把数据落盘，否则master节点宕机再重启会丢失所有数据，然后slave节点也会跟着清空
- 还要做其他备份方案，确保master node重启之后有数据。即使有哨兵机制——主从转换作为高可用方案，也有可能会丢失所有数据：比如master node重启太快，哨兵还没检测到master failure就重启了，如果没有其他备份方案一样会丢失所有数据。

Replication的过程：

1. 启动slave node的时候，它会发送一个psync命令给master node。
2. 如果是初次连接，会触发一次full resynchronization全量复制。master会启动一个线程来生成一份rdb快照文件，（太长了待续）

#### Redis的高可用机制

- Slave node挂了，不影响高可用，其他node照样能读
- Master node挂了，触发主备切换机制，故障转移，快速恢复可用性

### Redis的持久化方式、优缺点、底层实现

持久化主要是为了做灾难恢复、数据恢复

#### 持久化方式

- RDB：RDB持久化机制，对redis中的数据执行周期性的持久化（周期性生成rdb快照）
- AOF：AOF机制对写入命令做日志，以append-only的模式写入到一个日志文件中，可以通过回放AOF日志来构建数据集

持久化之后要备份到别的地方去，比如企业冷储存，阿里云等云服务。

同时开启时redis会选择用AOF重构数据集，因为完整度更高。

RDB优缺点：

- 生成每一时刻的数据快照，适合做冷备份。
- 对读写服务影响小，只要fork一个子进程来执行，从内存读取，写到磁盘。
- 相比较于AOF来说，RDB来恢复redis会更快。
- 丢失数据多，一般来说RDB备份5分钟执行一次，一旦宕机会丢失5分钟数据。
- RDB 每次在fork子进程来执行 RDB 快照数据文件生成的时候，如果数据文件特别大，可能会导致对客户端提供的服务暂停数毫秒，或者甚至数秒。（？）

AOF优缺点：

- AOF丢失数据少，一般AOF每隔一秒执行一次fsync，把数据落盘，最多丢失一秒的数据。
- 日志文件以append-only模式写入，没有磁盘寻址开销，写入性能非常高，文件不容易破损，即使破损也是尾部出问题。
- AOF日志文件过大会出现重写，但不影响客户端读写。rewrite log的时候会压缩指令，生成一份新的日志文件，替换老的。
- 可读性强，没有发生rewrite前可以手动修改指令。
- AOF日志会比RDB文件大。（包含指令和重复指令）
- AOF开启后会影响写入QPS。
- 方法复杂度高，容易出bug。（恢复不出一模一样的数据集）

### Redis哨兵集群高可用

#### Sentinel组件

- 集群监控：监控master和slave是否正常工作
- 消息通知：如果某个redis实例故障，哨兵负责发送报警给管理员
- 故障转移：判断一个master node宕机，需要majority哨兵同意
- 配置中心：如果故障转移发生，通知所有客户端新的master地址

注意：哨兵作为redis集群的高可用保障机制，本身也是分布式的，按理说哨兵集群的可用性应该大于等于redis集群的可用性，才能保障redis集群的高可用。

#### 哨兵的核心知识

- 哨兵至少需要3个实例来保证健壮性（2个实例的话down掉一个就没法选举了）
- 不保证数据0丢失，只保证高可用

#### Redis主备切换导致数据丢失问题

- 异步复制导致数据丢失

  master的数据还没复制到slave就宕机了，数据丢失

- 脑裂导致数据丢失

  master突然脱离正常网络，实际上还在运行，但是哨兵认为master宕机了，开启选举，集群里出现了两个master，这就是脑裂。

  client继续像旧的master写入数据，当旧的master作为slave跟随新的master时，这部分数据丢失。

#### 数据丢失解决

```bash
# 至少有一个slave可写
min-slaves-to-write 1
# 当master复制数据到slave的延迟大于10s，master就停止接收请求，最多丢失10s数据
min-slaves-max-lag 10
```

#### sdown和odown机制

- sdown是主观宕机，一个哨兵自己觉得一个master宕机了，即ping master超过 `is-master-down-after-milliseconds`。
- odown是客观宕机，quorum数量的哨兵都觉得一个master宕机了，即收到quorum数量其他哨兵发出的master sdown的消息。

#### 哨兵集群的自动发现机制

通过Redis的`pub/sub`系统实现，每个哨兵会往`__sentinel__:hello`这个channel里发消息，内容是自己的host、ip和runid还有对master的监控配置。其他所有哨兵都能消费到这个消息。

#### slave配置的自动修正

- 如果slave要成为潜在的master候选人，哨兵会确保slave复制现有的master的数据
- 如果slave连接到了错误的master，会被哨兵纠正

#### slave->master选举算法

如果master被认为odown，而且majority数量的哨兵都允许主备切换，那某个哨兵就会执行主备切换操作

此时首先要选举一个 slave 来，会考虑 slave 的一些信息：

- 跟 master 断开连接的时长
- slave 优先级
- 复制 offset
- run id

如果一个 slave 跟 master 断开连接的时间已经超过了 down-after-milliseconds 的 10 倍，外加 master 宕机的时长，那么 slave 就被认为不适合选举为 master。
接下来会对 slave 进行排序：

- 按照 slave 优先级进行排序，slave priority 越低，优先级就越高。
- 如果 slave priority 相同，那么看 replica offset，哪个 slave 复制了越多的数据，offset 越靠后，优先级就越高。
- 如果上面两个条件都相同，那么选择一个 run id 比较小的那个 slave。

#### quorum和majority

#### configuration epoch

#### configuration传播

### Redis cluster

主要是针对**海量数据+高并发+高可用**的场景。

结构：n个master，每个master上跟随m个slave。

主要优点：支持横向拓展，实现更大容量的缓存。

#### 一致性Hash算法

缓存倾斜——虚拟节点解决

## 分库分表

- 分表：单表数据量太大，影响sql的执行性能
- 分库：减少单库的并发量

| #            | 分库分表前                  | 分库分表后                                     |
| ------------ | --------------------------- | ---------------------------------------------- |
| 并发支撑情况 | MySQL单机部署，扛不住高并发 | MySQL从单机到多机，能承受的并发增加了多倍      |
| 磁盘使用情况 | MySQL单机磁盘容量几乎撑满   | 拆分为多个库，数据库服务器的磁盘使用率大大降低 |
| SQL执行性能  | 单表数据量太大，SQL跑不动   | 单表数据量减少，SQL执行效率明显提升            |

### 分库分表中间件

#### Cobar

阿里 b2b 团队开发和开源的，属于 proxy 层方案，就是介于应用服务器和数据库服务器之间。
应用程序通过 JDBC 驱动访问 Cobar 集群，Cobar 根据 SQL 和分库规则对 SQL 做分解，然后分
发到 MySQL 集群不同的数据库实例上执行。早些年还可以用，但是最近几年都没更新了，基本
没啥人用，差不多算是被抛弃的状态吧。而且不支持读写分离、存储过程、跨库 join 和分页等
操作。

#### TDDL

淘宝团队开发的，属于 client 层方案。支持基本的 crud 语法和读写分离，但不支持 join、多表
查询等语法。目前使用的也不多，因为还依赖淘宝的 diamond 配置管理系统。

#### Atlas

360 开源的，属于 proxy 层方案，以前是有一些公司在用的，但是确实有一个很大的问题就是
社区最新的维护都在 5 年前了。所以，现在用的公司基本也很少了。

#### Sharding-jdbc

当当开源的，属于 client 层方案，是 ShardingSphere 的 client 层方案， ShardingSphere
还提供 proxy 层的方案 Sharding-Proxy。确实之前用的还比较多一些，因为 SQL 语法支持也比
较多，没有太多限制，而且截至 2019.4，已经推出到了 4.0.0-RC1 版本，支持分库分表、读
写分离、分布式 id 生成、柔性事务（最大努力送达型事务、TCC 事务）。而且确实之前使用的
公司会比较多一些（这个在官网有登记使用的公司，可以看到从 2017 年一直到现在，是有不少
公司在用的），目前社区也还一直在开发和维护，还算是比较活跃，个人认为算是一个现在也
可以选择的方案。

#### Mycat

基于 Cobar 改造的，属于 proxy 层方案，支持的功能非常完善，而且目前应该是非常火的而且
不断流行的数据库中间件，社区很活跃，也有一些公司开始在用了。但是确实相比于 Sharding
jdbc 来说，年轻一些，经历的锤炼少一些。

综上：

- client层方案

  优点：

  - 不用部署
  - 运维成本低
  - 不需要代理层二次转发请求，性能高

  缺点：

  - 各个系统都需要耦合中间件的依赖

- proxy层方案

  优点：

  - 耦合度低，对于各个系统都是透明的

  缺点：

  - 需要部署维护

### 水平拆分&垂直拆分

#### 水平拆分

一个库的一个表，分到多个库一模一样的多个表，扛高并发、扩容

#### 垂直拆分

多个字段的表拆分成少数字段的多个表，访问频率高的字段放一个表，访问频率低的字段放一个表，缓存高频数据可以缓存更多行。

### 分库分表的方式

- 按照range来分，比如按照时间范围，一星期一个库，但是容易产生热点问题，大部分的流量都到某个范围的数据里。
- 按照某个字段hash来分，常用

### 分库分表的迁移方案

- **停机迁移**

  晚上十二点停机，把老数据导入到新的分好的库里。（问题在于大数据量的情况下六个小时都不一定导的完）

- **双写迁移**

  在代码里修改，同时写（增删改）新库和老库，部署完之后再用迁移工具，把老库中早于上线时间点的数据都写到新库里去。写完一轮之后还会存在数据不一致的情况（某一行数据导出到新库之后可能因为被操作发生改变），要一直循环这个迁移操作，直到老库新库数据一致。然后改代码，只写入新库，重新上线，迁移完成。

### 分库分表后ID如何处理

- 数据库自增id

  用一个单库，每次要生成id的时候插入一条垃圾数据，生成一个Id，用这个Id去写分库分表的库。

  缺点：高并发场景下这个单库存在单点问题。

- 设置数据库sequence或者表自增步长

  1: 1->5->9->13

  2: 2->6->10->14

  3: 3->7->11->15

  4: 4->8->12->16

  缺点：扩容加库操作十分繁琐。

- UUID

  UUID太长，占用空间大，作为主键性能太差；UUID不具备有序性，会导致B+树索引在写的时候有很多随机写操作。

- 时间戳

  高并发有重复，基本不用考虑。

- 雪花算法

  - 1bit：不用
  - 41bit：毫秒时间戳
  - 10bit：工作机器id
  - 12bit：同一毫秒内顺序生成

### MySQL读写分离