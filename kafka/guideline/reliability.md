# 六、可靠的数据传递

可靠性并不只是Kafka单方面的事情。应该从整个系统层面来考虑可靠性问题，包括**应用程序架构**、**生产者和消费者API的使用方式**、**生产者和消费者的配置**、**主题的配置以及broker的配置**。为了提升系统的可靠性，需要在许多方面（比如复杂性、性能、可用性和磁盘空间）做出权衡。

### 6.1 复制

**Kafka的复制机制和分区多副本架构是Kafka可靠性保证的核心**。把消息写入多个副本可以保证Kafka在发生崩溃时仍然能够提供消息的持久性。

Kafka的**主题**会被分为多个**分区**，分区是最基本的数据构建块。分区存储在单个磁盘上，**Kafka可以保证分区中的事件是有序的**。一个分区可以在线（可用），也可以离线（不可用）。每个分区可以有多个**副本**，其中有一个副本是**首领**。所有的事件都会被发送给首领副本，通常情况下消费者也直接从首领副本读取事件。其他副本只需要与首领保持同步，及时从首领那里复制最新的事件。当首领副本不可用时，其中一个同步副本将成为新首领。

**分区的首领肯定是同步副本**，而对**跟随者副本**来说，则需要满足以下条件才能被认为是同步副本。

* 与ZooKeeper之间有一个活跃的会话，也就是说，它在过去的6秒（可配置）内向ZooKeeper发送过心跳。
* 在过去的10秒（可配置）内从首领那里复制过消息。
* 在过去的10秒内从首领那里复制过最新的消息。仅从首领那里复制消息是不够的，它还必须在每10秒（可配置）内复制一次最新的消息。

如果跟随者副本不能满足以上任何一点（比如与ZooKeeper断开连接，或者不再复制新消息，或者复制消息滞后了10秒以上），那么它就会被认为是不同步的。一个不同步的副本可以通过与ZooKeeper重新建立连接并从首领那里复制最新的消息重新变成同步副本。

一个稍有**滞后的同步副本会导致生产者和消费者变慢**，因为在消息被认为已提交之前，客户端会等待所有同步副本确认消息。如果一个副本变成不同步的，那么我们就不再关心它是否已经收到消息。这个时候，**虽然不同步副本同样是滞后的，但它不影响性能**。然而，更少的同步副本意味着更小的有效复制系数，因此在停机时丢失数据的风险就更大了。

### 6.2 broker配置

broker中有3个配置参数会影响Kafka的消息存储可靠性。与其他配置参数一样，它们既可以配置在broker级别，用于控制所有主题的行为，也可以配置在主题级别，用于控制个别主题的行为。

#### 6.2.1 复制系数

主题级别的配置参数是 **replication.factor**。在broker级别，可以通过**default.replication.factor** 来设置自动创建的主题的复制系数。

那么该如何确定一个主题需要几个副本呢？这个时候需要考虑以下因素：

* **可用性**：**副本越多，可用性就越高**。
* **持久性**：每个副本都包含了一个分区的所有数据。**如果有更多的副本，并且这些副本位于不同的存储设备中，那么丢失所有副本的概率就降低了。**
* **吞吐量**：**每增加一个副本都会增加broker内的复制流量**。如果以10 MBps的速率向一个分区发送数据，并且只有1个副本，那么不会增加任何的复制流量。如果有2个副本，则会增加10 MBps的复制流量，3个副本会增加20 MBps的复制流量，5个副本会增加40 MBps的复制流量。在规划集群大小和容量时，需要把这个考虑在内。
* **端到端延迟**：每一条记录必须被复制到所有同步副本之后才能被消费者读取。从理论上讲，**副本越多，出现滞后的可能性就越大，因此会降低消费者的读取速度**。在实际当中，如果一个broker由于各种原因变慢，那么它就会影响所有的客户端，而不管复制系数是多少。
* **成本：**一般来说，出于成本方面的考虑，非关键数据的复制系数应该小于3。**数据副本越多，存储和网络成本就越高。**因为很多存储系统已经将每个数据块复制了3次，所以有时候可以将Kafka的复制系数设置为2，以此来降低成本。需要注意的是，与复制系数3相比，这样做仍然会降低可用性，但可以由存储设备来提供持久性保证。

