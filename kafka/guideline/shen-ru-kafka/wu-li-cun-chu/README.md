# 物理存储

<mark style="color:blue;">**Kafka的基本存储单元是分区**</mark>**。分区既无法在多个broker间再细分，也无法在同一个broker的多个磁盘间再细分。所以，分区的大小受单个挂载点可用空间的限制。**

在配置Kafka时，管理员会指定一个用于<mark style="color:blue;">**保存分区数据的目录列表**</mark>，也就是<mark style="color:blue;">**log.dirs**</mark>参数。这个参数一般会包含Kafka将要使用的每一个挂载点的目录。



##

##

## 5.3.5&#x20;
