# 配额和节流

Kafka 可以限制生产消息和消费消息的速率，这是通过配额机制来实现的。

<mark style="color:blue;">**Kafka 提供了 3 种配额类型：生产、消费和请求**</mark>。

* **生产配额和消费配额限制了客户端发送和接收数据的速率（以字节 / 秒为单位）**。
* **请求配额限制了 broker 用于处理客户端请求的时间百分比。**

**可以为所有客户端（使用默认配额）、特定客户端、特定用户、或特定客户端及特定用户设置配额。**

> **特定用户的配额只在集群配置了安全特性并对客户端进行了身份验证后才有效。**

**默认的生产配额和消费配额是 broker 配置文件的一部分**。如果要限制每个生产者平均发送的消息不超过 2  MBps，那么可以在 broker 配置文件中加入：

```properties
quota.producer.default=2M
```

也可以覆盖 broker 配置文件中的默认配额来为某些客户端配置特定的配额，尽管**不建议这么做**。如果允许clientA 的配额达到 4 MBps、clientB 的配额达到 10 MBps，则可以这样配置：

```properties
quota.producer.override="clientA:4M,clientB:10M"
```

**在配置文件中指定的配额都是静态的，如果要修改它们，则需要重启所有的 broker。**因为随时都可能有新客户端加入，所以这种配置方式不是很方便。

因此，特定客户端的配额通常采用动态配置。可以用 kafka-config.sh 或 AdminClient API 来动态设置配额：

{% code overflow="wrap" %}
```properties
# 限制clientC（通过客户端ID来识别）每秒平均发送不超过1024字节。
bin/kafka-configs --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024' --entity-name clientC --entity-type clients 
​
# 限制user1（通过已认证的账号来识别）每秒平均发送不超过1024字节以及每秒平均消费不超过2048字节。
bin/kafka-configs --bootstrap-server localhost:9092 --alter --add-config 'producer_byte_rate=1024,consumer_byte_rate=2048' --entity-name user1 --entity-type users
​
# 限制所有用户每秒平均消费不超过2048字节，有覆盖配置的特定用户除外。
# 这也是动态修改默认配置的一种方式。
bin/kafka-configs --bootstrap-server localhost:9092 --alter --add-config 'consumer_byte_rate=2048' --entity-type users
```
{% endcode %}

<mark style="color:blue;">**当客户端触及配额时，broker 会开始限制客户端请求，以防止超出配额**</mark>。这意味着 broker 将延迟对客户端请求做出响应。对大多数客户端来说，这样会自动降低请求速率（因为执行中的请求数量也是有限制的），并将客户端流量降到配额允许的范围内。但是，被节流的客户端还是有可能向服务器端发送额外的请求，为了不受影响，**broker 将在一段时间内暂停与客户端之间的通信通道，以满足配额要求**。

节流行为通过 **produce-throttle-time-avg、produce-throttle-time-max、fetch-throttle-time-avg 和fetch-throttle-time-max** 暴露给客户端，这几个参数是**生产请求和消费请求因节流而被延迟的平均时间和最长时间**。需要注意的是，这些时间对应的是生产消息和消费消息的吞吐量配额、请求时间配额，或两者兼而有之。其他类型的客户端请求只会因触及请求时间配额而被节流，这些节流行为也会通过其他类似的指标暴露出来。

{% hint style="info" %}
**如果异步调用 Producer.send()，并且发送速率超过了 broker 能够接受的速率**（无论是由于配额的限制还是由于处理能力不足），那么消息将会被放入客户端的内存队列。

* <mark style="color:blue;">**如果发送速率一直快于接收速率，那么客户端最终将耗尽内存缓冲区，并阻塞后续的Producer.send() 调用。**</mark>
* <mark style="color:blue;">**如果超时延迟不足以让 broker 赶上生产者，使其清理掉一些缓冲区空间，那么 Producer.send() 最终将抛出 TimeoutException 异常。**</mark>
* <mark style="color:blue;">**或者，批次里的记录因为等待时间超过了 delivery.timeout.ms 而过期，导致执行 send() 的回调，并抛出 TimeoutException异常。**</mark>

因此，要做好计划和监控，确保 broker 的处理能力总是与生产者发送数据的速率相匹配。
{% endhint %}
