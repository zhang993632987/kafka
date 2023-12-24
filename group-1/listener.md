# Listener

## listeners

**Kafka 服务器支持监听多个端口的连接。**这是通过服务器配置中的 <mark style="color:blue;">**listeners**</mark> 属性来配置的，它接受一个用**逗号分隔**的**监听器列表**来启用。

&#x20;<mark style="color:blue;">**listeners**</mark> 中定义的每个监听器的格式如下：

```properties
{LISTENER_NAME}://{hostname}:{port}
```

**LISTENER\_NAME 通常是一个描述性的名字，定义了监听器的用途。**例如，许多配置为客户端流量使用单独的监听器，所以他们可能在配置中把相应的监听器称为 **CLIENT**:

```properties
listeners=CLIENT://localhost:9092
```

## listener.security.protocol.map

**每个监听器的安全协议在一个单独的配置中定义：**<mark style="color:blue;">**listener.security.protocol.map**</mark>。**该值是一个逗号分隔的列表，列出了映射到其安全协议的每个监听器。**例如，下面值配置指定 CLIENT 监听器将使用 SSL，而BROKER 监听器将使用明文（plaintext）：

```properties
listener.security.protocol.map=CLIENT:SSL,BROKER:PLAINTEXT
```

> ## 安全协议的可能选项包括：
>
> * <mark style="color:blue;">**PLAINTEXT**</mark>
> * <mark style="color:blue;">**SSL**</mark>
> * <mark style="color:blue;">**SASL\_PLAINTEXT**</mark>
> * <mark style="color:blue;">**SASL\_SSL**</mark>
>
> 明文（PLAINTEXT）协议不提供安全性，不需要任何额外的配置。

> **如果每个需要的监听器使用单独的安全协议，也可以在监听器中使用安全协议名称作为监听器名称。**
>
> 使用上面的例子，我们可以使用下面的定义跳过 CLIENT 和 BROKER 监听器的定义：
>
> ```properties
> listeners=SSL://localhost:9092,PLAINTEXT://localhost:9093
> ```
>
> 建议用户为监听器提供明确的名称，因为可以使每个监听器的预期用途更加清晰。

## inter.broker.listener.name

<mark style="color:blue;">**inter.broker.listener.name**</mark> 指定的**监听器名称**（listeners 列表），将**用于 broker 间通信**。broker 间通信的主要目的在于分区复制。

> **如果没有配置 inter.broker.listener.name，则broker 间通信的监听器由security.inter.broker.protocol定义的安全协议确定，默认为PLAINTEXT。**

## control.plane.listener.name

对于依赖于Zookeeper存储集群元数据的传统集群，可以声明一个独立的监听器，用于从控制器向 broker 传播元数据。该监听器由 <mark style="color:blue;">**control.plane.listener.name**</mark> 定义。当控制器需要向集群中的 broker 推送元数据更新时，它将使用此监听器。

使用**控制平面监听器**的好处在于它使用一个独立的处理线程，这使得应用程序流量不太可能妨碍元数据更改的及时传播（例如，分区领导者和ISR更新）。

> control.plane.listener.name 的默认值为null，这意味着控制器将使用由 **inter.broker.listener** 定义的监听器传播元数据。

## KRaft 集群

> 在KRaft集群中，broker 是任何在 process.roles 中启用了 broker 角色的服务器，而控制器是任何在process.roles 中启用了 controller 角色的服务器。监听器的配置取决于角色。
>
> **由 inter.broker.listener.name 定义的监听器专门用于 broker 之间的请求**。另一方面，**控制器必须使用由 controller.listener.names 配置定义的单独监听器**。这不能被设置为与 broker 之间的监听器相同的值。

### 单一角色

控制器既会接收来自其他控制器的请求，也会接收来自 broker 的请求。因此，即使一个服务器没有启用控制器角色（即它只是一个broker），它仍然必须定义控制器监听器，以及配置它所需的安全属性。例如，在独立的broker上，我们可能会使用以下配置：

```properties
process.roles=broker
listeners=BROKER://localhost:9092
inter.broker.listener.name=BROKER
controller.quorum.voters=0@localhost:9093
controller.listener.names=CONTROLLER
listener.security.protocol.map=BROKER:SASL_SSL,CONTROLLER:SASL_SSL
```

在这个例子中，即使 broker 本身不暴露控制器监听器，控制器监听器仍然配置为使用 SASL\_SSL 安全协议。**在这种情况下，将使用 controller.quorum.voters 配置中定义的控制器完整列表中的端口。**

### 双重角色

对于启用了 **broker** 和 **controller** 角色的 KRaft 服务器，配置是相似的，唯一的区别是**控制器监听器必须包含在 listeners 中：**

```properties
process.roles=broker,controller
listeners=BROKER://localhost:9092,CONTROLLER://localhost:9093
inter.broker.listener.name=BROKER
controller.quorum.voters=0@localhost:9093
controller.listener.names=CONTROLLER
listener.security.protocol.map=BROKER:SASL_SSL,CONTROLLER:SASL_SSL
```

**在 controller.quorum.voters 定义的端口必须与公开的控制器监听器的端口完全匹配**。例如，在这里，CONTROLLER 监听器绑定到端口9093。然后，controller.quorum.voters 定义的连接字符串也必须使用端口9093。

控制器将接受在 controller.listener.names 中定义的所有监听器上的请求。通常只会有一个控制器监听器，但也可以有多个。例如，这提供了一种通过集群的滚动（一个滚动用于暴露新的监听器，另一个滚动用于移除旧的监听器）来将活动监听器从一个端口或安全协议更改为另一个的方法。当定义了多个控制器监听器时，列表中的第一个监听器将用于出站请求。

**在Kafka中，通常会为客户端使用单独的监听器。这允许在网络级别隔离集群间的监听器。在KRaft中，对于控制器监听器，由于客户端不会与其交互，因此该监听器应该被隔离。**
