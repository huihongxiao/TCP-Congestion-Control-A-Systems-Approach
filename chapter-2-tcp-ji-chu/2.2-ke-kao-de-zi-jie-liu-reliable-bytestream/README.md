# 2.2 可靠的字节流（Reliable Byte-Stream）

在IP协议的尽力而为（best-effort）传输模型之上，在运行于终端主机上的一对进程之间，TCP实现了可靠的字节流。这一节会详细描述TCP的实现，以助于更好的理解后面章节将要介绍的拥塞控制算法。

## 2.2.1 TCP需要解决的问题

TCP的核心是滑动窗口算法。TCP除了大家熟悉的确认/超时/重传机制外，还必须解决下面的问题。

1. 因为TCP运行在互联网上的任意两个计算机上的两个进程之间，它需要一个明确的连接建立过程，使得参与传输的双方都同意与对方进行数据传输。在连接建立的过程中，TCP的传输双方需要共享状态以启动滑动窗口算法。传输结束后还需要关闭连接，这样传输双方才知道可以释放状态。
2. 不同的TCP 连接的RTT（Round-trip time，往返时间）时间可能相差非常之大。例如，San Francisco 和 Boston之间相隔数千公里，它们之间的TCP 连接RTT 可能需要 100ms。然而，同一个房间的两个主机之间的 TCP 连接可能只需要 1ms 的 RTT。同一个 TCP 协议需要能同时支持这两种场景。更糟糕的是，San Francisco 和 Boston之间的 TCP 连接可能在早上 3 点的时候 RTT 是 100ms，而到了下午 3 点 RTT 就变成了 500ms。甚至在一个仅持续几分钟的 TCP 连接中，RTT也可能发生改变。这意味着TCP中触发重传的超时机制必须是自适应的。
3. 由于互联网的尽力而为（best-effort）传输特性，packet 在传输过程中可能会乱序。packet 轻微的乱序不会造成问题，因为滑动窗口算法可以基于 sequence number 正确的重排 packets，所以这不是真正的问题（╮(╯▽╰)╭）。互联网的尽力而为传输带来的真正问题时，packet 究竟可以多乱，换句话说，一个 packet 最长会延迟多久才到达目的地。在最糟的情况下，packet 可以在互联网上被耽搁无限久。每当一个 packet 被一个路由器转发时，IP header 中的 time-to-live（TTL）字段就会被减一，最终达到 0。此时，packet 会被丢弃（这种 packet 不会迟到，而是事实上的丢包了）。TTL 在 IP 协议里被错误命名了，并且在 IPv6中被重新命名为更准确的 Hop Count。由于知道 IP 协议会在 TTL 耗尽之后丢弃 packet，所以 TCP 假设每个 packet 都有一个最大的生命周期。在 TCP 协议里，这个最大生命周期是 Maximum Segment Life(MSL)，当前的默认值是 120 秒。但这仅仅是个工程实现上的变量，因为 IP 协议并不会强制在 120 秒之后丢包，这个值仅仅是 TCP 预估一个 packet 在互联网上存活的最长时间。虽然如此，这个值的象征意义还是非常重要，它表明TCP 需要准备好面对被耽误了很久（注，少于MSL 120秒）的 packet 突然出现，尽管这个 packet 会使得滑动窗口算法处理起来很复杂。
4. 因为任何计算机都可以接入到互联网，所以分配给 TCP 连接的资源也千差万别，尤其考虑到任何一个主机都可以同时支持数百（万）个 TCP 连接。这意味着 TCP 需要有一种机制来使得传输的双方能够“学习”到对方能够提供的资源（例如，对方有多少 buffer 空间）。这是流控（flow control）需要解决的问题。
5. TCP 的发送端并不知道要通过什么样的网络链路才能到达接收端。例如，发送端直连的是一个 10Gbps 的线路，但是网络的中间一个必经点可能只有 1.5Mbps。更糟糕的是，这 1.5Mbps 的线路还要被多个 TCP 连接共享。退一步说，即使是一个大容量的线路，如果承接了足够多的流量也会拥塞。有关拥塞控制是本书的重点，会在后面详细介绍。