**副本的位置分布也很重要。**Kafka可以确保**分区的每个副本被放在不同的broker上**。但是，在某些情况下，这样仍然不够安全。如果一个分区的所有副本所在的broker位于同一个机架上，那么一旦机架的交换机发生故障，不管设置了多大的复制系数，这个分区都不可用。**为了避免机架级别的故障，建议把broker分布在多个不同的机架上，并通过 broker.rack 参数配置每个broker所在的机架的名字。**如果配置了机架名字，那么Kafka就会保证分区的副本被分布在多个机架上，从而获得更高的可用性。**如果是在云端运行Kafka，则可以将可用区域视为机架。**

#### 6.2.2 不彻底的首领选举

**unclean.leader.election.enable** 只能在broker级别（实际上是在集群范围内）配置，它的默认值是 false。

当分区的首领不可用时，一个同步副本将被选举为新首领。如果在选举过程中未丢失数据，也就是说所有同步副本都包含了已提交的数据，那么这个选举就是**“彻底”**的。

**如果允许不同步副本成为首领，那么就要承担丢失数据和消费者读取到不一致的数据的风险。**如果不允许它们成为首领，那么就要接受较低的可用性，因为必须等待原先的首领恢复到可用状态。

#### 6.2.3 最少同步副本

**min.insync.replicas** 参数可以配置在主题级别和broker级别。

尽管为一个主题配置了3个副本，还是会出现**只剩下一个同步副本**的情况。如果这个同步副本变为不可用，则必须在可用性和一致性之间做出选择，而这是一个两难的选择。根据Kafka对可靠性保证的定义，一条消息只有在被写入所有同步副本之后才被认为是已提交的，但**如果这里的“所有”只包含一个同步副本，那么当这个副本变为不可用时，数据就有可能丢失。**

**如果想确保已提交的数据被写入不止一个副本，就要把最少同步副本设置得大一些。对于一个包含3个副本的主题，如果 min.insync.replicas 被设置为2，那么至少需要有两个同步副本才能向分区写入数据。**如果有两个副本变为不可用，那么broker就会停止接受生产者的请求。尝试发送数据的生产者会收到 **NotEnoughReplicasException** 异常，不过消费者仍然可以继续读取已有的数据。实际上，如果使用这样的配置，那么当只剩下一个同步副本时，它就变成只读的了。

#### 6.2.4 保持副本同步

一个副本可能在两种情况下变得不同步：要么它**与ZooKeeper断开连接**，要么它**从首领复制消息滞后**。对于这两种情况，Kafka提供了两个broker端的配置参数。

**zookeeper.session.timeout.ms** 是允许broker不向ZooKeeper发送心跳的时间间隔。如果超过这个时间不发送心跳，则ZooKeeper会认为broker已经“死亡”，并将其从集群中移除。在Kafka 2.5.0中，这个参数的默认值从6秒增加到了18秒，以提高Kafka集群在云端的稳定性，因为云环境的网络延迟更加多变。一般来说，我们希望将这个值设置得足够大，以避免因垃圾回收停顿或网络条件造成的随机抖动，但又要设置得足够小，以确保及时检测到确实已经发生故障的broker。

如果一个副本未能在 **replica.lag.time.max.ms** 指定的时间内从首领复制数据或赶上首领，那么它将变成不同步副本。在Kafka 2.5.0中，这个参数的默认值从10秒增加到了30秒，以提高集群的弹性，并避免不必要的抖动。需要注意的是，这个值也会影响消费者的最大延迟——值越大，等待一条消息被写入所有副本并可被消费者读取的时间就越长，最长可达30秒。

#### 6.2.5 持久化到磁盘

