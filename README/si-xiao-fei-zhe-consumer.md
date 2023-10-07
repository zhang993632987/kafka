---
layout:
  title:
    visible: true
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# 四、消费者（Consumer）

### 4.1 相关概念

#### 4.1.1 消费者和消费者群组

**Kafka消费者从属于消费者群组。一个群组里的消费者订阅的是同一个主题，每个消费者负责读取这个主题的部分消息。**

**向群组里添加消费者是横向扩展数据处理能力的主要方式**。Kafka消费者经常需要执行一些高延迟的操作，比如把数据写到数据库或用数据做一些比较耗时的计算。在这些情况下，单个消费者无法跟上数据生成的速度，因此可以增加更多的消费者来分担负载，让每个消费者只处理部分分区的消息，这是横向扩展消费者的主要方式。于是，我们可以为主题创建大量的分区，当负载急剧增长时，可以加入更多的消费者。不过需要注意的是，**不要让消费者的数量超过主题分区的数量，因为多余的消费者只会被闲置**。

不同于传统的消息系统，横向伸缩消费者和消费者群组并不会导致Kafka性能下降。

#### 4.1.2 消费者群组和分区再均衡

**消费者群组里的消费者共享主题分区的所有权**。当一个新消费者加入群组时，它将开始读取一部分原本由其他消费者读取的消息。当一个消费者被关闭或发生崩溃时，它将离开群组，原本由它读取的分区将由群组里的其他消费者读取。主题发生变化（比如管理员添加了新分区）会导致分区重分配。

**分区的所有权从一个消费者转移到另一个消费者的行为称为再均衡**。再均衡非常重要，它为消费者群组带来了高可用性和伸缩性（你可以放心地添加或移除消费者）。不过，在正常情况下，我们并不希望发生再均衡。

> 根据消费者群组所使用的分区分配策略的不同，再均衡可以分为两种类型：
>
> **主动再均衡**：在进行主动再均衡期间，所有消费者都会停止读取消息，放弃分区所有权，重新加入消费者群组，并获得重新分配到的分区。这样会导致整个消费者群组在一个很短的时间窗口内不可用。这个时间窗口的长短取决于消费者群组的大小和几个配置参数。主动再均衡包含两个不同的阶段：第一个阶段，所有消费者都放弃分区所有权；第二个阶段，消费者重新加入群组，获得重新分配到的分区，并继续读取消息。
>
> **协作再均衡**：协作再均衡（也称为增量再均衡）通常是指将一个消费者的部分分区重新分配给另一个消费者，其他消费者则继续读取没有被重新分配的分区。**这种再均衡包含两个或多个阶段**。在第一个阶段，消费者群组首领会通知所有消费者，它们将失去部分分区的所有权，然后消费者会停止读取这些分区，并放弃对它们的所有权。在第二个阶段，消费者群组首领会将这些没有所有权的分区分配给其他消费者。**虽然这种增量再均衡可能需要进行几次迭代，直到达到稳定状态，但它避免了主动再均衡中出现的“停止世界”停顿**。这对大型消费者群组来说尤为重要，因为它们的再均衡可能需要很长时间。

消费者会向被指定为**群组协调器**的broker（不同消费者群组的协调器可能不同）发送心跳，以此来保持群组成员关系和对分区的所有权关系。**心跳是由消费者的一个后台线程发送的，只要消费者能够以正常的时间间隔发送心跳，它就会被认为还“活着”**。如果消费者在足够长的一段时间内没有发送心跳，那么它的会话就将超时，群组协调器会认为它已经“死亡”，进而触发再均衡。

#### 4.1.3 群组固定成员

在默认情况下，消费者的群组成员身份标识是临时的。当一个消费者离开群组时，分配给它的分区所有权将被撤销；当该消费者重新加入时，将通过再均衡协议为其分配一个新的成员ID和新分区。

