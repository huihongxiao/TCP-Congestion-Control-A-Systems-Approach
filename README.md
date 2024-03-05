# 简介 by me

## 原文信息

**Title**: TCP Congestion Control: A Systems Approach

**Authors**: Larry Peterson, Lawrence Brakmo, and Bruce Davie

**Source**: https://github.com/SystemsApproach/tcpcc

**License**: [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0)

## 为什么要翻译这本书

理由是有一天，我在想一个问题，是什么让TCP变得如此特殊，以至于其他所有的4层及以下常见的协议都不能与其相比？其他的协议，似乎只要略微一看就能大致明白其工作原理，但是对于TCP，就算非常了解协议格式，但总给人一种蒙着面纱的感觉。这是为什么呢？

我自己给的答案是，TCP的独特之处在于它有一套成熟运行的拥塞控制算法。那接下来，我就想系统的了解它的拥塞控制，以揭开TCP的面纱。于是我上网搜索，网上的文章很多，我也很快找到了很多碎片化的信息。比较常见的有介绍重传的，介绍窗口的，介绍慢启动的；一些大学的课程ppt里有介绍Tahoe，Reno，NewReno等等TCP的代号。这些信息不仅没有让我更了解TCP，反而让我更加焦虑了。这些信息对我来说只是一些零散的点，我知道它们之间一定有关联，并且只有把它们都关联起来，按照一个合理的逻辑去解释它们，我才能真正揭开TCP拥塞控制的面纱，但是要怎么做呢？

经过了几天的搜索之后，我找到了这本书，简单阅读之后，我知道这本书就是我的答案。这本书完整的覆盖了理解TCP拥塞控制所需要的全部知识，并以一种历史陈述，渐进的方式逐步讲解。或者说这本书本身就是有关TCP拥塞控制的知识平面，如果你也想了解TCP拥塞控制，那我的建议是就它了。

从最开始找到这本书，到历时1个月零5天，我终于全部读完了它。因为我觉得这是一本很好的书，所以我在阅读的过程中也翻译了它，并将翻译的内容开源到 [github](https://github.com/huihongxiao/TCP-Congestion-Control-A-Systems-Approach)。希望我的翻译可以帮助到有需要的人。与之前其他的翻译一样，我还是没有咬死原文，而是根据自己对于网络和TCP的理解，尽量贴合原文进行翻译。时间匆匆，难免有疏漏，欢迎指正。下面简单介绍一下各个章节的内容：

* 第一章：介绍了拥塞控制的起源，以及它对于互联网的重要性。可以说拥塞控制是当今计算机网络存在的必要条件。
* 第二章：虽然标题是TCP基础，但是并没有一些TCP的八股文，例如三次握手四次挥手，而是介绍了理解TCP拥塞控制所必需的前提知识，例如IP网络的传输模型，TCP的流控等。这些前提知识或多或少决定了TCP拥塞控制的实现方式。
* 第三章：从学术的角度分析了一个拥塞控制算法应该如何实现，如何评判，如何分析。从我的角度看，第三章 + 所有拥塞控制算法的归类，就是一个拥塞控制算法的综述。
* 第四章：介绍目前主流的拥塞控制算法的技术细节和历史发展。这一章非常重要，如果你比较心急，可以先看或者只看这一章。
* 第五章：介绍最近兴起的一类新的拥塞控制算法，或者严格来说是拥塞规避算法。这一章也非常重要，因为它有大火的BBR算法。
* 第六章：介绍网络设备如何参与到拥塞控制当中来。拥塞控制已经发展30多年，一直以来大家固有的印象这就是一个软件实现的算法，但实际上硬件也一直在努力做出一定的配合。但是硬件的演进比软件要难得多，所以这部分不是所有人都知道的很全，但是这一章有介绍。
* 第七章：介绍的是TCP之外的一些拥塞控制算法，例如大火的QUIC，音视频传输算法等等。这一章很好的结束了本书，因为它让我们看到除了TCP，其他领域只要是想在网络上进行传输，都或多或少要有某种形式的拥塞控制。这印证了第一章的内容，拥塞控制是计算机网络的必要条件，而这个必要关系不局限于TCP。

## 与原文有差异的地方

虽然本书非常优秀，但是我还是稍微补充了部分较为实用的内容，改动如下：

* [2.2 TCP流控](chapter-2-tcp-ji-chu/2.2-ke-kao-de-zi-jie-liu-reliable-bytestream/)，新增了一个子节，稍微展开介绍了一下如何打开关闭Nagle算法和Delayed ACK
* [4.1 TCP 超时时间计算](chapter-4-controlbased-tcp-yong-sai-kong-zhi-suan-fa/4.1-tcp-chao-shi-shi-jian-ji-suan/)，新增了一个子节，补全了Jacobson/Karels 算法，并详细介绍了 RFC6298 和 Linux RTO 的实现方法。
* [4.5 NewReno](chapter-4-controlbased-tcp-yong-sai-kong-zhi-suan-fa/4.5-qi-ta-de-xiu-xiu-bu-bu-tcp-newreno.md)，有关NewReno原著介绍的太抽象了，但这又是一个很重要的TCP变种，我在本节最后通过了一个具体的例子来介绍 NewReno 的传输过程。

## 其他翻译

如果你对计算机，分布式系统也比较感兴趣，不妨看看下面我的其他翻译。

* MIT6.s081 - 操作系统：[github](https://github.com/huihongxiao/MIT6.S081)；[gitbook](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/)
* MIT6.824 - 分布式系统：[github](https://github.com/huihongxiao/MIT6.824)；[gitbook](https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/)
* MIT6.829 - 计算机网络 （只有前三章，已停更）：[github](https://github.com/huihongxiao/MIT6.829)；[gitbook](https://mit-public-courses-cn-translatio.gitbook.io/mit6.829/)

## _声明_

_此次翻译纯属个人爱好，如果涉及到任何版权行为，请联系我，我将删除内容。文中所有内容，与本人现在，之前或者将来的雇佣公司无关，本人保留自省的权利，也就是说你看到的内容也不一定代表本人最新的认知和观点。_
