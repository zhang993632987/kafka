# 影响吞吐量的属性

## <mark style="color:blue;">**fetch.min.bytes**</mark>

这个属性指定了**消费者从服务器获取记录的最小字节数，默认是1字节**。

broker在收到消费者的获取数据请求时，**如果可用数据量小于fetch.min.bytes指定的大小，那么它就会等到有足够可用数据时才将数据返回。**

## <mark style="color:blue;">**fetch.max.wait.ms**</mark>

通过设置fetch.min.bytes，可以让Kafka等到有足够多的数据时才将它们返回给消费者，feth.max.wait.ms则用于指定broker等待的时间，默认是500毫秒。**如果没有足够多的数据流入Kafka，那么消费者获取数据的请求就得不到满足，最多会导致500毫秒的延迟。**

## <mark style="color:blue;">**fetch.max.bytes**</mark>

这个属性指定了**Kafka返回的数据的最大字节数（默认为50 MB）**。

**消费者会将服务器返回的数据放在内存中，所以这个属性被用于限制消费者用来存放数据的内存大小**。需要注意的是，记录是分批发送给客户端的，如果broker要发送的批次超过了这个属性指定的大小，那么这个限制将被忽略。

## <mark style="color:blue;">**max.poll.records**</mark>

这个属性用于**控制单次调用poll()方法返回的记录条数**。

可以用它来控制应用程序在进行每一次轮询循环时需要处理的记录条数（不是记录的大小）。

## <mark style="color:blue;">**max.partition.fetch.bytes**</mark>

这个属性**指定了服务器从每个分区里返回给消费者的最大字节数（默认值是1 MB）**。

当KafkaConsumer.poll()方法返回ConsumerRecords时，从每个分区里返回的记录最多不超过max.partition.fetch.bytes指定的字节。