可以给消费者分配一个唯一的**group.instance.id**，让它成为**群组的固定成员**。通常，当消费者第一次以固定成员身份加入群组时，群组协调器会按照分区分配策略给它分配一部分分区。**当这个消费者被关闭时，它不会自动离开群组——它仍然是群组的成员，直到会话超时（session.timeout.ms）**。**当这个消费者重新加入群组时，它会继续持有之前的身份，并分配到之前所持有的分区**。群组协调器缓存了每个成员的分区分配信息，只需要将缓存中的信息发送给重新加入的固定成员，**不需要进行再均衡**。

### 4.2 创建Kafka消费者

在读取消息之前，需要先创建一个KafkaConsumer对象。

#### 4.2.1 三个必须设置的属性

**1. bootstrap.servers**

bootstrap.servers指定了连接Kafka集群的字符串。它的作用与KafkaProducer中的bootstrap.servers一样。

**2. key.deserializer和value.deserializer**

key.deserializer和value.deserializer与生产者的key.serializer和value.serializer类似，只不过它们不是使用指定类把Java对象转成字节数组，而是把字节数组转成Java对象。

生成消息所使用的序列化器与读取消息所使用的反序列化器应该是相对应的。

使用Avro和模式注册表进行序列化和反序列化的优势在于：**AvroSerializer可以保证写入主题的数据与主题的模式是兼容的**，也就是说，**可以使用相应的反序列化器和模式来反序列化数据**。另外，不管是在生产者端还是消费者端出现的任何一个与兼容性有关的错误都会被捕捉到，而且这些错误都带有描述性信息，这也就意味着，当出现序列化错误时，无须再费劲地调试字节数组了。

#### 4.2.2 会话

**1. session.timeout.ms和heartbeat.interval.ms**

session.timeout.ms指定了消费者可以在多长时间内不与服务器发生交互而仍然被认为还“活着”，默认是10秒。**如果消费者没有在session.timeout.ms指定的时间内发送心跳给群组协调器，则会被认为已“死亡”**，协调器就会触发再均衡，把分区分配给群组里的其他消费者。session.timeout.ms与heartbeat.interval.ms紧密相关。**heartbeat.interval.ms指定了消费者向协调器发送心跳的频率**，session.timeout.ms指定了消费者可以多久不发送心跳。因此，我们一般会同时设置这两个属性，heartbeat.interval.ms必须比session.timeout.ms小，通常前者是后者的1/3。如果session.timeout.ms是3秒，那么heartbeat.interval.ms就应该是1秒。

把session.timeout.ms设置得比默认值小，可以更快地检测到崩溃，并从崩溃中恢复，但也会导致不必要的再均衡。把session.timeout.ms设置得比默认值大，可以减少意外的再均衡，但需要更长的时间才能检测到崩溃。

**2. max.poll.interval.ms**

这个属性指定了**消费者在被认为已经“死亡”之前可以在多长时间内不发起轮询。**

**心跳是通过后台线程发送的，而后台线程有可能在消费者主线程发生死锁的情况下继续发送心跳**，但这个消费者并没有在读取分区里的数据。要想知道消费者是否还在处理消息，最简单的方法是检查它是否还在请求数据。但是，请求之间的时间间隔是很难预测的，它不仅取决于可用的数据量、消费者处理数据的方式，有时还取决于其他服务的延迟。在需要耗费时间来处理每个记录的应用程序中，可以通过max.poll.records来限制返回的数据量，从而限制应用程序在再次调用poll()之前的等待时长。但是，即使设置了max.poll.records，调用poll()的时间间隔仍然很难预测。于是，设置max.poll.interval.ms就成了一种保险措施。它必须被设置得足够大，让正常的消费者尽量不触及这个阈值，但又要足够小，避免有问题的消费者给应用程序造成严重影响。**这个属性的默认值为5分钟。当这个阈值被触及时，后台线程将向broker发送一个“离开群组”的请求，让broker知道这个消费者已经“死亡”，必须进行群组再均衡，然后停止发送心跳**。

#### 4.2.3 吞吐量

**1. fetch.min.bytes**