即使消息还没有被持久化到磁盘上，Kafka也可以向生产者发出确认，这取决于已接收到消息的副本的数量。**Kafka会在重启之前和关闭日志片段（默认1 GB大小时关闭）时将消息冲刷到磁盘上，或者等到Linux系统页面缓存被填满时冲刷。**其背后的想法是，拥有3台放置在不同机架或可用区域的机器，并在每台机器上放置一份数据副本比只将消息写入首领的磁盘更加安全，因为两个不同的机架或可用区域同时发生故障的可能性非常小。不过，也可以让broker更频繁地将消息持久化到磁盘上。配置参数 **flush.messages** 用于控制未同步到磁盘的最大消息数量，**flush.ms** 用于控制同步频率。在配置这些参数之前，最好先了解一下fsync 是如何影响Kafka的吞吐量的以及如何尽量避开它的缺点。

### 6.3 在可靠的系统中使用生产者

即使我们会尽可能地把broker配置得很可靠，但如果没有对生产者进行可靠性方面的配置，则整个系统仍然存在丢失数据的风险。

> 请看下面的两个例子。
>
> * 我们**为broker配置了3个副本，并禁用了不彻底的首领选举**，这样应该可以保证已提交的消息不会丢失。不过，我们把**生产者发送消息的 acks 设置成了 1**。生产者向首领发送了一条消息，虽然其被首领成功写入，但其他同步副本还没有收到这条消息。首领向生产者发送了一个响应，告诉它“消息写入成功”，然后发生了崩溃，而此时其他副本还没有复制这条消息。另外两个副本此时仍然被认为是同步的，并且其中的一个副本会成为新首领。因为消息还没有被写入这两个副本，所以就丢失了，但发送消息的客户端认为消息已经成功写入。从消费者的角度来看，系统仍然是一致的，因为它们看不到丢失的消息（副本没有收到这条消息，不算已提交），但从生产者的角度来看，这条消息丢失了。
> * 我们**为broker配置了3个副本，并禁用了不彻底的首领选举**。我们接受了之前的教训，把**生产者的 acks 设置成了 all**。假设现在生产者向Kafka发送了一条消息，此时分区首领刚好发生崩溃，新首领正在选举当中，Kafka会向生产者返回“首领不可用”的响应。在这个时候，如果生产者未能正确处理这个异常，也没有重试发送消息，那么消息也有可能丢失。这不算是broker的可靠性问题，因为broker并没有收到这条消息；这也不是一致性问题，因为消费者也不会读取到这条消息。问题在于，如果生产者未能正确处理异常，就有可能丢失数据。

从上面的两个例子可以看出，开发人员需要注意两件事情。

* 根据可靠性需求配置恰当的 acks。
* 正确配置参数，并在代码里正确处理异常。

#### 6.3.1 发送确认

生产者可以选择以下3种确认模式：

* **acks=0**：如果生产者能够通过网络把消息发送出去，那么就认为消息已成功写入Kafka。不过，在这种情况下仍然有可能出现错误，比如发送的消息对象无法被序列化或者网卡发生故障。如果此时分区离线、正在进行首领选举或整个集群长时间不可用，则并不会收到任何错误。在 acks=0 模式下，生产延迟是很低的，但它对端到端延迟并不会带来任何改进（在消息被所有可用副本复制之前，消费者是看不到它们的）。
* **acks=1**：首领在收到消息并把它写入分区数据文件（不一定要冲刷到磁盘上）时会返回确认或错误响应。在这种模式下，如果首领被关闭或发生崩溃，那么那些已经成功写入并确认但还没有被跟随者复制的消息就丢失了。
* **acks=all**：首领在返回确认或错误响应之前，会等待所有同步副本都收到消息。这个配置可以和 min.insync.replicas 参数结合起来，用于控制在返回确认响应前至少要有多少个副本收到消息。这是最安全的选项，因为生产者会一直重试，直到消息提交成功。

#### 6.3.2 配置生产者的重试参数

生产者需要处理的错误包括两个部分：一部分是由生产者自动处理的错误，另一部分是需要开发者手动处理的错误。

