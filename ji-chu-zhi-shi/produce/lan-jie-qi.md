# 拦截器

有时候，你希望**在不修改代码的情况下改变 Kafka 客户端的行为**。这或许是因为你想给公司所有的应用程序都加上同样的行为，或许是因为无法访问应用程序的原始代码。

> <mark style="color:blue;">**常见的生产者拦截器应用场景包括：捕获监控和跟踪信息、为消息添加标头，以及敏感信息脱敏。**</mark>

## ProducerInterceptor&#x20;

Kafka 的 ProducerInterceptor 拦截器包含两个关键方法：

* **ProducerRecord\<K, V> onSend(ProducerRecord\<K, V> record)**：**这个方法会在记录被发送给Kafka之前，甚至是在记录被序列化之前调用。**
  * 如果覆盖了这个方法，那么就可以捕获到有关记录的信息，甚至可以修改它。只需确保这个方法返回一个有效的 ProducerRecord 对象。
  * 这个方法返回的记录将被序列化并发送给 Kafka。
* **void onAcknowledgement(RecordMetadata metadata, Exception exception)**：这个方法会<mark style="color:blue;">**在收到Kafka 的确认响应时调用**</mark>。覆盖这个方法时，不可以修改 Kafka 返回的响应，但可以捕获到有关响应的信息。

## 示例

可以在不修改客户端代码的情况下使用生产者拦截器。**如果要在 kafka-console-producer 中使用上述的拦截器，那么可以遵循以下 3 个步骤：**

1.  将 jar 包加入类路径中。

    ```bash
    export CLASSPATH=$CLASSPATH:~./target/CountProducerInterceptor-1.0-
    SNAPSHOT.jar
    ```
2.  创建一个包含这些信息的配置文件 producer.config。

    ```properties
    interceptor.classes=com.shapira.examples.interceptors.CountProducer
    Interceptor counting.interceptor.window.size.ms=10000
    ```
3.  正常运行应用程序，但要指定上一步创建的配置文件。

    ```properties
    bin/kafka-console-producer.sh --broker-list localhost:9092 --topic
    interceptor-test --producer.config producer.config
    ```
