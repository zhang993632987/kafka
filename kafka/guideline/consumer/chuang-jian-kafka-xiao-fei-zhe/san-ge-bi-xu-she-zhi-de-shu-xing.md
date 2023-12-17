# 三个必须设置的属性

## <mark style="color:blue;">**bootstrap.servers**</mark>

**bootstrap.servers指定了连接Kafka集群的字符串**。它的作用与KafkaProducer中的bootstrap.servers一样。

## <mark style="color:blue;">**key.deserializer**</mark>**和**<mark style="color:blue;">**value.deserializer**</mark>

**key.deserializer和value.deserializer**与生产者的key.serializer和value.serializer类似，只不过它们不是使用指定类把Java对象转成字节数组，而是**把字节数组转成Java对象**。

<mark style="color:red;">**生成消息所使用的序列化器与读取消息所使用的反序列化器应该是相对应的。**</mark>

{% hint style="info" %}
## <mark style="color:blue;">提示</mark>

**使用Avro和模式注册表进行序列化和反序列化的优势在于：**

* <mark style="color:blue;">**AvroSerializer可以保证写入主题的数据与主题的模式是兼容的**</mark>，也就是说，<mark style="color:blue;">**可以使用相应的反序列化器和模式来反序列化数据**</mark>。
* 另外，**不管是在生产者端还是消费者端出现的任何一个与兼容性有关的错误都会被捕捉到，而且这些错误都带有描述性信息**，这也就意味着，当出现序列化错误时，无须再费劲地调试字节数组了。
{% endhint %}
