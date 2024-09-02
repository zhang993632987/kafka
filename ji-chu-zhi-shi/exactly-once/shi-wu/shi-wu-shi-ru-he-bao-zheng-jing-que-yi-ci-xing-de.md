# 事务是如何保证精确一次性的

## 原子多分区写入

<mark style="color:orange;">**精确一次处理意味着消费、处理和生产都是原子操作，要么提交偏移量和生成结果这两个操作都成功，要么都不成功。我们要确保不会出现只有部分操作执行成功的情况（提交了偏移量但没有生成结果，反之亦然）。**</mark>

为了支持这种行为，Kafka 事务引入了<mark style="color:blue;">**原子多分区写入**</mark>的概念。**提交偏移量和生成结果都涉及向分区写入数据，结果会被写入输出主题，偏移量会被写入 consumer\_offsets 主题**。如果可以**打开一个事务，向这两个主题写入消息，如果两个写入操作都成功就提交事务，如果不成功就中止，并进行重试，那么就会实现我们所追求的精确一次性语义**。

<figure><img src="../../../.gitbook/assets/原子多分区写入.jpg" alt=""><figcaption></figcaption></figure>

## 事务性生产者

为了启用事务和执行原子多分区写入，我们使用了<mark style="color:blue;">**事务性生产者**</mark>。

{% hint style="info" %}
<mark style="color:blue;">**事务性生产者实际上就是一个配置了 transactional.id 并用 initTransactions() 方法初始化的 Kafka 生产者。**</mark>
{% endhint %}

* **与 producer.id（由 broker 自动生成）不同，transactional.id 是一个生产者配置参数，在生产者重启之后仍然存在**。实际上，transactional.id 主要用于在重启之后识别同一个生产者。
* broker 维护了 transactional.id 和 producer.id 之间的映射关系，如果对一个已有的 transactional.id 再次调用 initTransactions() 方法，则生产者将分配到与之前一样的 producer.id，而不是一个新的随机数。

## “僵尸”隔离机制

防止“僵尸”应用程序实例重复生成结果需要一种<mark style="color:blue;">**“僵尸”隔离机制**</mark>，或者防止“僵尸”实例将结果写入输出流。通常可以使用 **epoch** 来隔离“僵尸”。**在调用 initTransaction() 方法初始化事务性生产者时，Kafka 会增加与 transactional.id 相关的 epoch。**

{% hint style="info" %}
<mark style="color:blue;">**带有相同 transactional.id 但 epoch 较小的发送请求、提交请求和中止请求将被拒绝**</mark>，并返回 FencedProducer 错误。旧生产者将无法写入输出流，并被强制 close()，以防止“僵尸”引入重复记录。
{% endhint %}

> Kafka 2.5 及以上版本支持将消费者群组元数据添加到事务元数据中。这些元数据也被用于隔离“僵尸”，在对“僵尸”实例进行隔离的同时允许带有不同事务 ID 的生产者写入相同的分区。

## 消费者隔离级别

{% hint style="info" %}
在很大程度上，<mark style="color:blue;">**事务是一个生产者特性**</mark>。创建事务性生产者、开始事务、将记录写入多个分区、生成偏移量并提交或中止事务，这些都是由生产者完成的。
{% endhint %}

<mark style="color:blue;">**以事务方式写入的记录，即使是最终被中止的部分，也会像其他记录一样被写入分区。消费者也需要配置正确的隔离级别，否则将无法获得我们想要的精确一次性保证。**</mark>

我们通过设置 <mark style="color:blue;">**isolation.level**</mark> 参数来控制消费者如何读取以事务方式写入的消息：

* **如果设置为 **<mark style="color:blue;">**read\_committed**</mark>**，那么调用 consumer.poll() 将返回属于**<mark style="color:blue;">**已成功提交的事务**</mark>**或**<mark style="color:blue;">**以非事务方式**</mark>**写入的消息，它不会返回属于已中止或执行中的事务的消息**。
* **默认的隔离级别是** <mark style="color:blue;">**read\_uncommitted**</mark>，它**将返回所有记录，包括属于执行中或已中止的事务的记录。**

为了保证按顺序读取消息，read\_committed 隔离级别将不返回在事务开始之后（这个位置也被叫作最后稳定偏移量，last stable oﬀset，LSO）生成的消息。这些消息将被保留，直到事务被生产者提交或终止，或者事务超时（通过 transaction.timeout.ms 参数指定，默认为 15 分钟）并被 broker 终止。<mark style="color:orange;">**长时间使事务处于打开状态会导致消费者延迟，从而导致更高的端到端延迟。**</mark>
