# 再均衡监听器

**消费者 API 提供了一些方法，让你可以在消费者分配到新分区或旧分区被移除时执行一些代码逻辑**。你所要做的就是在调用 <mark style="color:blue;">**subscribe()**</mark> 方法时传进去一个 <mark style="color:blue;">**ConsumerRebalanceListener**</mark> 对象。

**ConsumerRebalanceListener** 有 3 个需要实现的方法：

* **public void onPartitionsAssigned(Collection partitions)：这个方法会在**<mark style="color:blue;">**重新分配分区之后**</mark>**以及**<mark style="color:blue;">**消费者开始读取消息之前**</mark>**被调用。**
  * 可以在这个方法中**准备或加载与分区相关的状态信息**、**找到正确的偏移量**，等等。
  * <mark style="color:orange;">**这里所有的事情都应该保证在 max.poll.timeout.ms 内完成，以便消费者可以成功地加入群组**</mark>**。**
*   **public void onPartitionsRevoked(Collection partitions)：这个方法会在**<mark style="color:blue;">**消费者放弃对分区的所有权时**</mark>**调用**——可能是因为发生了**再均衡**或者**消费者正在被关闭**。

    * **如果使用了主动再均衡算法，那么这个方法会在再均衡开始之前以及消费者停止读取消息之后调用**。
    * **如果使用了协作再均衡算法，那么这个方法会在再均衡结束时调用，而且只涉及消费者放弃所有权的那些分区**。

    <mark style="color:orange;">**如果你要提交偏移量，那么可以在这里提交**</mark>，无论是哪个消费者接管这个分区，它都知道应该从哪里开始读取消息。
* **public void onPartitionsLost(Collectionpartitions)：这个方法**<mark style="color:blue;">**只会在使用了协作再均衡算法并且之前不是通过再均衡获得的分区被重新分配给其他消费者**</mark>**时调用**（**之前通过再均衡获得的分区被重新分配时会调用 onPartitionsRevoked()**）。
  * 你可以在这里**清除与这些分区相关的状态或资源**。需要注意的是，在清理状态时要非常小心，因为分区的新所有者可能也保存了分区状态，需要避免发生冲突。
  * 如果你没有实现这个方法，则 onPartitionsRevoked() 将被调用。

{% hint style="warning" %}
## <mark style="color:orange;">注意</mark>

如果使用了协作再均衡算法，那么需要注意以下几点：

* **onPartitionsAssigned()** 在<mark style="color:blue;">**每次进行再均衡**</mark>时都会被调用，以此来告诉消费者发生了再均衡。如果没有新的分区分配给消费者，那么它的参数就是一个空集合。
* **onPartitionsRevoked()** 会在进行<mark style="color:blue;">**正常的再均衡**</mark>并且<mark style="color:blue;">**有消费者放弃分区所有权**</mark>时被调用。如果它被调用，那么参数就不会是空集合。
* **onPartitionsLost()** 会在进行<mark style="color:blue;">**意外的再均衡**</mark>并且<mark style="color:blue;">**参数集合中的分区已经有新的所有者**</mark>的情况下被调用。

<mark style="color:blue;">**如果这 3 个方法你都实现了，那么就可以保证在一个正常的再均衡过程中，分区新所有者的 onPartitionsAssigned() 会在之前的所有者的 onPartitionsRevoked() 被调用完毕并放弃了所有权之后被调用。**</mark>
{% endhint %}

<details>

<summary><mark style="color:purple;">示例</mark></summary>

```java
 private Map<TopicPartition, OffsetAndMetadata> currentOffsets =
     new HashMap<>();
 Duration timeout = Duration.ofMillis(100);
 ​
 private class HandleRebalance implements ConsumerRebalanceListener { 
     public void onPartitionsAssigned(Collection<TopicPartition>
         partitions) { 
     }
 ​
     // 如果发生了再均衡，则要在即将失去分区所有权时提交偏移量
     // 提交的是所有分区而不只是那些即将失去所有权的分区的偏移量
     public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
         System.out.println("Lost partitions in rebalance. " +
             "Committing current offsets:" + currentOffsets);
         consumer.commitSync(currentOffsets); 
     }
 }
 ​
 try {
     // 把ConsumerRebalanceListener对象传给subscribe()方法，这样消费者才能调用它
     consumer.subscribe(topics, new HandleRebalance()); 
 ​
     while (true) {
         ConsumerRecords<String, String> records = consumer.poll(timeout);
         for (ConsumerRecord<String, String> record : records) {
             System.out.printf("topic = %s, partition = %s, offset = %d,
                 customer = %s, country = %s\n",
                 record.topic(), record.partition(), record.offset(),
                 record.key(), record.value());
             currentOffsets.put(
                 new TopicPartition(record.topic(), record.partition()),
                 new OffsetAndMetadata(record.offset()+1, null));
         }
         consumer.commitAsync(currentOffsets, null);
     }
 } catch (WakeupException e) {
     // 忽略异常
 } catch (Exception e) {
     log.error("Unexpected error", e);
 } finally {
     try {
         consumer.commitSync(currentOffsets);
     } finally {
         consumer.close();
         System.out.println("Closed consumer and we are done");
     }
 }
```

</details>
