# Chapter 4: Control-Based TCP拥塞控制算法

这一章会介绍当今互联网上所使用的主流TCP 拥塞控制算法。TCP 拥塞控制算法最开始在1988年由Van Jacobson和Mike Karels提出（注，TCP Tahoe），并在之后的多年里几经改进。今天被广泛使用的算法版本被称为CUBIC，为什么叫这个名字呢？在本章结束的时候大家就知道了。

TCP 拥塞控制算法的总体思路很直观。TCP的发送端会根据来自于网络的两个信号预估可用的网络带宽，再基于这个预估的可用网络带宽来传输packets。来自网络的两个信号包括：

* 当TCP发送端收到了一个ACK时，这个ACK表明发送端之前送出的一个packet已经完成了网络中的传输。因此发送端可以在不增加网络负担的情况下，再传输一个新的packet。因为TCP通过使用ACK来控制数据包的传输，所以它也被认为具有自我校正的能力。（注，虽然这里提的是 packet，但是 packet，frame，segment 本质含都是一个东西在不同纬度的表现。在后面提到的 segment 等同于这里的 packet。）
* 如果TCP在设定的超时时间之后还没有收到ACK，那表明一个packet已经丢失了，同时也表明网络现在正在拥塞，因此TCP的发送端要降低发送速率。丢包表示网络拥塞已经发生，TCP发送端根据丢包来响应是一种事后反应，这种方式被也称为“control-based”（注，与之相对的是avoidance-based。两者的区别在于，control-based算法是在拥塞已经发生时做出动作，此时拥塞已经发生，没法avoid了）。

TCP 拥塞控制算法的实现需要考虑很多细节。这一章会描述这些细节，因此这一章也可以被认为是识别并解决TCP拥塞控制算法在实现过程中遇到的各种问题的case study。我们将通过回顾历史的方式，来介绍接下来的每个技术。

