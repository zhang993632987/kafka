# 从特定偏移量位置读取记录

如果你想**从分区的起始位置读取所有的消息**，或者**直接跳到分区的末尾读取新消息**，那么 Kafka API 分别提供了两个方法：

* **seekToBeginning(Collection\<Topic Partition> tp)**&#x20;
* **seekToEnd(Collection\<TopicPartition> tp)**

Kafka 还提供了用于**查找特定偏移量**的 API。下面的例子演示了如何将分区的当前偏移量定位到在指定时间点生成的记录：

```java
Long oneHourEarlier = Instant.now().atZone(ZoneId.systemDefault())
         .minusHours(1).toEpochSecond();
​
// 创建一个map，将所有分配给这个消费者的分区映射到我们想要回退到的时间戳
Map<TopicPartition, Long> partitionTimestampMap = consumer.assignment()
       .stream()
       .collect(Collectors.toMap(tp -> tp, tp -> oneHourEarlier)); 
// 通过时间戳获取对应的偏移量
// 这个方法会向broker发送请求，通过时间戳获取对应的偏移量。
Map<TopicPartition, OffsetAndTimestamp> offsetMap
       = consumer.offsetsForTimes(partitionTimestampMap); 
​
// 将每个分区的偏移量重置成上一步返回的偏移量
for(Map.Entry<TopicPartition,OffsetAndTimestamp> entry: offsetMap.entrySet()) {
   consumer.seek(entry.getKey(), entry.getValue().offset()); 
}
```
