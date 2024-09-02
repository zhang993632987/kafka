# 如何使用事务

> **使用事务的最常见也最推荐的方式是在 Streams 中启用精确一次性保证。**无须直接管理事务，Streams 会自动提供我们需要的保证。事务最初就是为这个场景而设计的，所以在 Streams 中启用事务是最简单也最有可能符合我们预期的方式。

如果想在不使用Streams的情况下获得精确一次性保证，可以直接使用事务API。

{% code overflow="wrap" fullWidth="false" %}
```java
Properties producerProps = new Properties();
producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
producerProps.put(ProducerConfig.CLIENT_ID_CONFIG, "DemoProducer");
// 为生产者配置transactional.id，让它成为一个能够进行原子多分区写入的事务性生产者。
// 事务ID必须是唯一且长期存在的，因为本质上就是用它定义了应用程序的一个实例。
producerProps.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, transactionalId);

producer = new KafkaProducer<>(producerProps);


Properties consumerProps = new Properties();
consumerProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
consumerProps.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
// 消费者不提交自己的偏移量——生产者会将偏移量提交作为事务的一部分，所以需要禁用自动提交。
consumerProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
// 为了干净地读取事务（忽略执行中和已中止的事务），可以将消费者隔离级别设置为read_committed。
consumerProps.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");

consumer = new KafkaConsumer<>(consumerProps);


// 事务性生产者要做的第一件事是初始化，包括注册事务ID和增加epoch的值（确保其他具有相同ID的生产者将被视为“僵尸”，并中止具有相同事务ID的旧事务）。
producer.initTransactions();

consumer.subscribe(Collections.singleton(inputTopic));

while (true) {
    try {
        ConsumerRecords<Integer, String> records = consumer.poll(Duration.ofMillis(200));
        if (records.count() > 0) {
            // 开始事务
            producer.beginTransaction();
            for (ConsumerRecord<Integer, String> record : records) {
                ProducerRecord<Integer, String> customizedRecord = transform(record);
                producer.send(customizedRecord);
            }
            Map<TopicPartition, OffsetAndMetadata> offsets = consumerOffsets();
            // 需要将偏移量提交作为事务的一部分，
            // 这样可以保证如果生成结果失败，则未成功处理的消息的偏移量将不会被提交
            producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
            // 提交事务
            producer.commitTransaction();
        }
    } catch (ProducerFencedException | InvalidProducerEpochException e) {
        // 如果遇到这个异常，则说明应用程序实例变成“僵尸”了
        throw new KafkaException(String.format(
                "The transactional.id %s is used by another process", transactionalId));
    } catch (KafkaException e) {
        // 如果在提交事务时遇到错误，则可以中止事务，重置消费者偏移量位置，并进行重试。
        producer.abortTransaction();
        resetToLastCommittedPositions(consumer);
    }
}
```
{% endcode %}
