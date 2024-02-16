# 4.1 TCP超时时间计算（timeout）

超时和重传是TCP实现可靠字节流传输的核心部分。同时，超时在拥塞控制中也起了很重要的作用。因为超时代表了丢包，而丢包又表示网络现在极有可能处于拥塞状态。可以这样说，TCP的超时机制是实现拥塞控制必不可少的一个环节。（注，这里的超时时间是指 TCP 发出 segment 之后，超过多长时间没有收到 ACK 就认为 segment 已经丢失了。）

在实际中，超时可能有以下几种原因：

* segment丢包了
* segment 没丢，但是ACK丢了
* 都没丢，ACK就是单纯的比期望的时间慢

所以，到底花多少时间等待ACK（注，也就是具体设置的超时时间）非常重要。如果设置不当，我们会误认为当前处于拥塞（注，也就是上面的第三种情况，没有丢包，但是判定认为丢包了进而认为网络拥塞了）。

TCP的拥塞控制算法采用一种自适应的，根据测量获得的RTT作为参数来计算并设置超时时间。听起来很简单，但是完整的实现一方面会比你预期的要复杂的多，另一方面在过去的时间里也经历过多次改进。这一部分我们来回顾超时时间算法的演进过程。

### 4.1.1 最初的算法

我们从最开始在TCP标准文档（注，[RFC793](https://www.ietf.org/rfc/rfc793.txt)）中的简单算法开始。最初算法的核心思想就是计算RTT的运行平均值，然后根据这个值来计算超时时间。具体来说，每次TCP发送一个segment，它会记录发送的时间。当收到这个segment的ACK时，TCP再次读取时间，取差值作为`SampleRTT`。然后，TCP会计算`EstimatedRTT`，具体方法是将上一次的`EstimatedRTT`和新的`SampleRTT`求加权平均。

<figure><img src="../../.gitbook/assets/image (2) (1) (1) (1) (1) (1) (1).png" alt="" width="375"><figcaption></figcaption></figure>

这里的参数α就是加权值。α小一点的话，RTT的波动会体现的更加明显，但`EstimatedRTT`可能会受临时的波动影响。另一方面α大一些的话，`EstimatedRTT`会更加平稳，但是对于真实的网络质量的改变又不够灵敏。在 RFC793里建议设置α在0.8到0.9之间。

之后，TCP会使用一种非常保守的方式来根据`EstimatedRTT`计算超时时间：

<figure><img src="../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt="" width="253"><figcaption></figcaption></figure>

### 4.1.2 Karn/Partridge 算法

在最初的算法提出几年后，有个明显的缺陷被发现了：ACK确认的不是数据的传输，而是数据的接收。举个例子，当一个segment重传了之后，TCP的发送端收到了属于这个segment的ACK，它不能确定这个ACK应该和最初的还是重传的segment去对应（注，也就是说 ACK 只是确认这个 segment 收到了，但是不确认自己对应哪次发送），进而计算`SampleRTT`。

为了得到一个精确的`SampleRTT`，还是需要知道这个ACK对应最初的segment，还是重传的segment。如图21所示，如果认为ACK对应原始的segment，但实际上它又属于重传的segment，那么算出来的`SampleRTT`就会大于实际值（a）；但是如果认为ACK是重传的segment的，但是它又实际属于原始的segment，那么`SampleRTT`又会小于实际值（b）。

<figure><img src="../../.gitbook/assets/image (2) (1) (1) (1) (1) (1) (1) (1).png" alt=""><figcaption><p>图21. 将ACK关联到（a）原始segment 和 （b）重传的segment。</p></figcaption></figure>

Karn/Partridge 算法以其发明者命名，它的解决思路很简单，包含两个主要的改进：

1. 当TCP重传一个segment时，停止采样RTT，也就是说TCP只会对没有发生重传的segment来测量`SampleRTT`。&#x20;
2. 当TCP重传一个segment时，它会将下一次的超时时间设置为当前超时时间的2倍，而不是基于上面的`EstimatedRTT`公式进行计算。也就是说，该算法在重传时，会按照指数回退的原则更新超时时间。这里的出发点是：既然重传是因为超时引起的，那么相同的超时时间很有可能再次引起重传，为了避免陷入到无限的超时<-->重传的循环中，接下来应该更加谨慎的判定丢包了。（注，通过设置更长的超时时间）我们将在后面一种更加复杂的机制中再次见到指数回退。

### 4.1.3 Jacobson/Karels 算法

4.1.2中的算法提升了RTT的计算，但是本身并没有消除拥塞。1988年，由Jacobson和Karels提出的拥塞控制算法中，提出了一种新的方式来确定何时超时并重传一个segment。

最初的超时算法的主要问题是没有考虑到`SampleRTT`的波动。直观上来说，如果`SampleRTT`的波动很小，那么`EstimatedRTT`就可以被信任。如果`EstimatedRTT`可以被信任，也就没有必要对其乘以2来作为实际的超时时间。而另一方面，如果`SampleRTT`的波动很大，那么`EstimatedRTT`与实际的超时时间相差会较大。

在这一节的算法中，TCP的发送端还是如之前一样测量`SampleRTT`。之后采用下面的方式来计算超时时间。

<figure><img src="../../.gitbook/assets/image (12).png" alt="" width="375"><figcaption></figcaption></figure>

这里 δ位于0和1之间。在这里的公式中，我们同时计算了RTT的加权移动平均值（`EstimatedRTT`）和其波动的加权移动平均值（`Deviation`）。之后，TCP再根据下面的公式计算超时时间：

<figure><img src="../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt="" width="375"><figcaption></figcaption></figure>

基于经验，这里的μ通常设置为1，而φ通常设置为4。所以，当RTT的波动很小时，超时时间（`Timeout`）接近于`EstimatedRTT`，而波动很大时，`Deviation`会主导超时时间。

### 4.1.4 Implementation

在实现TCP超时时间的计算时，有两点需要注意。

1. 可以不使用浮点计算来得出`EstimatedRTT`和`Deviation`。如果将δ取值为1/2^n，那么所有的计算都可以是整数运算，乘法和除法可以通过移位实现，进而得到更高的性能（注，这里牺牲的就是精度）。如果取n=3（也就是δ=1/8），那么就可以通过下面的伪代码完成`Timeout`的计算。

```
{
    SampleRTT -= (EstimatedRTT >> 3);
    EstimatedRTT += SampleRTT;
    if (SampleRTT < 0) 
        SampleRTT = -SampleRTT;
    SampleRTT -= (Deviation >> 3);
    Deviation += SampleRTT;
    TimeOut = (EstimatedRTT >> 3) + (Deviation >> 1);
}
```

2. 算法的精度与系统的时钟相关。在算法提出时，一般的Unix系统的时钟精度最大有500ms，这远远大于跨（美）国的延时，也就是100-200ms。更糟糕的是，Unix中实现的TCP只会每500ms查看是否应该超时。所以超时可能会发生在segment送出之后的1s（注，两个时钟周期之后才判定超时）。下一小节中会介绍一个TCP的扩展来使得这里的计算更加精确。

（注，这里的两点都是强调，在具体实现 TCP 的超时时间计算时，不能达到十分精确）

有关TCP的超时时间的计算的更多内容，可以参考[RFC6298](https://tools.ietf.org/html/rfc6298)。

### 4.1.5 TCP Timestamp Extension

截止到目前为止的TCP的改动都还只是关于发送端如何计算其超时时间，并没有修改传输协议本身。现实中还有一些扩展用来提升TCP管理超时时间和重传的能力，它们分别是：

* 这一节我们将讨论一个与RTT计算相关的扩展。
* 另一个扩展是构建一个针对`AdvertisedWindow`的放大因子，这在2.3节介绍过了。
* 第三个是SACK，后面会讨论。

TCP的timestamp extension可以用来提升TCP的超时时间计算精度。TCP可以在发送segment的时候读取系统的时钟值，然后将这个32bit的时间放在TCP segment的header中。接收端在回复ACK时，将segment header中的时间，原封不动的放在ACK的header中。TCP的发送端在收到了ACK之后，将当前的时间减去header中的时间，就得到了RTT。本质上来说，TCP header中的timestamp option为TCP提供了一个方便的位置来存储segment是何时被送出的，这个时间就存在segment本身中。并且，TCP连接的两端并不需要同步时钟，因为timestamp只会在TCP的发送端读写。这个扩展提升了RTT的计算精度，进而提升了TCP超时时间的计算精度。

TCP timestamp extension还有第二个作用：它提供了一种逻辑的64bit sequence number的构成方法，解决了2.3节中提出的32bit sequence number带来的问题。原本的32bit sequence number比较容易溢出重新计数，所以存在一种可能：两个不同TCP连接的sequence number相同，进而导致数据错乱（注，这只可能发生在前一个连接有一个TCP segment在网络上耽搁了，然后前一个TCP连接终止了，新的TCP连接使用了相同的端口，碰巧使用的sequence number也跟网络上的耽搁的那个segment一样）。由于timestamp必然是递增的，它可以用来区分具备相同sequence number的两个TCP连接（注，网络上耽搁的那个segment的timestamp必然小于新连接的任何一个timestamp，所以TCP的接收端就可以丢弃这个小timestamp的segment）。
