# 会话的相关属性

## <mark style="color:blue;">**session.timeout.ms**</mark>**和**<mark style="color:blue;">**heartbeat.interval.ms**</mark>

**session.timeout.ms**指定了**消费者可以在多长时间内不与服务器发生交互而仍然被认为还“活着”，默认是10秒。如果消费者没有在session.timeout.ms指定的时间内发送心跳给群组协调器，则会被认为已“死亡”，协调器就会触发再均衡，把分区分配给群组里的其他消费者**。

**heartbeat.interval.ms**指定了**消费者向协调器发送心跳的频率**。

> 一般会同时设置这两个属性，**heartbeat.interval.ms**必须比**session.timeout.ms**小，通常前者是后者的1/3。如果session.timeout.ms是3秒，那么heartbeat.interval.ms就应该是1秒。

{% hint style="info" %}
## <mark style="color:blue;">提示</mark>

* **把session.timeout.ms设置得比默认值小，可以更快地检测到崩溃，并从崩溃中恢复，但也会导致不必要的再均衡。**
* **把session.timeout.ms设置得比默认值大，可以减少意外的再均衡，但需要更长的时间才能检测到崩溃。**
{% endhint %}

## <mark style="color:blue;">**max.poll.interval.ms**</mark>

这个属性指定了**消费者在被认为已经“死亡”之前可以在多长时间内不发起轮询。**

> **心跳是通过后台线程发送的，而后台线程有可能在消费者主线程发生死锁的情况下继续发送心跳**，**但这个消费者并没有在读取分区里的数据。**
>
> 要想知道消费者是否还在处理消息，最简单的方法是**检查它是否还在请求数据**。但是，请求之间的时间间隔是很难预测的，它不仅取决于可用的数据量、消费者处理数据的方式，有时还取决于其他服务的延迟。
>
> **在需要耗费时间来处理每个记录的应用程序中，可以通过max.poll.records来限制返回的数据量，从而限制应用程序在再次调用poll()之前的等待时长**。但是，即使设置了max.poll.records，调用poll()的时间间隔仍然很难预测。
>
> 于是，<mark style="color:orange;">**设置max.poll.interval.ms就成了一种保险措施。它必须被设置得足够大，让正常的消费者尽量不触及这个阈值，但又要足够小，避免有问题的消费者给应用程序造成严重影响**</mark>。**这个属性的默认值为5分钟。当这个阈值被触及时，后台线程将向broker发送一个“离开群组”的请求，让broker知道这个消费者已经“死亡”，必须进行群组再均衡，然后停止发送心跳**。
