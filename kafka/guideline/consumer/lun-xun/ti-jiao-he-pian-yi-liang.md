# 提交和偏移量

**消费者会向一个叫作**<mark style="color:blue;">**\_\_consumer\_offset**</mark>**的主题发送消息，消息里包含每个分区的偏移量**。**如果消费者一直处于运行状态，那么偏移量就没有什么实际作用**。

但是，如果消费者发生崩溃或有新的消费者加入群组，则会触发**再均衡**。再均衡完成之后，每个消费者可能会被分配新的分区，而不是之前读取的那个。为了能够继续之前的工作，**消费者需要读取每个分区最后一次提交的偏移量，然后从偏移量指定的位置继续读取消息：**

* **如果最后一次提交的偏移量小于客户端处理的最后一条消息的偏移量，那么处于两个偏移量之间的消息就会被重复处理。**
* **如果最后一次提交的偏移量大于客户端处理的最后一条消息的偏移量，那么处于两个偏移量之间的消息就会丢失。**

{% hint style="info" %}
## <mark style="color:blue;">提示</mark>

<mark style="color:blue;">**如果使用自动提交或不指定提交的偏移量，那么将默认提交poll()返回的最后一个位置之后的偏移量。**</mark>
{% endhint %}

## **自动提交**

最简单的提交方式是让消费者自动提交偏移量。**如果**<mark style="color:blue;">**enable.auto.commit**</mark>**被设置为true，那么每过5秒，消费者就会自动提交poll()返回的最大偏移量。提交时间间隔通过**<mark style="color:blue;">**auto.commit.interval.ms**</mark>**来设定**，默认是5秒。

* 与消费者中的其他处理过程一样，自动提交也是在轮询循环中进行的。**消费者会在每次轮询时检查是否该提交偏移量了，如果是，就会提交最后一次轮询返回的偏移量**。
* 在使用自动提交时，到了该提交偏移量的时候，轮询方法将提交上一次轮询返回的偏移量，但它并不知道具体哪些消息已经被处理过了，所以，<mark style="color:red;">**在再次调用poll()之前，要确保上一次poll()返回的所有消息都已经处理完毕**</mark>**（调用close()方法也会自动提交偏移量）**。通常情况下这不会有什么问题，但在处理异常或提前退出轮询循环时需要特别小心。

## **手动提交**

把<mark style="color:blue;">**enable.auto.commit**</mark>**设置为false**，让应用程序自己决定何时提交偏移量。

### **提交当前偏移量**

使用<mark style="color:blue;">**commitSync()**</mark>和<mark style="color:blue;">**commitAsync()**</mark>提交poll()返回的最新偏移量。

* **commitSync()**是**同步**提交，在提交成功或碰到无法恢复的错误之前，commitSync()会一直重试；
* **commitAsync()**是异步提交，在提交成功或碰到无法恢复的错误之前，commitAsync()不会进行重试，因为commitAsync()在收到服务器端的响应时，可能已经有一个更大的偏移量提交成功。

{% hint style="warning" %}
## <mark style="color:orange;">注意</mark>

* commitSync()将会提交poll()返回的最新偏移量，所以，<mark style="color:orange;">**如果你在处理完所有记录之前就调用了commitSync()，那么一旦应用程序发生崩溃，就会有丢失消息的风险**</mark>（消息已被提交但未被处理）
* <mark style="color:orange;">**如果应用程序在处理记录时发生崩溃，但commitSync()还没有被调用，那么从最近批次的开始位置到发生再均衡时的所有消息都将被再次处理**</mark>
{% endhint %}

{% hint style="success" %}
## <mark style="color:blue;">**提示**</mark>

**commitAsync()支持回调，回调会在broker返回响应时执行**。回调经常被用于记录偏移量提交错误或生成指标，如果要用它来重试提交偏移量，那么一定要注意提交顺序。

可以用一个**单调递增的消费者序列号**变量来维护异步提交的顺序。

* 每次调用commitAsync()后增加序列号，并在回调中更新序列号变量。
* 在准备好进行重试时，先检查回调的序列号与序列号变量是否相等。
  * 如果相等，就说明没有新的提交，可以安全地进行重试。
  * 如果序列号变量比较大，则说明已经有新的提交了，此时应该停止重试。
{% endhint %}

{% hint style="warning" %}
## <mark style="color:orange;">注意</mark>

**一般情况下，偶尔提交失败但不进行重试不会有太大问题，因为如果提交失败是由于临时问题导致的，后续的提交总会成功**。<mark style="color:orange;">**但如果这是发生在消费者被关闭或再均衡前的最后一次提交，则要确保提交是成功的。**</mark>

<mark style="color:orange;">**如果是消费者被关闭，那么一般会使用commitAsync()和commitSync()的组合**</mark><mark style="color:orange;">。</mark>
{% endhint %}

### **提交特定的偏移量**

消费者API允许在调用<mark style="color:blue;">**commitSync()**</mark>和<mark style="color:blue;">**commitAsync()**</mark>时传给它们想要提交的分区和偏移量。

需要注意的是，**因为一个消费者可能不止读取一个分区，因此需要跟踪所有分区的偏移量**，所以通过这种方式提交偏移量会让代码变得复杂。
