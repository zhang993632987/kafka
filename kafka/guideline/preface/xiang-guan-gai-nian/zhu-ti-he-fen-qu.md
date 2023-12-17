# 主题和分区

## 主题

Kafka的消息通过**主题**进行分类。**主题就好比数据库的表或文件系统的文件夹。**

* <mark style="color:blue;">**主题可以被分为若干个分区，一个分区就是一个提交日志**</mark>。
* **消息会以**<mark style="color:blue;">**追加**</mark>**的方式被写入分区，然后按照先入先出的顺序读取。**

{% hint style="warning" %}
## <mark style="color:orange;">注意：</mark>

<mark style="color:red;">**由于一个主题一般包含几个分区，因此无法在整个主题范围内保证消息的顺序。**</mark>

<mark style="color:blue;">**但可以保证消息在单个分区内是有序的。**</mark>
{% endhint %}

## 分区

<mark style="color:blue;">**Kafka通过分区来实现数据的冗余和伸缩。**</mark>

* 分区可以分布在不同的服务器上，也就是说，**一个主题可以横跨多台服务器，以此来提供比单台服务器更强大的性能。**
* 此外，<mark style="color:blue;">**分区可以被复制，相同分区的多个副本可以保存在多台服务器上，以防其中一台服务器发生故障。**</mark>
