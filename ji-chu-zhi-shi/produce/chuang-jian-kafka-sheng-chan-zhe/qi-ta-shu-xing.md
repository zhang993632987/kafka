# 其他属性

## <mark style="color:blue;">**client.id**</mark>

client.id 是**客户端标识符**，它的值可以是任意字符串，broker 用它来识别从客户端发送过来的消息。

> client.id 可以被用在日志、指标和配额中。

## <mark style="color:blue;">**acks**</mark>

acks 指定了**生产者在多少个同步分区副本收到消息的情况下才会认为消息写入成功**。

* 如果 <mark style="color:blue;">**acks=0**</mark>，则**生产者不会等待任何来自 broker 的响应**。
  * 也就是说，如果 broker 因为某些问题没有收到消息，那么生产者便无从得知，消息也就丢失了。
  * 不过，因为生产者不需要等待 broker 返回响应，所以它们能够以网络可支持的最大速度发送消息，从而达到很高的吞吐量。
* 如果 <mark style="color:blue;">**acks=1**</mark>，那么**只要集群的首领副本收到消息，生产者就会收到消息成功写入的响应**。
  * 如果消息无法到达首领副本（比如首领副本发生崩溃，新首领还未选举出来），那么生产者会收到一个错误响应。为了避免数据丢失，生产者会尝试重发消息。
  * 不过，在首领副本发生崩溃的情况下，如果消息还没有被复制到新的首领副本，则消息还是有可能丢失。
* 如果 <mark style="color:blue;">**acks=all**</mark>，那么**只有当所有同步副本全部收到消息时，生产者才会收到消息成功写入的响应**。这种模式是最安全的，它可以保证不止一个 broker 收到消息，就算有个别 broker 发生崩溃，整个集群仍然可以运行。

{% hint style="info" %}
## <mark style="color:blue;">提示</mark>

为 acks 设置的值越小，生产者发送消息的速度就越快。也就是说，我们通过牺牲可靠性来换取较低的生产者延迟。

不过，<mark style="color:blue;">**端到端延迟是指从消息生成到可供消费者读取的时间，这对 3 种配置来说都是一样的**</mark><mark style="color:blue;">。</mark>这是因为为了保持一致性，<mark style="color:blue;">**在消息被写入所有同步副本之前，Kafka 不允许消费者读取它们**</mark><mark style="color:blue;">。</mark>

因此，<mark style="color:blue;">**如果关心的是端到端延迟，而不是生产者延迟，那么就不需要在可靠性和低延迟之间做权衡了：可以选择最可靠的配置，但仍然可以获得相同的端到端延迟。**</mark>
{% endhint %}

## <mark style="color:blue;">**batch.size**</mark>

当有多条消息被发送给同一个分区时，生产者会把它们放在同一个批次里。这个参数指定了**一个批次可以使用的内存大小**。

需要注意的是，**该参数是按照字节数而不是消息条数来计算的**。当批次被填满时，批次里所有的消息都将被发送出去。

## <mark style="color:blue;">**max.in.flight.requests.per.connection**</mark>

这个参数指定了**生产者在收到服务器响应之前可以发送多少个消息批次**。它的值越大，占用的内存就越多，不过吞吐量也会得到提升。

Apache wiki 页面上的实验数据表明，在单数据中心环境中，该参数被设置为 2 时可以获得最佳的吞吐量，但使用**默认值 5** 也可以获得差不多的性能。

> <mark style="color:purple;">**假设我们把 retries 设置为非零的整数，并把 max.in.flight.requests.per.connection 设置为比 1大的数。**</mark>
>
> **如果第一个批次写入失败，第二个批次写入成功，那么 broker 会重试写入第一个批次，等到第一个批次也写入成功，两个批次的顺序就反过来了。**
>
> 我们希望至少有 2 个正在处理中的请求（出于性能方面的考虑），并且可以进行多次重试（出于可靠性方面的考虑），这个时候，最好的解决方案是将 **enable.idempotence** 设置为 **true**。这样就可以在最多有 5 个正在处理中的请求的情况下保证顺序，并且可以保证重试不会引入重复消息

## <mark style="color:blue;">**enable.idempotence**</mark>

从 0.11 版本开始，<mark style="color:blue;">**Kafka 支持精确一次性(exactly once)语义**</mark>。

> **假设为了最大限度地提升可靠性，你将生产者的 acks 设置为all，并将 delivery.timeout.ms 设置为一个比较大的数，允许进行尽可能多的重试。这些配置可以确保每条消息被写入 Kafka 至少一次。**

**但在某些情况下，消息有可能被写入 Kafka 不止一次。**

**假设一个 broker 收到了生产者发送的消息，然后消息被写入本地磁盘并成功复制给了其他 broker。此时，这个 broker 还没有向生产者发送响应就发生了崩溃。**

**而生产者将一直等待，直到达到 request.timeout.ms，然后进行重试。重试发送的消息将被发送给新的首领，而这个首领已经有这条消息的副本，因为之前写入的消息已经被成功复制给它了。**现在，你就有了一条重复的消息。

<mark style="color:blue;">**为了避免这种情况，可以将 enable.idempotence 设置为 true。**</mark>**当幂等生产者被启用时，生产者将给发送的每一条消息都加上一个序列号。如果 broker 收到具有相同序列号的消息，那么它就会拒绝第二个副本，而生产者则会收到 DuplicateSequenceException**，这个异常对生产者来说是无害的。

<mark style="color:red;">**如果要启用幂等性，那么 max.in.flight.requests.per.connection 应小于或等于 5、retries 应大于 0，并且 acks 被设置为 all**</mark>**。**如果设置了不恰当的值，则会抛出 ConfigException 异常。

## <mark style="color:blue;">**buffer.memory**</mark>

这个参数用来**设置生产者用来暂存消息的内存缓冲区大小**。

如果应用程序调用 send() 方法的速度超过生产者将消息发送给服务器的速度，那么生产者的缓冲空间可能会被耗尽，后续的 send() 方法调用会等待内存空间被释放，如果在 **max.block.ms** 之后还没有可用空间，就抛出异常。

> 需要注意的是，这个异常与其他异常不一样，它是 send() 方法而不是 Future 对象抛出来的。

## <mark style="color:blue;">**compression.type**</mark>

在默认情况下，生产者发送的消息是未经压缩的。这个参数可以被设置为 snappy、gzip、lz4 或 zstd，这指定了**消息被发送给 broker 之前使用哪一种压缩算法**。

## <mark style="color:blue;">**max.request.size**</mark>

这个参数用于**控制生产者发送的请求的大小**。它限制了可发送的**单条最大消息**的大小和**单个请求的消息总量**的大小。

假设这个参数的值为 1 MB，那么可发送的单条最大消息就是 1 MB，或者生产者最多可以在单个请求里发送一条包含 1024 个大小为 1 KB的消息。

{% hint style="warning" %}
## <mark style="color:orange;">注意</mark>

**broker 对可接收的最大消息也有限制(message.max.bytes)，其两边的配置最好是匹配的，以免生产者发送的消息被 broker 拒绝。**
{% endhint %}

## <mark style="color:blue;">**receive.buffer.bytes**</mark>** 和 **<mark style="color:blue;">**send.buffer.bytes**</mark>

这两个参数**分别指定了 TCP socket 接收和发送数据包的缓冲区大小**。

* 如果它们被设为 –1，就使用操作系统默认值。
* 如果生产者或消费者与 broker 位于不同的数据中心，则可以适当加大它们的值，因为跨数据中心网络的延迟一般都比较高，而带宽又比较低。