## 2.2.2 TCP 数据格式

TCP 是一个面向字节的协议，这意味着发送端向 TCP 连接中写入字节，而接收端从 TCP 连接中读取字节。尽管“字节流”描述了 TCP 为应用程序提供的服务，但是 TCP 本身并不会在互联网上传输单个的字节。相应的，在发送端，TCP会从发送程序中接收并缓存足够多字节数的数据，然后填装在合适大小的 packet 中，然后将这个 packet 发送给 TCP 的接收端。TCP 的接收端会将 packet 中的所有数据存入到接收缓存中，之后，接收端程序在有空的时候读取这个 buffer 中的数据。这个过程描述在图 7 中，为了简化描述，图中只画了单向数据传输。

<figure><img src="../../.gitbook/assets/image (3) (1).png" alt="" width="375"><figcaption><p>图 7：TCP 如何管理字节流</p></figcaption></figure>

图 7 中，TCP 连接的两端交换的 packet 被称为 segment，这是因为每个 segment 都携带了字节流的一段数据。每个 TCP segment 都携带了图 8 所示的 header。

<figure><img src="../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt="" width="375"><figcaption><p>图 8：TCP header 格式</p></figcaption></figure>

* `SrcPort` 和 `DstPort` 表明了源和目的的传输层端口。这两个字段加上源目的IP地址，一起唯一标识了一个 TCP 连接。所有与 TCP 连接相关的状态，包括了后面章节介绍的拥塞控制的状态，都唯一对应一个4 元组`(SrcPort, SrcIPAddr, DstPort, DstIPAddr)`。
* `Acknowledgment`, `SequenceNum`, 和 `AdvertisedWindow`字段与TCP 的滑动窗口算法相关。因为 TCP 是面向字节的协议，传输数据中的每一个字节都有一个序列号（Sequence number）。`SequenceNum`是当前 segment 中所有数据第一个字节对应的序列号。`Acknowledgment`和`AdvertisedWindow`包含了数据流在接收端的信息。为了简化这里的讨论，我们先不考虑数据是双向传递的，这里我们只关注数据向一边传输，如图 9 所示。

<figure><img src="../../.gitbook/assets/image (2) (1) (1) (1) (1) (1).png" alt="" width="375"><figcaption><p>图 9：简化版 TCP 流程，数据只在一个方向传递，ACK 在另一个方向传递</p></figcaption></figure>

* 6bit 的 `Flags` 字段用来在 TCP 两端传递控制信息。控制信息包含了：SYN 和 FIN 标志，用来建立和终结 TCP 连接；ACK 标志，用来表示`Acknowledgment`是有用的（也就是说接收端应该关注`Acknowledgment`）。
* 最后，由于 TCP header 是可变长度的（因为在固定字段后面还有 options），所以 header 中还包含了 `HdrLen` 用来表示 TCP header 的长度是 32bit word 的多少倍。当 TCP extenstion 存在于 header 之后时，这个字段是有用的。为什么选择将 TCP extenstion 作为 option 加在 TCP header 之后而不是修改 TCP header 的固定字段？因为将TCP extension作为可变长度挂在固定字段之后，就算某些 TCP 实现没有包含对应的 extenstion，仍然能使用 TCP 进行通信（注，如果放在固定字段会导致解析出错，因为一般header解析都是移位读取内容）。TCP 会话的两端会在连接建立的过程中协商并同意使用哪些 extenstion/option。

## 2.2.3 可靠和有序的传输

TCP 的滑动窗口算法主要解决两个问题：

1. 确保数据可靠有序的传输
2. 在发送端和接收端之间进行流控（flow control）