这个属性指定了消费者从服务器获取记录的最小字节数，默认是1字节。broker在收到消费者的获取数据请求时，如果可用数据量小于fetch.min.bytes指定的大小，那么它就会等到有足够可用数据时才将数据返回。

**2. fetch.max.wait.ms**

通过设置fetch.min.bytes，可以让Kafka等到有足够多的数据时才将它们返回给消费者，feth.max.wait.ms则用于指定broker等待的时间，默认是500毫秒。如果没有足够多的数据流入Kafka，那么消费者获取数据的请求就得不到满足，最多会导致500毫秒的延迟。

**3. fetch.max.bytes**

这个属性指定了Kafka返回的数据的最大字节数（默认为50 MB）。**消费者会将服务器返回的数据放在内存中，所以这个属性被用于限制消费者用来存放数据的内存大小**。需要注意的是，记录是分批发送给客户端的，如果broker要发送的批次超过了这个属性指定的大小，那么这个限制将被忽略。

**4. max.poll.records**

这个属性用于**控制单次调用poll()方法返回的记录条数**。可以用它来控制应用程序在进行每一次轮询循环时需要处理的记录条数（不是记录的大小）。

**5. max.partition.fetch.bytes**

这个属性**指定了服务器从每个分区里返回给消费者的最大字节数**（默认值是1 MB）。当KafkaConsumer.poll()方法返回ConsumerRecords时，从每个分区里返回的记录最多不超过max.partition.fetch.bytes指定的字节。

#### 4.2.4 偏移量

**1. auto.offset.reset**

这个属性指定了消费者在读取一个没有偏移量或偏移量无效（因消费者长时间不在线，偏移量对应的记录已经过期并被删除）的分区时该做何处理。它的默认值是latest，意思是说，如果没有有效的偏移量，那么消费者将从最新的记录（在消费者启动之后写入Kafka的记录）开始读取。另一个值是earliest，意思是说，如果没有有效的偏移量，那么消费者将从起始位置开始读取记录。如果将auto.offset.reset设置为none，并试图用一个无效的偏移量来读取记录，则消费者将抛出异常。

**2. enable.auto.commit**

这个属性指定了消费者是否自动提交偏移量，默认值是true。如果它被设置为true，那么还有另外一个属性auto.commit.interval.ms可以用来控制偏移量的提交频率。

> **offsets.retention.minutes**
>
> **这是broker端的一个配置属性**，需要注意的是，它也会影响消费者的行为。**只要消费者群组里有活跃的成员（也就是说，有成员通过发送心跳来保持其身份），群组提交的每一个分区的最后一个偏移量就会被Kafka保留下来，在进行重分配或重启之后就可以获取到这些偏移量**。但是，如果一个消费者群组失去了所有成员，则Kafka只会按照这个属性指定的时间（默认为7天）保留偏移量。一旦偏移量被删除，即使消费者群组又“活”了过来，它也会像一个全新的群组一样，没有了过去的消费记忆。

#### 4.2.5 协作（增量式）再均衡

**1. group.id**

group.id不是必需的，它指定了一个消费者属于哪一个消费者群组。可以创建不属于任何一个群组的消费者，只是这种做法不太常见。

**2. group.instance.id**

这个属性可以是任意具有唯一性的字符串，被用于消费者群组的固定名称。

**3. partition.assignment.strategy**

PartitionAssignor根据给定的消费者和它们订阅的主题来决定哪些分区应该被分配给哪个消费者。Kafka提供了几种默认的分配策略。

**区间(range)**

这个策略会**把每一个主题的若干个连续分区分配给消费者**。假设消费者C1和消费者C2同时订阅了主题T1和主题T2，并且每个主题有3个分区。那么消费者C1有可能会被分配到这两个主题的分区0和分区1，消费者C2则会被分配到这两个主题的分区2。因为每个主题拥有奇数个分区，并且都遵循一样的分配策略，所以第一个消费者会分配到比第二个消费者更多的分区。只要使用了这个策略，并且分区数量无法被消费者数量整除，就会出现这种情况。

