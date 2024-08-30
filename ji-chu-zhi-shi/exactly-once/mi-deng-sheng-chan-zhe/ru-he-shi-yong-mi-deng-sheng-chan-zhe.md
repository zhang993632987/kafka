# 如何使用幂等生产者

{% hint style="info" %}
<mark style="color:blue;">**幂等生产者使用起来非常简单，只需在生产者配置中加入 enable.idempotence=true。如果生产者已经配置了 acks=all，那么在性能上就不会有任何差异。**</mark>
{% endhint %}

在启用了幂等生产者之后，会发生下面这些变化。

* 为了获取生产者 ID，生产者在启动时会调用一个额外的 API。
*   <mark style="color:blue;">**每个消息批次里的第一条消息都将包含生产者 ID和序列号（批次里其他消息的序列号基于第一条消息的序列号递增）。**</mark>

    > 这些新字段给每个消息批次增加了 96 位（生产者 ID 是长整型，序列号是整型），这对大多数工作负载来说几乎算不上是额外的开销。
* broker 将会验证来自每一个生产者实例的序列号，并保证没有重复消息。
* <mark style="color:blue;">**每个分区的消息顺序都将得到保证，即使 max.in.flight.requests.per.connection 被设置为大于 1 的值**</mark>**（5 是默认值，这也是幂等生产者可以支持的最大值）。**