为了实现流控，接收端确定好滑动窗口的大小，并通过 TCP header 中的`AdvertisedWindow` 字段传递给发送端。TCP 的接收端根据当前 TCP 连接所分配到的内存大小（也就是接收端的 buffer 大小）来确定合适的`AdvertisedWindow`。TCP 的发送端在任何时候都不会发送超过`AdvertisedWindow`的未确认数据。这样，发送端就不会打爆接收端的buffer。

图 10 展示了 TCP 滑动窗口的工作方式。TCP 的发送端维护了一个发送端 buffer，用来保存已发送但是还未被确认的数据，以及被发送端应用程序写入但是还未被传输的数据。在接收端，TCP 维护了接收端 buffer，用来保存所有接收到的数据，包括乱序和正确顺序但是应用程序还没来的及读的部分。

<figure><img src="../../.gitbook/assets/image (3) (1) (1).png" alt="" width="375"><figcaption><p>图 10：TCP 发送端缓存和接收端缓存之间的关系</p></figcaption></figure>

为了让接下来的讨论更加简单，我们先忽略 buffer 和序列号都是有限大小，并且最终都会溢出这个事实（注，TCP的序列号时32bit整数，针对字节数而言极易溢出）。并且我们也不区分buffer 使用的指针和序列号（注，实际中两者的相对位置是一样的，但是具体数值大概率不一样）。

发送端buffer维护了三个指针，`LastByteAcked`, `LastByteSent`, 和 `LastByteWritten`，它们的名字都说明了它们的含义。因为只有数据发送了才有可能被接收端确认，所以`LastByteSent`大于等于`LastByteAcked`。因为只有数据被TCP 写入到buffer了，才有可能被发送，所以`LastByteWritten`大于等于`LastByteSent`。最终，这三个指针之间有以下明显的关系：

<figure><img src="../../.gitbook/assets/image (4) (1).png" alt="" width="375"><figcaption></figcaption></figure>

在接收端也有类似的指针（序列号），`LastByteRead`, `NextByteExpected`, 和 `LastByteRcvd`。由于乱序的存在，这里的不等式有一点点不好理解。首先，因为TCP会确保顺序，所以NextByteExpected之前的字节必然已经全部被接受，而数据只有被接受了才有可能被接收端应用程序读取，所以`NextByteExpected - 1` 大于等于`LastByteRead`。当数据顺序传输时，`NextByteExpected` 等于`LastByteRcvd + 1`。而当数据乱序时，`LastByteRcvd` 大于`NextByteExpected` ，如图10所示（注，此时表明接收端希望重传一个`LastByteRcvd` 之前的packet）。最终，这三者的关系如下表示：

<figure><img src="../../.gitbook/assets/image (5).png" alt="" width="375"><figcaption></figcaption></figure>

## 2.2.4 流控（Flow Control）

截止到目前为止，所有的讨论都假设接收端可以瞬间处理完接收到的数据。但实际情况并不一定这样，接收端程序可能本身要花费大量的时间来处理和运算数据，所以数据可能在接收端buffer堆积，而接收端 buffer大小又不是无限的。所以接收端需要有某种方式来降低发送段端的速率，以避免自身buffer被打爆，这就是TCP流控的核心。

尽管前面已经说过，TCP 的流控（Flow Control）和拥塞控制（Congestion Control）是不同的机制，为了理解拥塞控制，必须要先理解流控。另一方面，用来实现流控的窗口机制，在拥塞控制中也很重要。在流控中，窗口机制向发送端提供了一个清晰的指示：当前可以有多少未被确认的数据被发送出来，而这同时是流控和拥塞控制的核心问题。

接下来，我们将认为 buffer 都是有限大小。TCP 接收端会通过通知发送端自己当前的 buffer 大小来调整发送端的速率。TCP 的接收端必须保证：

<figure><img src="../../.gitbook/assets/image (6).png" alt="" width="375"><figcaption></figcaption></figure>

来避免自己的 buffer 被打爆。因此它会发布其 buffer 中空余的空间 `AdvertisedWindow` 大小为：