**轮询(roundRobin)**

这个策略会**把所有被订阅的主题的所有分区按顺序逐个分配给消费者**。如果使用轮询策略为消费者C1和消费者C2分配分区，那么消费者C1将分配到主题T1的分区0和分区2以及主题T2的分区1，消费者C2将分配到主题T1的分区1以及主题T2的分区0和分区2。一般来说，如果所有消费者都订阅了相同的主题（这种情况很常见），那么轮询策略会给所有消费者都分配相同数量（或最多就差一个）的分区。

**黏性(sticky)**

设计黏性分区分配器的目的有两个：一是尽可能均衡地分配分区，二是**在进行再均衡时尽可能多地保留原先的分区所有权关系，减少将分区从一个消费者转移给另一个消费者所带来的开销**。如果所有消费者都订阅了相同的主题，那么黏性分配器初始的分配比例将与轮询分配器一样均衡。后续的重新分配将同样保持均衡，但减少了需要移动的分区的数量。如果同一个群组里的消费者订阅了不同的主题，那么黏性分配器的分配比例将比轮询分配器更加均衡。

**协作黏性(cooperative sticky)**

这个分配策略与黏性分配器一样，只是它**支持协作（增量式）再均衡**，在进行再均衡时消费者可以继续从没有被重新分配的分区读取消息。

#### 4.2.6 其他

**1. client.id**

这个属性可以是任意字符串，broker用它来标识从客户端发送过来的请求，比如获取请求。它通常被用在日志、指标和配额中。

**2. client.rack**

在默认情况下，消费者会从每个分区的首领副本那里获取消息。但是，如果集群跨越了多个数据中心或多个云区域，那么让消费者从位于同一区域的副本那里获取消息就会具有性能和成本方面的优势。**要从最近的副本获取消息，需要设置client.rack这个参数，用于标识客户端所在的区域**。然后，可以**将broker的replica.selector.class参数值改为org.apache.kafka.common.replica.RackAwareReplicaSelector**。

**3. default.api.timeout.ms**

如果在调用消费者API时没有显式地指定超时时间，那么消费者就会在调用其他API时使用这个属性指定的值。默认值是1分钟，因为它比请求超时时间的默认值大，所以可以将重试时间包含在内。poll()方法是一个例外，因为它需要显式地指定超时时间。

**4. request.timeout.ms**

这个属性指定了**消费者在收到broker响应之前可以等待的最长时间**。如果broker在指定时间内没有做出响应，那么客户端就会关闭连接并尝试重连。它的默认值是30秒。不建议把它设置得比默认值小。在放弃请求之前要给broker留有足够长的时间来处理其他请求，因为向已经过载的broker发送请求几乎没有什么好处，况且断开并重连只会造成更大的开销。

**5. receive.buffer.bytes和send.buffer.bytes**

这两个属性分别指定了socket在读写数据时用到的TCP缓冲区大小。如果它们被设置为–1，就使用操作系统的默认值。如果生产者或消费者与broker位于不同的数据中心，则可以适当加大它们的值，因为跨数据中心网络的延迟一般都比较高，而带宽又比较低。

### 4.3 订阅主题

在创建好消费者之后，下一步就可以开始订阅主题了。**subscribe()**方法会接收一个主题列表作为参数，也可以在调用subscribe()方法时传入一个**正则表达式**。**正则表达式可以匹配多个主题，如果有人创建了新主题，并且主题的名字与正则表达式匹配，那么就会立即触发一次再均衡，然后消费者就可以读取新主题里的消息**。如果应用程序需要读取多个主题，并且可以处理不同类型的数据，那么这种订阅方式就很有用。在Kafka和其他系统之间复制数据的应用程序或流式处理应用程序经常使用正则表达式来订阅多个主题。

