# kafka面试题

Kafka最初是由Linkedin公司开发的，是一个分布式的、可扩展的、容错的、支持分区的（Partition）、多副本的（replica）、基于Zookeeper框架的发布-订阅消息系统，Kafka适合离线和在线消息消费。它是分布式应用系统中的重要组件之一，也被广泛应用于大数据处理。Kafka是用Scala语言开发，它的Java版本称为Jafka。Linkedin于2010年将该系统贡献给了Apache基金会并成为顶级开源项目之一。

### 1、Kafka 的设计

Kafka 将消息以 topic 为单位进行归纳，发布消息的程序称为 **Producer**，消费消息的程序称为 **Consumer**。它是以集群的方式运行，可以由一个或多个服务组成，每个服务叫做一个 **Broker**，Producer 通过网络将消息发送到 kafka 集群，Consumer 向集群拉取消息，broker 在中间起到一个代理保存消息的中转站。

**Kafka 中重要的组件**

_1）Producer_：消息生产者，发布消息到Kafka集群。

_2）Broker_：一个 Kafka 节点就是一个 Broker，多个Broker可组成一个Kafka 集群。

_3）Topic_：消息主题，每条发布到Kafka集群的消息都会归集于此，Kafka是面向Topic 的。

_4）Partition_：Partition 是Topic在物理上的分区，一个Topic可以分为多个Partition，每个Partition是一个有序的不可变的记录序列。单一主题中的单一分区内的消息有序，但无法保证主题中所有分区的消息有序。

_5）Consumer_：从Kafka集群中消费消息的终端或服务。

_6）Consumer Group_：每个Consumer都属于一个Consumer Group，每个分区在任一时刻只能被消费者群组中的一个Consumer消费，但可以被多个Consumer Group消费。

_7）Replica_：Partition 的副本，用来保障Partition的高可用性。

_8）Controller：_ Kafka 集群中的其中一个服务器，用来进行Replica Leader election以及各种 Failover 操作。

_9）Zookeeper_：Kafka 通过Zookeeper来存储集群中的 meta 消息。

### 2、Kafka 性能高原因

* 利用了操作系统的页面缓存
* 磁盘顺序写
* 零复制技术
* pull 拉模式

### 3、Kafka 文件高效存储设计原理

1. Kafka把Topic中一个Partition大文件分成多个小文件段，通过多个小文件段，就容易根据指定规则清除过期的文件，从而减少磁盘占用。
2. kakfa为每个分区维护了一个索引，该索引将偏移量与片段文件及偏移量在文件中的位置做了映射。因此，可以快速定位到指定偏移量。
3.  根据kafka的使用模式，采用了顺序读写方式进行磁盘访问，减少了磁盘寻址的开销。

    ​

### 4、Kafka 的优缺点

**优点：**

* 高性能、高吞吐量、低延迟：Kafka 生产和消费消息的速度都达到每秒10万级。
* 高可用：所有消息持久化存储到磁盘，并支持数据备份防止数据丢失。
* 高并发：支持数千个客户端同时读写。
* 高扩展性：Kafka 集群支持热伸缩，无须停机。

**缺点：**

* 没有完整的监控工具集。

### 5、Kafka 的应用场景

1. **日志聚合**：收集各种服务的日志写入kafka的消息队列进行存储。
2. **消息系统**：广泛用于消息中间件。
3. **系统解耦**：在重要操作完成后，发送消息，由别的服务系统来完成其他操作。
4. **流量削峰**：一般用于秒杀或抢购活动中，来缓冲网站短时间内高流量带来的压力。
5. **异步处理**：通过异步处理机制，可以把一个消息放入队列中，但不立即处理它，在需要的时候再进行处理。

### 6、Kafka 中分区的概念

主题是一个逻辑上的概念，还可以细分为多个分区，一个分区只属于单个主题，很多时候也会把分区称为主题分区（Topic-Partition）。同一主题下的不同分区包含的消息是不同的，分区在存储层面可以看做一个可追加的**日志文件** ，消息在被追加到分区日志文件的时候都会分配一个特定的偏移量（offset）。offset 是消息在分区中的唯一标识，kafka 通过它来保证消息在分区内的顺序性，不过 offset 并不跨越分区，也就是说，kafka保证的是分区有序而不是主题有序。

