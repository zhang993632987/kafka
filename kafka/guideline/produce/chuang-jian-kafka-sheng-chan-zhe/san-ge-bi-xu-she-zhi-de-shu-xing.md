# 三个必须设置的属性

## <mark style="color:blue;">**bootstrap.servers**</mark>

**broker的地址**。可以由多个host:port组成，生产者用它们来建立初始的Kafka集群连接。

它不需要包含所有的broker地址，因为生产者在建立初始连接之后可以从给定的broker那里找到其他broker的信息。不过还是<mark style="color:orange;">**建议至少提供两个broker地址**</mark>，因为**一旦其中一个停机，则生产者仍然可以连接到集群**。

## <mark style="color:blue;">**key.serializer**</mark>

**一个类名，用来序列化消息的键。**

broker希望接收到的消息的键和值都是字节数组。生产者可以把任意Java对象作为键和值发送给broker，但它需要知道如何把这些Java对象转换成字节数组。

{% hint style="info" %}
## <mark style="color:orange;">注意：</mark>

<mark style="color:orange;">**必须设置key.serializer这个属性，尽管你可能只需要将值发送给Kafka。**</mark>

<mark style="color:blue;">**如果只需要发送值，则可以将Void作为键的类型，然后将这个属性设置为VoidSerializer。**</mark>
{% endhint %}

## <mark style="color:blue;">**value.serializer**</mark>

**一个类名，用来序列化消息的值。**

与设置key.serializer属性一样，需要将value.serializer设置成可以序列化消息值对象的类。