> 当你使用正则表达式而不是指定列表订阅主题时，消费者将定期向broker请求所有已订阅的主题及分区。然后，客户端会用这个列表来检查是否有新增的主题，如果有，就订阅它们。**如果主题很多，消费者也很多，那么通过正则表达式订阅主题就会给broker、客户端和网络带来很大的开销**。在某些情况下，主题元数据使用的带宽会超过用于发送数据的带宽。另外，**为了能够使用正则表达式订阅主题，需要授予客户端获取集群全部主题元数据的权限，即全面描述整个集群的权限**。

### 4.4 轮询

消费者API最核心的东西是通过一个简单的轮询向服务器请求数据。

像鲨鱼停止移动就会死掉一样，**消费者必须持续对Kafka进行轮询，否则会被认为已经“死亡”**，它所消费的分区将被移交给群组里其他的消费者。传给poll()的参数是一个超时时间间隔，用于控制poll()的阻塞时间（当消费者缓冲区里没有可用数据时会发生阻塞）。如果这个参数被设置为0或者有可用的数据，那么poll()就会立即返回，否则它会等待指定的毫秒数。

> 在旧版本Kafka中，轮询方法的完整签名是poll(long)。现在，这个签名被弃用了，新API的签名是poll(Duration)。除了参数类型发生变化，方法体里的阻塞语义也发生了细微的改变。**原来的方法会一直阻塞，直到从Kafka获取所需的元数据，即使阻塞时间比指定的超时时间还长**。**新方法将遵守超时限制，不会一直等待元数据返回**。如果你已经有一个消费者使用poll(0) 来获取Kafka元数据（不消费任何记录，这是一种相当常见的做法），那么就不要指望把它改成poll(Duration.ofMillis(0)) 后还能获得同样的效果。你需要想新的办法来达到目的。通常的解决办法是将逻辑放在rebalanceListener.onPartitionAssignment()方法里，这个方法一定会在获取分区元数据之后以及记录开始到达之前被调用。

轮询不只是获取数据那么简单。**在第一次调用消费者的poll()方法时，它需要找到群组协调器，加入群组，并接收分配给它的分区**。如果触发了再均衡，则整个**再均衡过程也会在轮询里进行**，包括执行相关的回调。所以，消费者或回调里可能出现的错误最后都会转化成poll()方法抛出的异常。

需要注意的是，**如果超过max.poll.interval.ms没有调用poll()，则消费者将被认为已经“死亡”**，并被逐出消费者群组。因此，要避免在轮询循环中做任何可能导致不可预知的阻塞的操作。

> 我们既不能在同一个线程中运行多个同属一个群组的消费者，也不能保证多个线程能够安全地共享一个消费者。按照规则，一个消费者使用一个线程。

#### 4.4.1 提交和偏移量

**消费者会向一个叫作\_\_consumer\_offset的主题发送消息，消息里包含每个分区的偏移量**。**如果消费者一直处于运行状态，那么偏移量就没有什么实际作用**。但是，如果消费者发生崩溃或有新的消费者加入群组，则会触发再均衡。再均衡完成之后，每个消费者可能会被分配新的分区，而不是之前读取的那个。为了能够继续之前的工作，消费者需要读取每个分区最后一次提交的偏移量，然后从偏移量指定的位置继续读取消息。

如果最后一次提交的偏移量小于客户端处理的最后一条消息的偏移量，那么处于两个偏移量之间的消息就会被重复处理。如果最后一次提交的偏移量大于客户端处理的最后一条消息的偏移量，那么处于两个偏移量之间的消息就会丢失。

> 如果使用自动提交或不指定提交的偏移量，那么将默认提交poll()返回的最后一个位置之后的偏移量。

**1. 自动提交**

最简单的提交方式是让消费者自动提交偏移量。**如果enable.auto.commit被设置为true，那么每过5秒，消费者就会自动提交poll()返回的最大偏移量。提交时间间隔通过auto.commit.interval.ms来设定**，默认是5秒。与消费者中的其他处理过程一样，自动提交也是在轮询循环中进行的。**消费者会在每次轮询时检查是否该提交偏移量了，如果是，就会提交最后一次轮询返回的偏移量**。

