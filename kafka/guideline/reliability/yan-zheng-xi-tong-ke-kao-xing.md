# 验证系统可靠性

对系统可靠性做3个层面的验证：<mark style="color:blue;">**验证配置**</mark>、<mark style="color:blue;">**验证应用程序**</mark>以及<mark style="color:blue;">**在生产环境中监控可靠性**</mark>。

## 验证配置

Kafka提供了两个重要的配置验证工具：**org.apache.kafka.tools** 包下面的**VerifiableProducer** 类和 **VerifiableConsumer** 类。可以在命令行中运行这两个工具，或把它们嵌入自动化测试框架中。

**VerifiableProducer** 会生成一系列消息，消息里包含了一个从1到指定数字的数字。可以**使用与真实的生产者相同的配置参数来配置 VerifiableProducer**，比如配置相同的 acks、retries、delivery.timeout.ms 和消息生成速率。在运行过程中，**VerifiableProducer** 会根据收到的确认响应将每条消息发送成功或失败的结果打印出来。

反过来，**VerifiableConsumer** 会读取由 **VerifiableProducer** 生成的消息，并按照读取顺序把它们打印出来，它也会把与偏移量和再均衡相关的信息打印出来。

要仔细考虑需要测试哪些场景，比如以下场景：

* **首领选举**：如果停掉首领会发生什么事情？生产者和消费者需要多长时间来恢复状态？
* **控制器选举**：重启控制器后系统需要多少时间来恢复状态？
* **滚动重启**：可以滚动重启broker而不丢失消息吗？
* **不彻底的首领选举**：如果依次停止一个分区的所有副本（确保每个副本都变为不同步的），然后启动一个不同步的broker会发生什么？要怎样才能恢复正常？这样做是可接受的吗？

**从中选择一个场景，比如停掉正在写入消息的分区的首领，然后启动VerifiableProducer 和 VerifiableConsumer 开始进行测试。**如果期望在一个短暂的停顿之后恢复正常，并且没有丢失任何消息，那么只需验证一下生产者生成的消息条数与消费者读取的消息条数相匹配即可。

## 验证应用程序

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

## 在生产环境中监控可靠性

对应用程序进行测试固然重要，但它无法取代在生产环境中对应用程序进行持续的监控，以确保数据按照期望的方式流动。**除了监控集群的健康状况，也要对客户端和数据流进行监控。**

Kafka的Java客户端提供了一些**JMX指标**，可用于监控客户端的状态和事件：

对**生产者**来说，最重要的两个可靠性指标是消息的**错误率**和**重试率**（聚合过的）。如果这两个指标上升，则说明系统出问题了。除此之外，还要监控生产者日志，注意那些被设为 WARN 级别的错误日志，以及包含“Got error produce response with correlation id 5689 on topic-partition \[topic-1,3], retrying (two attempts left). Error: ...”的日志。如果看到消息剩余的重试次数为0，则说明生产者已经没有多余的重试机会。

在**消费者**端，最重要的指标是**消费者滞后指标**。这个指标告诉我们消费者正在处理的消息与最新提交到分区的消息偏移量之间有多少差距。理想情况下，这个指标的值是0，也就是说消费者读取的是最新的消息。但是，在实际当中，因为 poll() 方法会返回很多消息，消费者在读取下一批数据之前需要花一些时间来处理它们，所以这个指标会有些**波动**。**关键在于要确保消费者最终会赶上去，而不是越落越远**。因为会出现正常波动，所以为这个指标配置告警有一定难度。**Burrow**是LinkedIn公司开发的一款消费者滞后检测工具，其可以让这个指标的配置变得容易一些。

监控**数据流**是为了**确保所有生成的数据会被及时地读取**（这里的“及时”视具体的业务需求而定）。为了确保数据能够被及时读取，需要知道数据是什么时候生成的。从0.10.0版本开始，Kafka在所有消息里加入了**生成消息的时间戳**（但需要注意的是，发送消息的应用程序或配置了相应参数的broker都可以覆盖这个时间戳）。为了确保所有消息在合理的时间内被读取，应用程序需要记录生成消息的数量（一般用每秒多少条消息来表示）。消费者需要记录单位时间内已读取消息的数量以及消息生成时间与读取时间之间的时间差（根据消息的时间戳来判断）。然后，我们需要一个系统将生产者和消费者记录的消息数量收集起来（确保没有丢失消息），确保二者之间的时间差在合理的范围内。

除了监控客户端和端到端数据流，broker还提供了一些指标，用于说明broker发送给客户端的**错误响应率**。建议收集**kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec** 和**kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec** 这两个指标。