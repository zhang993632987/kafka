# 偏移量的相关属性

## <mark style="color:blue;">**auto.offset.reset**</mark>

这个属性指定了**消费者在读取一个没有偏移量或偏移量无效（因消费者长时间不在线，偏移量对应的记录已经过期并被删除）的分区时该做何处理**。

* 它的**默认值是latest**，意思是说，如果没有有效的偏移量，那么消费者将从最新的记录（在消费者启动之后写入Kafka的记录）开始读取。
* 另一个值是**earliest**，意思是说，如果没有有效的偏移量，那么消费者将从起始位置开始读取记录。

如果将auto.offset.reset设置为none，并试图用一个无效的偏移量来读取记录，则消费者将抛出异常。

## <mark style="color:blue;">**enable.auto.commit**</mark>

这个属性指定了**消费者是否自动提交偏移量，默认值是true**。

如果它被设置为true，那么还有另外一个属性**auto.commit.interval.ms**可以用来**控制偏移量的提交频率**。

{% hint style="info" %}
## <mark style="color:blue;">**offsets.retention.minutes**</mark>

**这是broker端的一个配置属性**，需要注意的是，它也会影响消费者的行为。

<mark style="color:blue;">**只要消费者群组里有活跃的成员（也就是说，有成员通过发送心跳来保持其身份），群组提交的每一个分区的最后一个偏移量就会被Kafka保留下来，在进行重分配或重启之后就可以获取到这些偏移量**</mark><mark style="color:blue;">。</mark>

但是，如果一个消费者群组失去了所有成员，则Kafka只会按照这个属性指定的时间（默认为7天）保留偏移量。一旦偏移量被删除，即使消费者群组又“活”了过来，它也会像一个全新的群组一样，没有了过去的消费记忆。
{% endhint %}