在使用自动提交时，到了该提交偏移量的时候，轮询方法将提交上一次轮询返回的偏移量，但它并不知道具体哪些消息已经被处理过了，所以，**在再次调用poll()之前，要确保上一次poll()返回的所有消息都已经处理完毕**（调用close()方法也会自动提交偏移量）。通常情况下这不会有什么问题，但在处理异常或提前退出轮询循环时需要特别小心。

**2. 手动提交**

把**enable.auto.commit设置为false**，让应用程序自己决定何时提交偏移量。

**1）提交当前偏移量**

使用commitSync()和commitAsync()提交poll()返回的最新偏移量。commitSync()是同步提交，在提交成功或碰到无法恢复的错误之前，commitSync()会一直重试；但commitAsync()是异步提交，在提交成功或碰到无法恢复的错误之前，commitAsync()不会进行重试，因为commitAsync()在收到服务器端的响应时，可能已经有一个更大的偏移量提交成功。

需要注意的是，commitSync()将会提交poll()返回的最新偏移量，所以，如果你在处理完所有记录之前就调用了commitSync()，那么一旦应用程序发生崩溃，就会有丢失消息的风险（消息已被提交但未被处理）。如果应用程序在处理记录时发生崩溃，但commitSync()还没有被调用，那么从最近批次的开始位置到发生再均衡时的所有消息都将被再次处理。

commitAsync()支持回调，回调会在broker返回响应时执行。回调经常被用于记录偏移量提交错误或生成指标，如果要用它来重试提交偏移量，那么一定要注意提交顺序。

> 可以用一个单调递增的消费者序列号变量来维护异步提交的顺序。每次调用commitAsync()后增加序列号，并在回调中更新序列号变量。在准备好进行重试时，先检查回调的序列号与序列号变量是否相等。如果相等，就说明没有新的提交，可以安全地进行重试。如果序列号变量比较大，则说明已经有新的提交了，此时应该停止重试。

一般情况下，偶尔提交失败但不进行重试不会有太大问题，因为如果提交失败是由于临时问题导致的，后续的提交总会成功。但如果这是发生在消费者被关闭或再均衡前的最后一次提交，则要确保提交是成功的。如果是消费者被关闭，那么一般会使用commitAsync()和commitSync()的组合。

**2）提交特定的偏移量**

消费者API允许在调用commitSync()和commitAsync()时传给它们想要提交的分区和偏移量。需要注意的是，因为一个消费者可能不止读取一个分区，你需要跟踪所有分区的偏移量，所以通过这种方式提交偏移量会让代码变得复杂。

#### 4.4.2 再均衡监听器

消费者API提供了一些方法，让你可以在消费者分配到新分区或旧分区被移除时执行一些代码逻辑。你所要做的就是在调用subscribe()方法时传进去一个**ConsumerRebalanceListener**对象。ConsumerRebalanceListener有3个需要实现的方法：

```
  
  public void onPartitionsAssigned(Collection<TopicPartition> partitions)
```

这个方法会在**重新分配分区之后**以及**消费者开始读取消息之前**被调用。你可以在这个方法中**准备或加载与分区相关的状态信息**、**找到正确的偏移量**，等等。**这里所有的事情都应该保证在max.poll.timeout.ms内完成，以便消费者可以成功地加入群组。**

```
  
  public void onPartitionsRevoked(Collection<TopicPartition> partitions)
```

这个方法会在**消费者放弃对分区的所有权时**调用——可能是因为发生了**再均衡**或者**消费者正在被关闭**。通常情况下，**如果使用了主动再均衡算法，那么这个方法会在再平衡开始之前以及消费者停止读取消息之后调用**。**如果使用了协作再均衡算法，那么这个方法会在再均衡结束时调用，而且只涉及消费者放弃所有权的那些分区**。如果你**要提交偏移量，那么可以在这里提交**，无论是哪个消费者接管这个分区，它都知道应该从哪里开始读取消息。

