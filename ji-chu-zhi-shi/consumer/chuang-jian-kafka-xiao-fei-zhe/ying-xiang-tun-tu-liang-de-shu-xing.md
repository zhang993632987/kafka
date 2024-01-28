# 影响吞吐量的属性

## <mark style="color:blue;">**fetch.min.bytes**</mark>

这个属性指定了**消费者从服务器获取记录的最小字节数，默认是 1 字节**。

broker 在收到消费者的获取数据请求时，**如果可用数据量小于 fetch.min.bytes 指定的大小，那么它就会等到有足够可用数据时才将数据返回。**

## <mark style="color:blue;">**fetch.max.wait.ms**</mark>

通过设置 fetch.min.bytes，可以让 Kafka 等到有足够多的数据时才将它们返回给消费者，feth.max.wait.ms 则用于指定 broker 等待的时间，默认是 500 毫秒。**如果没有足够多的数据流入 Kafka，那么消费者获取数据的请求就得不到满足，最多会导致 500 毫秒的延迟。**

## <mark style="color:blue;">**fetch.max.bytes**</mark>

这个属性指定了 **Kafka 返回的数据的最大字节数（默认为 50 MB）**。**消费者会将服务器返回的数据放在内存中，所以这个属性被用于限制消费者用来存放数据的内存大小**。

> 需要注意的是，记录是分批发送给客户端的，如果 broker 要发送的批次超过了这个属性指定的大小，那么这个限制将被忽略。

## <mark style="color:blue;">**max.poll.records**</mark>

这个属性用于**控制单次调用 poll() 方法返回的记录条数**。

可以用它来控制应用程序在进行每一次轮询循环时需要处理的**记录条数**（不是记录的大小）。

## <mark style="color:blue;">**max.partition.fetch.bytes**</mark>

这个属性**指定了服务器从每个分区里返回给消费者的最大字节数（默认值是 1 MB）**。

当 KafkaConsumer.poll() 方法返回 ConsumerRecords 时，从每个分区里返回的记录最多不超过max.partition.fetch.bytes 指定的字节。
