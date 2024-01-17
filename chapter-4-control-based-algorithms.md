# Chapter 4: Control-Based Algorithms

这一章会介绍当今互联网上所使用的主流拥塞控制算法。该算法1988年由Van Jacobson和Mike Karels提出，并在之后的多年里几经改进。今天广泛所使用的算法版本被称为CUBIC，至于为什么叫这个名字，在本章结束之后大家就知道了。

该算法的的总体思路非常直观。TCP的发送端会根据来自于网络的两个信号，预估可用带宽，并根据预估的带宽来传输packets。这两个信号包括：

* 当TCP发送端收到了一个ACK时，这个ACK表明发送端之前送出的一个packet已经完成了网络中的传输。因此发送端可以在不增加网络负担的情况下，再传输一个新的packet。因为TCP通过使用ACK来控制数据包的传输，所以它也被认为具有自我校正的能力。
* 如果TCP在timeout之后还没有收到ACK，那表明一个packet已经丢失了，同时也表明网络现在正在拥塞，因此TCP的发送端要降低发送速率。使用丢包来表示拥塞表明拥塞已经发生了，这时TCP发送端是在事后反应，所以这种方式被称为“control-based”（而不是avoidance-base，因为拥塞已经发生，没法avoid）。

为了让这种算法具备实用价值，需要考虑很多细节。这一章会描述这些细节，因此这一章也可以被认为是识别并解决TCP拥塞控制算法中遇到的各种问题的case study。在接下来介绍的每个技术的过程中，我们都会回顾一些历史。

## 4.1 Timeout Calculation

超时和重传是TCP实现可靠字节流传输的核心部分。而超时在拥塞控制中还起了很重要的作用，因为超时代表了丢包，而丢包又表示现在极有可能处于拥塞状态。可以这样说，TCP的超时机制是实现拥塞控制必不可少的一个环节。

在实际中，超时可能有以下几种原因：

* 丢包
* ACK丢了
* 没有任何丢包，ACK就是单纯的比期望的回的慢

所以，到底花多少时间等待ACK尤为重要，否则的话，我们会误判当前是否处于拥塞（比如没有丢包，但是认为丢包进而认为拥塞了）。

TCP采用一种自适应的，根据测量获得的RTT作为参数的方法来设置timeout。尽管听起来很简单，但是完整的实现首先会比你预期的要复杂的多，其次在也已经经历过了多次改进。这一部分我们来回顾这段历程。

### 4.1.1 最初的算法

我们从最开始在TCP标准文档中的简单算法开始。它的核心思想就是保持计算RTT的运行平均值，然后根据这个值来计算超时。具体来说，每次TCP发送一个segment，它会记录发送的时间。当这个segment的ACK收到时，TCP再次读取时间，取差值作为`SampleRTT`。然后，TCP会计算`EstimatedRTT`，具体方法是将上一次的`EstimatedRTT`和新的`SampleRTT`求加权平均。

<figure><img src=".gitbook/assets/image.png" alt="" width="375"><figcaption></figcaption></figure>

这里的参数α就是加权值。α小一点的话，RTT的变化表现的更加明显，但是`EstimatedRTT`可能会受临时的波动影响。另一方面α大一些的话，`EstimatedRTT`会更加平稳，但是对于真实的网络质量的概念又不够灵敏。最初的TCP标准文档里建议设置α在0.8到0.9之间。TCP会使用一种非常保守的方式来根据`EstimatedRTT`计算超时时间：

<figure><img src=".gitbook/assets/image (1).png" alt="" width="253"><figcaption></figcaption></figure>

### 4.1.2 Karn/Partridge 算法

在最初的算法提出几年后，有个明显的缺陷被发现了：ACK并不确认一次传输，而是确认数据的接收。换句话说，当一个segment重传了，TCP的发送端收到了一个这个segment的ACK，它不能确定这个ACK应该和第一个还是第二个segment来计算`SampleRTT`。

为了得到一个精确的`SampleRTT`，还是需要知道这个ACK是第一个segment的，还是重传的segment的。如图21所示，如果认为ACK时原始的segment，但实际上它又属于重传的segment，那么算出来的`SampleRTT`就太大了（a）；但是如果认为ACK是重传的segment的，但是它又实际属于原始的segment，那么`SampleRTT`又太小了（b）。

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption><p>图21. 将ACK关联到（a）原始segment 和 （b）重传的segment。</p></figcaption></figure>

这里的解决方法还是很简单的，它就是Karn/Partridge 算法，以其发明者命名。它包含两个主要的改进：

1. 当TCP重传一个segment，它会停止采样RTT，也就是说TCP只会对没有重传的segment来测量`SampleRTT`。&#x20;
2. 当TCP重传一个segment时，它会将下一次的超时时间设置为当前超时时间的2倍，而不是基于上面的`EstimatedRTT`公式进行计算。也就是说，该算法在重传时，按照指数回退的原则更新超时时间。这里的出发点时：既然重传是因为超时引起的，并且重传对于EstimatedRTT的计算也没有帮助，那接下来就应该更小心的认为丢包发生了（通过设置更长的超时时间），而不是陷入到可能的不停的超时然后重传的循环中。我们将在后面一种更加复杂的多的机制中再次见到指数回退。\


