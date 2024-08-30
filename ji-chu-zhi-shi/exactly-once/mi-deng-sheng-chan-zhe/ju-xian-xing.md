# 局限性

* <mark style="color:blue;">**幂等生产者只能防止由生产者内部重试逻辑引起的消息重复**</mark>**，不管这种重试是由生产者、网络还是 broker 错误所导致。**<mark style="color:orange;">**对于使用同一条消息调用两次 producer.send() 就会导致消息重复的情况，即使使用幂等生产者也无法避免。**</mark>
* <mark style="color:orange;">**如果两个生产者尝试发送同样的消息，则幂等生产者将无法检测到消息重复。**</mark>