<figure><img src="../../.gitbook/assets/image (7).png" alt="" width="375"><figcaption></figcaption></figure>

当接收到数据时，只要 TCP 接收端已经收到了这份数据前面的所有数据，它就会确认这份数据。对应的`NextByteExpected`会向右移动（增加），意味着 `AdvertisedWindow` 相应的减少。实际中，窗口是否真的会变小，取决于本地应用程序消费数据的速度。如果本地进程能够与数据传输相同的速度消费数据（也就是`LastByteRead` 以与`LastByteRcvd`相同的速率增加），那么`AdvertisedWindow` = `RcvBufferSize`，也就是说接收端的buffer总是为空。然而，如果接收程序落后了，例如程序要对读取的每个字节进行大量的运算，那么每当segment到达时，`AdvertisedWindow`都会变小，最终变成0。

TCP发送端需要遵循它从接收端获取的`AdvertisedWindow`，在任何时刻，它必须保证：

<figure><img src="../../.gitbook/assets/image (8).png" alt="" width="375"><figcaption></figcaption></figure>

换句话说，发送端会计算出它还可以发送的未被确认的数据量，并记录为`EffectiveWindow`（注，EffectiveWindow表示的是将来还可以发送的未被确认的数据量，而不是当前所有的未被确认的数据量）。

<figure><img src="../../.gitbook/assets/image (9).png" alt="" width="375"><figcaption></figcaption></figure>

很明显，`EffectiveWindow`要大于0，TCP发送端才能继续发送数据。存在这样一种场景：TCP发送端收到了一个ACK确认了x字节的数据，因此使得发送端增加`LastByteAcked` x个字节；但是因为接收端程序并没有读取任何数据，`AdvertisedWindow`比之前小了x个字节。在这种情况下，发送端是可以释放自己的buffer（注，刚刚被确认的数据可以从buffer中被释放掉），但是不能发送任何新的数据。

除了接收端的buffer之外，发送端的buffer也必须注意。在发送数据的过程中，TCP发送端必须确保本地发送数据的程序不会过载发送端buffer，也就是要确保：

<figure><img src="../../.gitbook/assets/image (10).png" alt="" width="375"><figcaption></figcaption></figure>

如果发送端程序想要向TCP写入b个字节，但是

<figure><img src="../../.gitbook/assets/image (11).png" alt="" width="375"><figcaption></figcaption></figure>

此时，TCP会挂起发送端程序，并且不允许它向TCP写入数据。

现在大家应该可以理解一个慢接收端是如何阻止一个快的发送端了。

* 首先，慢接收端处理数据很慢，导致从接收端buffer读数据很慢，导致buffer被填满。这意味着`AdvertisedWindow`减小为0，也意味着发送端不能再发送任何数据，即使之前发送的数据被ACK了也不行。
* 其次，不能发送数据意味着发送端buffer会被发送端程序填满，最终会导致TCP挂起发送端程序，从而导致发送端被阻止。
* 一旦接收端程序开始读取数据，接收端TCP就能打开其窗口（`AdvertisedWindow` > 0），进而使得TCP的发送端从其buffer中发送数据。
* 当发送的数据被ACK之后，`LastByteAcked` 会增加，对应的被确认部分的数据会被从发送端buffer中释放，发送端的程序会继续运行。

接下来还有最后一个细节需要关注：发送端怎么能知道`AdvertisedWindow`不为0了？

正常情况下，TCP接收端是通过ACK来确认接收了一个数据segment。在ACK里，会包含最新的`Acknowledge`和`AdvertisedWindow`字段，就算这些字段与上一次相比没有变化，也会包含在ACK里，发送端可以用这些数据来更新`AdvertisedWindow`。

现在的问题是，当`AdvertisedWindow`为0时，发送端不再发送任何数据，也就不能收到任何ACK，也就没法知道什么时候`AdvertisedWindow`不再是0。TCP的接收端不会自发的发送ACK，它只会在收到数据segment时回复对应的ACK。

