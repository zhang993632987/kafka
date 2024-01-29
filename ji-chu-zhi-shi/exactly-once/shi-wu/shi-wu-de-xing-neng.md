# 事务的性能

事务给**生产者**带来了一些额外的开销：

* 事务ID注册在生产者生命周期中只会发生一次。
* 分区事务注册最多会在每个分区加入每个事务时发生一次，然后每个事务会发送一个提交请求，并向每个分区写入一个额外的提交标记。
* 事务初始化和事务提交请求都是同步的，在它们成功、失败或超时之前不会发送其他数据，这进一步增加了开销。

需要注意的是，**生产者在事务方面的开销与事务包含的消息数量无关**。因此，<mark style="color:orange;">**一个事务包含的消息越多，相对开销就越小，同步调用次数也就越少，从而提高了总体吞吐量。**</mark>

在**消费者**方面，读取提交标记会增加一些开销。<mark style="color:blue;">**事务对消费者的性能影响主要是在read\_committed隔离级别下的消费者无法读取未提交事务所包含的记录。提交事务的时间间隔越长，消费者在读取到消息之前需要等待的时间就越长，端到端延迟也就越高。**</mark>