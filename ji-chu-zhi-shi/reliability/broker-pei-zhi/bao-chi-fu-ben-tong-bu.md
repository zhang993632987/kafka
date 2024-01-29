# 保持副本同步

一个副本可能在两种情况下变得不同步：要么它**与 ZooKeeper 断开连接**，要么它**从首领复制消息滞后**。对于这两种情况，Kafka 提供了两个 broker 端的配置参数：

*   <mark style="color:blue;">**zookeeper.session.timeout.ms**</mark> 是允许 broker 不向 ZooKeeper 发送心跳的时间间隔。如果超过这个时间不发送心跳，则 ZooKeeper 会认为 broker 已经“死亡”，并将其从集群中移除。

    一般来说，我们**希望将这个值设置得足够大，以避免因垃圾回收停顿或网络条件造成的随机抖动，但又要设置得足够小，以确保及时检测到确实已经发生故障的 broker**。

    > 在Kafka 2.5.0中，这个参数的默认值从 6 秒增加到了 18 秒，以提高 Kafka 集群在云端的稳定性，因为云环境的网络延迟更加多变。
*   如果一个副本未能在 <mark style="color:blue;">**replica.lag.time.max.ms**</mark> 指定的时间内从首领复制数据或赶上首领，那么它将变成不同步副本。

    需要注意的是，**这个值也会影响消费者的最大延迟——值越大，等待一条消息被写入所有副本并可被消费者读取的时间就越长**，最长可达 30 秒。

    > 在 Kafka 2.5.0 中，这个参数的默认值从 10 秒增加到了 30 秒，以提高集群的弹性，并避免不必要的抖动。