在分区中又引入了多副本（replica）的概念，通过增加副本数量可以提高容灾能力。同一分区的不同副本中保存的是相同的消息。副本之间是一主多从的关系，其中主副本负责读写，从副本只负责消息同步。副本处于不同的 broker 中，当主副本出现异常，便会在从副本中提升一个为主副本。

### 7、Kafka 生产者中的分区规则

1. 指明Partition的情况下，直接将指明的值作为Partition值。
2. 没有指明Partition值但有 key 的情况下，将 key 的 Hash 值与 topic 的Partition数量进行取余得到Partition值。
3.  既没有Partition值又没有 key 值的情况下，第一次调用时随机生成一个整数（后面每次调用在这个整数上自增），将这个值与Topic可用的Partition总数取余得到Parittion值，也就是常说的 round-robin 算法。从kafka 2.4，在处理此种情况时，分区器使用的 round-robin算法具备了黏性，在切换到下一个分区之前，分区器会将同一个批次的消息全部写入当前分区。

    ​

### 8、Kafka 为什么要把消息分区

1. 方便在集群中扩展，每个 Partition 可用通过调整以适应它所在的机器。而一个Topic可以有多个Partition组成，因此整个集群就可以适应任意大小的数据了。
2. 可以提高并发，因为可以以Partition为单位进行读写。

### 9、Kafka 中生产者运行流程

1. 一条消息发过来首先会被封装成一个 ProducerRecord 对象。
2. 对该对象进行序列化。
3. 对消息进行分区处理，分区的时候需要获取集群的元数据，决定这个消息会被发送到哪个主题的哪个分区。
4. 分好区的消息不会直接发送到服务端，而是放入生产者的缓存区，多条消息会被封装成一个批次（Batch）。
5. Sender 线程启动以后会从缓存里面去获取可以发送的批次。
6. Sender 线程把一个一个批次发送到服务端。

### 10、Kafka 中的消息封装

在Kafka 中 Producer 可以 Batch的方式推送数据达到提高效率的作用。Kafka Producer 可以将消息在内存中累积到一定数量后作为一个 Batch 发送请求。Batch 的数量大小可以通过 Producer 的参数进行控制，可以从2个维度进行控制

* 累计的时间间隔
* 累计的数据大小

通过增加 Batch 的大小，可以减少网络请求和磁盘I/O的频次，具体参数配置需要在效率和时效性做一个权衡。

### 11、Kafka 消息的消费模式

如果采用 **Push** 模式，则Consumer难以处理不同速率的上游推送消息。

采用 Pull 模式的好处是Consumer可以自主决定是否批量的从Broker拉取数据。Pull模式有个缺点是，如果Broker没有可供消费的消息，将导致Consumer不断在循环中轮询，直到新消息到达。为了避免这点，Kafka有个参数可以让Consumer阻塞等待新消息到达。

### 12、Kafka 如何实现负载均衡与故障转移

**负载均衡**

Kakfa 的负载均衡就是每个 **Broker** 都有均等的机会为 Kafka 的客户端（生产者与消费者）提供服务，可以负载分散到所有集群中的机器上。Kafka 通过智能化的分区首领选举来实现负载均衡，提供智能化的 Leader 选举算法，可在集群的所有机器上均匀分散各个Partition的Leader，从而整体上实现负载均衡。

**故障转移**

Kafka 的故障转移是通过使用**会话机制**实现的，每台 Kafka 服务器启动后会以会话的形式把自己注册到 Zookeeper 服务器上。一旦服务器运转出现问题，就会导致与Zookeeper 的会话不能维持从而超时断连，此时Kafka集群会进行分区首领选举，将故障服务器上的所有分区首领转移到其他服务器上。

### 13、Kafka 中 Zookeeper 的作用

Kafka 是一个使用 Zookeeper 构建的分布式系统。Kafka 的各 Broker 在启动时都要在Zookeeper上注册，由Zookeeper统一协调管理。同一个Topic的消息会被分成多个分区并将其分布在多个Broker上，这些分区信息及与Broker的对应关系也是Zookeeper在维护。

### 14、Kafka 提供了哪些系统工具

* **Kafka 迁移工具**：它有助于将代理从一个版本迁移到另一个版本。
* **Mirror Maker**：Mirror Maker 工具有助于将一个 Kafka 集群的镜像提供给另一个。
* **消费者检查**：对于指定的主题集和消费者组，可显示主题、分区、所有者。

### 15、Kafka 中消费者与消费者组的关系与负载均衡实现