生产者可以自动处理**可重试**的错误。当生产者向broker发送消息时，broker可以返回一个成功响应或者错误响应。错误响应可以分为**两种**，一种是在**重试之后可以解决的**，另一种是**无法通过重试解决的**。如果broker返回 LEADER\_NOT\_AVAILABLE 错误，那么生产者可以尝试重新发送消息——或许新首领被选举出来了，那么第二次尝试发送就会成功。也就是说，LEADER\_NOT\_AVAILABLE 是一个**可重试**错误。如果broker返回 INVALID\_CONFIG 错误，那么即使重试发送消息也无法解决这个问题，所以这样的重试是没有意义的，这是**不可重试**错误。

一般来说，如果你的目标是不丢失消息，那么就让生产者在遇到可重试错误时保持重试。**最好的重试方式是使用默认的重试次数（整型最大值或无限），并把 delivery.timeout.ms 配置成我们愿意等待的时长，生产者会在这个时间间隔内一直尝试发送消息。**

重试发送消息存在一定的风险，因为如果两条消息都成功写入，则会导致消息重复。通过重试和小心翼翼地处理异常，可以保证每一条消息都会被保存至少一次，但不能保证只保存一次。**如果把 enable.idempotence 参数设置为 true，那么生产者就会在消息里加入一些额外的信息，broker可以使用这些信息来跳过因重试导致的重复消息。**

#### 6.3.3 额外的错误处理

使用生产者内置的重试机制可以在不造成消息丢失的情况下轻松地处理大部分错误，但开发人员仍然需要处理以下这些其他类型的错误：

* 不可重试的broker错误，比如消息大小错误、身份验证错误等。
* 在将消息发送给broker之前发生的错误，比如序列化错误。
* 在生产者达到重试次数上限或重试消息占用的内存达到上限时发生的错误。
* 超时。

这些错误的处理逻辑与具体的应用程序及其目标有关——丢弃“不合法的消息”？把错误记录下来？停止从源系统读取消息？对源系统应用回压策略以便暂停发送消息？把消息保存到本地磁盘的某个目录里？具体使用哪一种逻辑要根据实际的架构和产品需求来决定。**只需记住，如果错误处理只是为了重试发送消息，那么最好还是使用生产者内置的重试机制。**

### 6.4 在可靠的系统中使用消费者

**只有已经被提交到Kafka的数据，也就是已经被写入所有同步副本的数据，对消费者是可用的**。这保证了消费者读取到的数据是一致的。**消费者唯一要做的是跟踪哪些消息是已经读取过的，哪些消息是还未读取的，这是消费者在读取消息时不丢失消息的关键。**

#### 6.4.1 消费者的可靠性配置

为了保证消费者行为的可靠性，需要注意以下4个非常重要的配置参数：

* **group.id**：如果两个消费者具有相同的群组ID，并订阅了同一个主题，那么每个消费者将分到主题分区的一个子集，也就是说它们只能读取到所有消息的一个子集（但整个群组可以读取到主题所有的消息）。如果你希望一个消费者可以读取主题所有的消息，那么就需要为它设置唯一的 group.id。
* **auto.offset.reset**：这个参数指定了当没有偏移量（比如在消费者首次启动时）或请求的偏移量在broker上不存在时消费者该作何处理。这个参数有两个值，一个是 **earliest**，如果配置了这个值，那么消费者将从分区的开始位置读取数据，这会导致消费者读取大量的重复数据，但可以保证最少的数据丢失。另一个值是 **latest**，如果配置了这个值，那么消费者将从分区的末尾位置读取数据，这样可以减少重复处理消息，但很有可能会错过一些消息。
* **enable.auto.commit**：你可以决定让消费者自动提交偏移量，也可以在代码里手动提交偏移量。自动提交的一个最大好处是可以少操心一些事情。如果是在消费者的消息轮询里处理数据，那么自动提交可以确保不会意外提交未处理的偏移量。自动提交的主要缺点是我们无法控制应用程序可能重复处理的消息的数量，比如消费者在还没有触发自动提交之前处理了一些消息，然后被关闭。如果应用程序的处理逻辑比较复杂（比如把消息交给另外一个后台线程去处理），那么就只能使用手动提交了，因为自动提交机制有可能会在还没有处理完消息时就提交偏移量。
* **auto.commit.interval.ms**：如果选择使用自动提交，那么可以通过这个参数来控制提交的频率，默认每5秒提交一次。一般来说，**频繁提交会增加额外的开销，但也会降低重复处理消息的概率**。

