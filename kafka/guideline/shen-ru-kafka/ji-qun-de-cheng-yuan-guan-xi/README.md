# 集群的成员关系

## <mark style="color:blue;">**Kafka使用ZooKeeper维护集群的成员信息**</mark>

* 每个broker都有一个唯一的标识符，这个标识符既可以在配置文件中指定，也可以自动生成。**broker在启动时通过创建ZooKeeper 临时节点把自己的ID注册到ZooKeeper中**。
* broker、控制器和其他的一些生态系统工具会订阅ZooKeeper的 <mark style="color:blue;">**/brokers/ids**</mark> 路径（也就是broker在ZooKeeper上的注册路径），当有broker加入或退出集群时，它们可以收到通知。
* 当broker与ZooKeeper断开连接时，它在启动时创建的临时节点会自动从ZooKeeper上移除。监听broker节点路径的Kafka组件会被告知这个broker已被移除。

broker对应的ZooKeeper节点会在broker被关闭之后消失，但它的ID会**继续存在于其他数据结构**中。例如，每个主题的副本集中就可能包含这个ID。**在完全关闭一个broker后，如果使用相同的ID启动另一个全新的broker，则它会立即加入集群，并获得与之前相同的分区和主题。**