Consumer Group 是Kafka独有的可扩展且具有容错性的消费者机制。一个组内可以有多个Consumer，它们共享一个全局唯一的Group ID。组内的所有Consumer协调在一起来消费订阅主题（Topic）内的所有分区（Partition）。当然，每个Partition只能由同一个Consumer Group内的一个Consumer 来消费。消费组内的消费者可以使用多线程的方式实现，消费者的数量通常不超过分区的数量，且二者最好保持整数倍的关系，这样不会造成有空闲的消费者。

Consumer Group与Consumer的关系是动态维护的，当一个Consumer进程挂掉或者是卡住时，该Consumer所订阅的Partition会被重新分配到该组内的其他Consumer上，当一个Consumer加入到一个Consumer Group中时，同样会从其他的Consumer中分配出一个或者多个Partition到这个新加入的Consumer。

**负载均衡**

当启动一个Consumer时，会指定它要加入的Group，使用的配置项是：[Group.idopen in new window](http://group.id/)

为了维持Consumer与Consumer Group之间的关系，Consumer 会周期性地发送 hearbeat 到 coodinator（协调者），如果有 hearbeat 超时或未收到 hearbeat，coordinator 会认为该Consumer已经退出，那么它所订阅的Partition会分配到同一组内的其他Consumer上，这个过程称为 rebalance（再均衡）。

### 16、Kafka 中消息偏移的作用

生产过程中给分区中的消息提供一个顺序ID号，称之为偏移量，偏移量的主要作用为了唯一地区别分区中的每条消息。

### 17、Consumer 如何消费指定分区消息

可以使用 `seek(TopicPartition topicPartition)` 来指定消费的位置。

### 18、Replica、Leader 和 Follower 三者的概念

**Leader：** 副本中的领导者。负责对外提供服务，与客户端进行交互。

**Follower：** 副本中的跟随者。被动地跟随 Leader，不能与外界进行交付。只是向Leader发送消息，请求Leader把最新生产的消息发给它，进而保持同步。

### 19、Replica 的重要性

Replica 可以确保发布的消息不会丢失，保证了Kafka的高可用性。

### Kafka 中的 Geo-Replication 是什么

Kafka官方提供了MirrorMaker组件，作为跨集群的流数据同步方案。借助MirrorMaker，消息可以跨多个数据中心或云区域进行复制。您可以在主动/被动场景中将其用于备份和恢复，或者在主动/主动方案中将数据放置得更靠近用户，或支持数据本地化要求。

它的实现原理比较简单，就是通过从源集群消费消息，然后将消息生产到目标集群，即普通的消息生产和消费。用户只要通过简单的Consumer配置和Producer配置，然后启动Mirror，就可以实现集群之间的准实时的数据同步。

### Kafka 中 AR、ISR、OSR 三者的概念

* `AR`：分区中的所有副本称为 AR。
* `ISR`：所有与首领副本保持同步的副本（包括首领副本）称为 ISR。
* `OSR`：与首领副本滞后过多的副本组成 OSR。

### 分区副本什么情况下会从 ISR 中剔出剔除

为了与首领保持同步，跟随者需要向首领发送Fetch请求，这与消费者为了读取消息而发送的请求是一样的。如果副本没有在30秒内发送请求，或者即使发送了请求但与最新消息的间隔超过了30秒，那么它将被认为是不同步的。

### 分区副本中的 Leader 如果宕机但 ISR 却为空该如何处理

可以通过配置`unclean.leader.election` ：

* **true**：允许 OSR 成为 Leader，但是 OSR 的消息较为滞后，可能会出现消息不一致的问题。
* **false**：会一直等待旧 leader 恢复正常，降低了系统可用性。

### 如何判断一个 Broker 是否还有效

Broker启动时会在Zookeeper上注册一个临时节点，Zookeeper通过心跳机制检查每个结点的连接

### Kafka 可接收的消息最大默认多少字节，如何修改

Kafka可以接收的最大消息默认为**1000000**字节，如果想调整它的大小，可在Broker中修改配置参数：`message.max.bytes`的值。

### Kafka 的 ACK 机制

acks指定了生产者在多少个同步分区副本收到消息的情况下才会认为消息写入成功。

* 如果**acks=0**，则**生产者不会等待任何来自broker的响应**。也就是说，如果broker因为某些问题没有收到消息，那么生产者便无从得知，消息也就丢失了。不过，因为生产者不需要等待broker返回响应，所以它们能够以网络可支持的最大速度发送消息，从而达到很高的吞吐量。
* 如果**acks=1**，那么**只要集群的首领副本收到消息，生产者就会收到消息成功写入的响应**。如果消息无法到达首领副本（比如首领副本发生崩溃，新首领还未选举出来），那么生产者会收到一个错误响应。为了避免数据丢失，生产者会尝试重发消息。不过，在首领副本发生崩溃的情况下，如果消息还没有被复制到新的首领副本，则消息还是有可能丢失。
* 如果**acks=all**，那么**只有当所有同步副本全部收到消息时，生产者才会收到消息成功写入的响应**。这种模式是最安全的，它可以保证不止一个broker收到消息，就算有个别broker发生崩溃，整个集群仍然可以运行。

### Kafka 的 consumer 如何消费数据

在Kafka中，Producers将消息推送给Broker端，在Consumer和Broker建立连接之后，会主动去 Pull（或者说Fetch）消息。这种模式有些优点，首先Consumer端可以根据自己的消费能力适时的去fetch消息并处理，且可以控制消息消费的进度（offset）；此外，消费者可以控制每次消费的数量，实现批量消费。

### Kafka 的Topic中 Partition 数据是怎么存储到磁盘的

Topic 中的多个 Partition 以文件夹的形式保存到 Broker，每个分区序号从0递增，且消息有序。Partition 文件下有多个Segment（xxx.index，xxx.log），Segment文件里的大小和配置文件大小一致。默认为1GB，但可以根据实际需要修改。如果大小大于1GB时，会滚动一个新的Segment并且以上一个Segment最后一条消息的偏移量命名。

### Kafka 创建Topic后如何将分区放置到不同的 Broker 中

假设你有6个broker，打算创建一个包含10个分区的主题，并且复制系数为3，那么总共会有30个分区副本，它们将被分配给6个broker，此时，分配方案如下：

* **先随机选择一个broker（假设是4），然后使用轮询的方式给每个broker分配分区首领**。于是，分区0的首领在broker 4上，分区1的首领在broker 5上，分区2的首领在broker 0上（因为只有6个broker），以此类推。
* **接下来，从分区首领开始，依次分配跟随者副本**。如果分区0的首领在broker 4上，那么它的第一个跟随者副本就在broker 5上，第二个跟随者副本就在broker 0上。如果分区1的首领在broker 5上，那么它的第一个跟随者副本就在broker 0上，第二个跟随者副本在broker 1上。
* **如果配置了机架信息，那么就不是按照数字顺序而是按照机架交替的方式来选择broker了**。假设broker 0和broker 1被放置在一个机架上，broker 2和broker 3被放置在另一个机架上。我们不是按照从0到3的顺序来选择broker，而是按照0、2、1、3的顺序来选择，以保证相邻的broker总是位于不同的机架上。于是，如果分区0的首领在broker 2上，那么第一个跟随者副本就在broker 1上，以保证它们位于不同的机架上。

### Kafka 的日志保留期与数据清理策略

保留期内保留了Kafka群集中的所有已发布消息，超过保期的数据将被按清理策略进行清理。默认保留时间是7天，如果想修改时间，在`server.properties`里更改参数`log.retention.hours/minutes/ms` 的值便可。

**清理策略**

* **删除delete：** `log.cleanup.policy=delete` 表示启用删除策略，这也是默认策略。。
* **压实compact：** `log.cleanup.policy=compact` 表示启用压实策略，将数据压实，只保留每个Key最后一个版本的数据。首先在Broker的配置中设置`log.cleaner.enable=true` 启用 cleaner，这个默认是关闭的。

### Kafka 是否支持多租户隔离

> 多租户技术（multi-tenancy technology）是一种软件架构技术，它是实现如何在多用户的环境下共用相同的系统或程序组件，并且仍可确保各用户间数据的隔离性。

通过配置哪个主题可以生产或消费数据来启用多租户，也有对配额的操作支持。管理员可以对请求定义和强制配额，以控制客户端使用的Broker资源。

### Kafka 日志的存储格式

每个日志片段被保存在一个单独的数据文件中，文件中包含了消息和偏移量。**保存在磁盘上的数据格式与生产者发送给服务器的消息格式以及服务器发送给消费者的消息格式是一样的**。

### Kafka 的日志分段策略与刷新策略

**日志分段（Segment）策略**