```
  
  public void onPartitionsLost(Collection<TopicPartition>partitions)
```

这个方法**只会在使用了协作再均衡算法并且之前不是通过再均衡获得的分区被重新分配给其他消费者时调用**（之前通过再均衡获得的分区被重新分配时会调用onPartitionsRevoked()）。你可以在这里**清除与这些分区相关的状态或资源**。需要注意的是，在清理状态时要非常小心，因为分区的新所有者可能也保存了分区状态，需要避免发生冲突。如果你没有实现这个方法，则onPartitionsRevoked()将被调用。

> 如果使用了协作再均衡算法，那么需要注意以下几点。
>
> * onPartitionsAssigned()在每次进行再均衡时都会被调用，以此来告诉消费者发生了再均衡。如果没有新的分区分配给消费者，那么它的参数就是一个空集合。
> * onPartitionsRevoked()会在进行正常的再均衡并且有消费者放弃分区所有权时被调用。如果它被调用，那么参数就不会是空集合。
> * onPartitionsLost()会在进行意外的再均衡并且参数集合中的分区已经有新的所有者的情况下被调用。
>
> 如果这3个方法你都实现了，那么就可以保证在一个正常的再均衡过程中，分区新所有者的onPartitionsAssigned()会在之前的所有者的onPartitionsRevoked()被调用完毕并放弃了所有权之后被调用。

```
  
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

#### 4.4.3 从特定偏移量位置读取记录

如果你想从分区的起始位置读取所有的消息，或者直接跳到分区的末尾读取新消息，那么Kafka API分别提供了两个方法：`seekToBeginning(Collection<Topic Partition> tp)` 和`seekToEnd(Collection<TopicPartition> tp)`。

Kafka还提供了用于查找特定偏移量的API。下面的例子演示了如何将分区的当前偏移量定位到在指定时间点生成的记录。

```
  
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

### 4.5 关闭消费者

如果你确定马上要关闭消费者（即使消费者还在等待一个poll()返回），那么可以在另一个线程中调用**consumer.wakeup()**。如果轮询循环运行在主线程中，那么可以在ShutdownHook里调用这个方法。需要注意的是，**consumer.wakeup()是消费者唯一一个可以在其他线程中安全调用的方法**。调用consumer.wakeup()会导致poll()抛出WakeupException，如果调用consumer.wakeup()时线程没有在轮询，那么异常将在下一次调用poll()时抛出。不一定要处理WakeupException，但在**退出线程之前必须调用consumer.close()**。**消费者在被关闭时会提交还没有提交的偏移量，并向消费者协调器发送消息，告知自己正在离开群组。协调器会立即触发再均衡，被关闭的消费者所拥有的分区将被重新分配给群组里其他的消费者，不需要等待会话超时。**

```
  
  Runtime.getRuntime().addShutdownHook(new Thread() {
      public void run() {
          System.out.println("Starting exit...");
          consumer.wakeup(); 
          try {
              mainThread.join();
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }
  });
  ​
  ...
  Duration timeout = Duration.ofMillis(10000); 
  ​
  try {
      // 一直循环，直到按下Ctrl-C组合键，关闭钩子会在退出时做清理工作
      while (true) {
          ConsumerRecords<String, String> records =
              movingAvg.consumer.poll(timeout);
          System.out.println(System.currentTimeMillis() +
              "-- waiting for data...");
          for (ConsumerRecord<String, String> record : records) {
              System.out.printf("offset = %d, key = %s, value = %s\n",
                  record.offset(), record.key(), record.value());
          }
          for (TopicPartition tp: consumer.assignment())
              System.out.println("Committing offset at position:" +
                  consumer.position(tp));
          movingAvg.consumer.commitSync();
      }
  } catch (WakeupException e) {
      // 忽略异常 
  } finally {
      // 在退出之前，确保彻底关闭了消费者
      consumer.close(); 
      System.out.println("Closed consumer and we are done");
  }
```
