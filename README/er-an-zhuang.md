# 二、安装

### 2.1 目录说明

/opt/module：存放下载的jar包

/opt/software：应用放置于此

/opt/etc/kafka：配置、数据以及日志存放路径（只是为了方便分发所以放到了一个目录下，推荐分开放置）

/opt/etc/kafka/data：持久化的消息日志存放路径

/opt/etc/kafka/logs：运行日志，log4j日志

### 2.2 下载并解压

```
  
  cd /opt/module
  wget https://downloads.apache.org/kafka/3.5.1/kafka_2.12-3.5.1.tgz
```

```
  
  tar -xzv -f kafka_2.12-3.5.1.tgz -C /opt/software/
```

```
  
  ln -s /opt/software/kafka_2.12-3.5.1 /opt/software/kafka
```

### 2.3 配置环境变量

```
  
  sudo vim /etc/profile.d/kafka.sh
```

```
  
  KAFKA_HOME=/opt/software/kafka
  export PATH=$PATH:$KAFKA_HOME/bin
```

### 2.4 配置文件

```
  
  mkdir /opt/etc/kafka
  ​
  # 将config文件夹变成一个指向/opt/etc/kafka/config的软连接
  mv /opt/software/kafka/config /opt/etc/kafka/
  ln -s /opt/etc/kafka/config /opt/software/kafka/config
```

修改server.properies中的配置内容：

```
  
  # 集群中个各个broker必须拥有不同的id
  broker.id=3
  ​
  listeners=PLAINTEXT://0.0.0.0:9092
  advertised.listeners=PLAINTEXT://hadoop103:9092
  ​
  # 存放
  log.dirs=/opt/etc/kafka/data
  ​
  zookeeper.connect=hadoop101:2181,hadoop102:2181,hadoop103:2181/kafka
```

**要将一个broker加入到集群里，只需要修改三个配置参数：**

1. zookeeper.connect：所有broker都必须配置相同的值
2. broker.id：每一个broker都必须有一个唯一的ID
3. advertised.listeners

### 2.5 集群脚本kf.sh

```bash
  vim ~/bin/kf.sh
```

```bash
  
  #! /bin/bash
  ​
  KAFKA_HOME=/opt/software/kafka
  export PATH=$PATH:$KAFKA_HOME/bin
  ​
  KAFKA_CONFIG=/opt/etc/kafka/config/server.properties
  ​
  case $1 in
  "start"){
      for i in hadoop101 hadoop102 hadoop103
      do
          echo " --------启动 $i Kafka-------"
          ssh $i "kafka-server-start.sh -daemon $KAFKA_CONFIG"
      done
  };;
  "stop"){
      for i in hadoop101 hadoop102 hadoop103
      do
          echo " --------停止 $i Kafka-------"
          ssh $i "kafka-server-stop.sh stop"
      done
  };;
  esac
```