#### 6.4.2 手动提交偏移量

如果想要更大的灵活性，选择了手动提交，那么就需要考虑正确性和性能方面的问题：

1.  **总是在处理完消息后提交偏移量**

    在轮询过程中提交偏移量有一个缺点，就是有可能会意外提交已读取但未处理的消息的偏移量。一定要在处理完消息后再提交偏移量，这点很关键——提交已读取但未处理的消息的偏移量会导致消费者错过消息。
2.  **提交频率是性能和重复消息数量之间的权衡**

    提交频率需要在性能需求和重复消息量之间取得平衡。处理一条消息就提交一次偏移量的方式只适用于吞吐量非常低的主题。
3.  **再均衡**

    在设计应用程序时，需要考虑到消费者会发生再均衡并需要处理好它们。
4.  **消费者可能需要重试**

    有时候，在调用了轮询方法之后，有些消息需要稍后再处理。需要注意的是，消费者提交偏移量并不是对单条消息的“确认”，这与传统的发布和订阅消息系统不一样。也就是说，如果记录 #30处理失败，但记录 #31处理成功，那么就不应该提交记录 #31的偏移量——如果提交了，就表示 #31以内的记录都已处理完毕，包括记录 #30在内，但这可能不是我们想要的结果。不过，可以采用以下两种模式来解决这个问题：

    * 第一种模式，在遇到可重试错误时，提交最后一条处理成功的消息的偏移量，然后把还未处理好的消息保存到缓冲区（这样下一个轮询就不会把它们覆盖掉），并调用消费者的 **pause()** 方法，确保其他的轮询不会返回数据，之后继续处理缓冲区里的消息。
    * 第二种模式，在遇到可重试错误时，把消息写到另一个**重试主题**，并继续处理其他消息。另一个消费者群组负责处理重试主题中的消息，或者让一个消费者同时订阅主主题和重试主题。这种模式有点儿像其他消息系统中的死信队列。
5.  **消费者可能需要维护状态**

    在一些应用程序中，需要维护多个轮询之间的状态。如果想计算移动平均数，就需要在每次轮询之后更新结果。如果应用程序重启，则不仅需要从上一个偏移量位置开始处理消息，还需要恢复之前保存的移动平均数。**一种办法是在提交偏移量的同时把算好的移动平均数写到一个“结果”主题中。**当一个线程重新启动时，它就可以获取到之前算好的移动平均数，并从上一次提交的偏移量位置开始读取数据。

    一般来说，这是一个比较复杂的问题，建议尝试使用其他框架，比如Kafka Streams或Flink，它们为聚合、连接、时间窗和其他复杂的分析操作提供了高级的DSL API。

### 6.5 验证系统可靠性

对系统可靠性做3个层面的验证：**验证配置**、**验证应用程序**以及**在生产环境中监控可靠性**。

#### 6.5.1 验证配置

Kafka提供了两个重要的配置验证工具：**org.apache.kafka.tools** 包下面的**VerifiableProducer** 类和 **VerifiableConsumer** 类。可以在命令行中运行这两个工具，或把它们嵌入自动化测试框架中。

**VerifiableProducer** 会生成一系列消息，消息里包含了一个从1到指定数字的数字。可以**使用与真实的生产者相同的配置参数来配置 VerifiableProducer**，比如配置相同的 acks、retries、delivery.timeout.ms 和消息生成速率。在运行过程中，**VerifiableProducer** 会根据收到的确认响应将每条消息发送成功或失败的结果打印出来。

反过来，**VerifiableConsumer** 会读取由 **VerifiableProducer** 生成的消息，并按照读取顺序把它们打印出来，它也会把与偏移量和再均衡相关的信息打印出来。

要仔细考虑需要测试哪些场景，比如以下场景：

