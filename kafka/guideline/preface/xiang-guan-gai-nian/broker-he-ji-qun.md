# broker和集群

## broker

一台单独的Kafka服务器被称为<mark style="color:blue;">**broker**</mark>。

* broker会**接收来自生产者的消息**，为其**设置偏移量**，并**提交到**<mark style="color:blue;">**磁盘**</mark>**保存**。
* broker会**为消费者提供服务**，对**读取分区的请求做出响应**，并**返回已经发布的消息**。

## 集群

broker组成了集群。

* 每个集群都有一个同时充当了**集群控制器**角色的broker（自动从活动的集群成员中选举出来）。
  * <mark style="color:blue;">**控制器负责管理工作，包括为broker分配分区和监控broker**</mark>**。**
* **在集群中，一个分区从属于一个broker，这个broker被称为分区的首领**。
  * **一个被分配给其他broker的分区副本叫作这个分区的“跟随者”。**
  * **分区复制提供了分区的消息冗余，如果一个broker发生故障，则其中的一个跟随者可以接管它的领导权。**
  * <mark style="color:blue;">**所有想要发布消息的生产者必须连接到首领**</mark><mark style="color:blue;">，</mark>但<mark style="color:blue;">**消费者可以从首领或者跟随者那里读取消息**</mark><mark style="color:blue;">。</mark>

## 保留消息

<mark style="color:blue;">**保留消息**</mark>（在一定期限内）是Kafka的一个重要特性。

* <mark style="color:blue;">**broker默认的消息保留策略是这样的：要么保留一段时间（如7天），要么保留消息总量达到一定的字节数（如1 GB）**</mark>**。**当消息数量达到这些上限时，旧消息就会过期并被删除。所以，在任意时刻，可用消息的总量都不会超过配置参数所指定的大小。
* <mark style="color:blue;">**主题可以配置自己的保留策略**</mark>，将消息保留到不再使用它们为止。我们可以把主题配置成<mark style="color:blue;">**紧凑型日志**</mark><mark style="color:blue;">，</mark><mark style="color:blue;">**只有最后一条带有特定键的消息会被保留下来。**</mark>
