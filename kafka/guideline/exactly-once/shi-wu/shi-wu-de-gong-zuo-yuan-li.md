# 事务的工作原理

Kafka事务的基本算法受到了**Chandy-Lamport快照**的启发，它会将一种被称为**“标记”(marker)**的消息发送到通信通道中，并根据标记的到达情况来确定一致性状态。

Kafka事务根据**标记消息**来判断跨多个分区的事务是否被提交或被中止——当生产者要提交一个事务时，它会发送**“提交”消息**给**事务协调器**，事务协调器会将**提交标记**写入所有涉及这个事务的**分区**。如果生产者在向部分分区写入提交消息后发生崩溃，该怎么办？Kafka事务使用<mark style="color:blue;">**两阶段提交**</mark>和<mark style="color:blue;">**事务日志**</mark>来解决这个问题。

总的来说，这个算法会执行如下步骤：

1. 记录正在执行中的事务，包括所涉及的分区。
2. 记录提交或中止事务的意图——一旦被记录下来，到最后要么被提交，要么被中止。
3. 将所有事务标记写入所有分区。
4. 记录事务的完成情况。

要实现这个算法，Kafka需要一个事务日志。这里使用了一个叫作<mark style="color:blue;">**\_\_transaction\_state**</mark>的内部主题。

在开始一个事务之前，生产者需要通过调用<mark style="color:blue;">**initTransaction()**</mark>来注册自己。这个请求会被发送给一个broker，它将成为这个事务性生产者的<mark style="color:blue;">**事务协调器**</mark>。每一个事务ID对应的事务协调器就是映射到这个事务ID的事务日志分区的首领。<mark style="color:blue;">**initTransaction() API注册了一个带有新事务ID的协调器或者增加现有事务ID的epoch，用以隔离变成“僵尸”的旧生产者**</mark><mark style="color:blue;">。</mark>当epoch增加时，挂起的事务将被中止。

下一步是调用<mark style="color:blue;">**beginTransaction()**</mark>。这个方法不是协议的一部分，它只是**告诉生产者，现在有一个正在执行中的事务**。broker端的事务协调器仍然不知道事务已经开始。不过，一旦生产者开始发送消息，**每次生产者检测到消息被发送给一个新分区时，都会向broker发送AddPartitionsToTxnRequest请求，告诉broker自己有一个执行中的事务，并且这些分区是事务的一部分**。这些信息将被记录在事务日志中。

当生成结果并准备提交事务时，首先需要提交在这个事务中处理好的消息的偏移量。**偏移量可以在任何时候提交，但一定要在事务提交之前**。<mark style="color:blue;">**sendOffsetsToTransaction()**</mark>**方法将向事务协调器发送一个请求，其中包含了偏移量和消费者群组ID**。事务协调器将用消费者群组ID查找群组协调器，并提交偏移量。

<mark style="color:blue;">**commitTransaction()**</mark>**方法或**<mark style="color:blue;">**abortTransaction()**</mark>**方法**将向事务协调器发送一个EndTransactionRequest。事务协调器会把提交或中止事务的意图记录到事务日志中。**如果这个步骤执行成功，那么事务协调器将负责完成提交（或中止）过程。它会向所有涉及事务的分区写入一个提交标记，然后将提交成功的信息写入事务日志。**

需要注意的是，**如果事务协调器在记录提交意图之后以及在完成提交流程之前被关闭或发生崩溃，那么将会选举出一个新的事务协调器，它会从事务日志中获取提交意图，并完成提交流程。**

如果一个事务未能在**transaction.timeout.ms**指定的时间内提交或中止，则事务协调器将自动中止它。

{% hint style="info" %}
<mark style="color:blue;">**每个收到由事务性或幂等生产者发送的消息的broker都会在内存中保存生产者ID或事务性ID，以及生产者发送的最后5个消息批次的相关状态：序列号、偏移量等。**</mark>这些状态在生产者停止活动之后会继续保留transactional.id.expiration.ms指定的时间（默认为7天）。这样生产者就可以在不抛出UNKNOWN\_PRODUCER\_ID异常的情况下恢复活动。

<mark style="color:red;">**如果以非常高的速率创建新的幂等生产者或事务ID，但从不重用它们，则可能会导致broker发生内存泄漏。**</mark>如果在一周内连续每秒新增3个幂等生产者，那么将产生180万个状态条目，总共需要保存900万个消息批次元数据，占用大约5 GB内存。这可能会导致broker内存不足或出现严重的垃圾回收停顿。

<mark style="color:orange;">**建议在应用程序启动时初始化几个长期使用的生产者，并在应用程序生命周期中重用它们。如果不能这么做（FaaS会让这变得很困难），那么建议减小transactional.id.expiration.ms的值，这样事务ID就会更快过期，不会让旧状态占用broker太大的内存。**</mark>
{% endhint %}