TCP会这样处理这个问题：当对端宣告`AdvertisedWindow`为0时，发送端会时不时的发送一个1字节的segment，它知道数据可能不会被接收，但它还是要尝试一下，因为每一个这样的segment都会触发接收端的一个回复，其中包含了最新的`AdvertisedWindow`，而这个窗口终将打开（注，即AdvertisedWindow变成非0）。这里的1字节消息被称为Zero Window Probe，实际中它们会每5-60秒发送一次。

## 2.2.5 触发TCP传输

我们接下来讨论一个非常微妙的问题：TCP如何决定是否应该传输一个segment？如果我们忽略流控并认为窗口总是打开的，那么TCP发送端有三个场景可以触发一个segment的传输：

* TCP维护了一个变量，称为maximum segment size（MSS），当它从发送程序中收集到MSS个字节时，它会立刻送出一个segment。
* 发送端程序通过一个push操作，显示的要求TCP发送一个segment。这会使得TCP发送完buffer中未发送的字节。
* 定时器触发，会使得当前buffer中的所有数据被打包成一个segment被发送出去。

当然，我们不能忽略流控。如果发送端的buffer已经有了MSS个字节，且当前`AdvertisedWindow`也支持足够多的数据，那么TCP发送端就发送一个完整的segment。然而，假设发送端buffer正在累积数据，而此时窗口是关闭的；过会窗口打开了，但是只有MSS/2个字节。发送端应该发送一个一半大小的segment还是等窗口打开到完整的MSS再进行发送？

最早的TCP规范文档中没有提及这个问题，早期的TCP实现会发送一个一半大小的segment。但是后来发现，这种利用任何可用窗口的激进策略会导致一个问题：_silly window syndrome。_在这个问题里，由于`AdvertisedWindow`断断续续的打开，发送端不能将数据聚合成一个完整的MSS再发送。为了解决这个问题，引入了一个复杂的决策过程，被称为Nagle算法。之所以这里要提到这个算法，因为它会成为后面介绍的拥塞控制算法的核心部分。

Nagle算法面对的核心问题是：当`EffectiveWindow`小于MSS时，发送端需要等待多久再发送一个segment？如果我们等的时间过长，那么我们就伤害到了应用程序的交互的实时性。如果我们等的时间不够长，那么我们可能会发送一堆小的packet，并且陷入到_silly window syndrome（注，SWS的主要问题是效率太低，因为处理每个segment都需要一定的代价，而现在每个segment的数据都很少，处理代价就相对较高）。_

Nagle算法引入了一种优雅的自我驱动的解决方案。它的想法是，只要TCP有任何数据在传输，发送端最终总会收到一个ACK。这个ACK可以被用来当做一个自我驱动的机制，触发发送端发送更多的数据。Nagle算法提供了一个简单的，统一的规则来决定何时传输数据：

{% code overflow="wrap" %}
```
When the application produces data to send 
    if both the available data and the window >= MSS 
        send a full segment 
    else 
        if there is unACKed data in flight 
            buffer the new data until an ACK arrives 
        else 
            send all the new data now 
```
{% endcode %}

可以这样理解Nagle算法：

* 当窗口允许时，总是发送一个完整的segment
* 如果当前没有segment在发送的过程中，那么也可以立即发送一小部分数据
* 如果当前有任何segment在传输过程中，发送端等待ACK再发送下一个segment

这样对于一个连续一次只写入1个字节的应用程序，在每次RTT时间只会发送一个segment。某些segment可能只会包含1个字节，而其他的segment可能会包含在一个RTT时间内用户输入的尽可能多的数据。因为有些程序不能接受这里的延时，所以TCP socket接口提供了一个`TCP_NODELAY`参数，它会使得数据传输尽可能的快（注，也就是相当于关闭了Nagle算法，同时承担SWS的后果）。