* **首领选举**：如果停掉首领会发生什么事情？生产者和消费者需要多长时间来恢复状态？
* **控制器选举**：重启控制器后系统需要多少时间来恢复状态？
* **滚动重启**：可以滚动重启broker而不丢失消息吗？
* **不彻底的首领选举**：如果依次停止一个分区的所有副本（确保每个副本都变为不同步的），然后启动一个不同步的broker会发生什么？要怎样才能恢复正常？这样做是可接受的吗？

**从中选择一个场景，比如停掉正在写入消息的分区的首领，然后启动VerifiableProducer 和 VerifiableConsumer 开始进行测试。**如果期望在一个短暂的停顿之后恢复正常，并且没有丢失任何消息，那么只需验证一下生产者生成的消息条数与消费者读取的消息条数相匹配即可。

#### 6.5.2 验证应用程序

在确定broker和客户端的配置可以满足需求之后，接下来要验证应用程序是否能够提供我们想要的保证。应用程序的验证包括**检查错误处理逻辑**、**偏移量提交的方式**、**再均衡监听器**以及**其他使用了Kafka客户端的地方**。

建议将应用程序集成测试作为开发流程的一部分，并针对以下这些故障场景做一些测试：

* 客户端与服务器断开连接
* 客户端与服务器之间存在高延迟
* 磁盘被填满
* 磁盘被挂起（也就是所谓的“掉电”）
* 首领选举
* 滚动重启broker
* 滚动重启消费者
* 滚动重启生产者

> Kafka项目本身也提供了一个故障注入框架**Trogdor**。

#### 6.5.3 在生产环境中监控可靠性

对应用程序进行测试固然重要，但它无法取代在生产环境中对应用程序进行持续的监控，以确保数据按照期望的方式流动。**除了监控集群的健康状况，也要对客户端和数据流进行监控。**

Kafka的Java客户端提供了一些**JMX指标**，可用于监控客户端的状态和事件：

对**生产者**来说，最重要的两个可靠性指标是消息的**错误率**和**重试率**（聚合过的）。如果这两个指标上升，则说明系统出问题了。除此之外，还要监控生产者日志，注意那些被设为 WARN 级别的错误日志，以及包含“Got error produce response with correlation id 5689 on topic-partition \[topic-1,3], retrying (two attempts left). Error: ...”的日志。如果看到消息剩余的重试次数为0，则说明生产者已经没有多余的重试机会。

在**消费者**端，最重要的指标是**消费者滞后指标**。这个指标告诉我们消费者正在处理的消息与最新提交到分区的消息偏移量之间有多少差距。理想情况下，这个指标的值是0，也就是说消费者读取的是最新的消息。但是，在实际当中，因为 poll() 方法会返回很多消息，消费者在读取下一批数据之前需要花一些时间来处理它们，所以这个指标会有些**波动**。**关键在于要确保消费者最终会赶上去，而不是越落越远**。因为会出现正常波动，所以为这个指标配置告警有一定难度。**Burrow**是LinkedIn公司开发的一款消费者滞后检测工具，其可以让这个指标的配置变得容易一些。

监控**数据流**是为了**确保所有生成的数据会被及时地读取**（这里的“及时”视具体的业务需求而定）。为了确保数据能够被及时读取，需要知道数据是什么时候生成的。从0.10.0版本开始，Kafka在所有消息里加入了**生成消息的时间戳**（但需要注意的是，发送消息的应用程序或配置了相应参数的broker都可以覆盖这个时间戳）。为了确保所有消息在合理的时间内被读取，应用程序需要记录生成消息的数量（一般用每秒多少条消息来表示）。消费者需要记录单位时间内已读取消息的数量以及消息生成时间与读取时间之间的时间差（根据消息的时间戳来判断）。然后，我们需要一个系统将生产者和消费者记录的消息数量收集起来（确保没有丢失消息），确保二者之间的时间差在合理的范围内。

除了监控客户端和端到端数据流，broker还提供了一些指标，用于说明broker发送给客户端的**错误响应率**。建议收集**kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec** 和**kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec** 这两个指标。