# Chapter 1: 简介 未校正

互联网被认为是一项工程上的成功，几乎无与伦比且理所当然。它现在连接了数十亿个设备，支持所有能想象的到的通信程序，并且兼容从每天数十 bit 到每秒数百 Gigabits 的各种速率。但其核心是一个棘手的技术挑战，在过去30多年来一直受到包括，试图让互联网表现的更好的从业者和希望理解其数据基础的理论加 ，的广泛关注。这个核心的挑战就是：互联网资源如何最好地分配给所有竞争使用它的利益相关者。

对于任何计算机系统来说，资源分配都是个难题，对于像互联网这样一个复杂的系统来说，更是难上加难。这个问题在 1980 年代初 TCP/IP 协议栈刚刚部署时并不严重，但是到了 1980 年代末，随着互联网在大学中的重度使用（当时甚至还没有发明 World Wide Web），网络开始经历了一种叫做拥塞崩溃（congestion collapse）的现象。一种解决方案——拥塞控制——在1980 年代末开发并部署，立即解决了危机。从那以后，互联网社区就开始研究并优化拥塞控制方法。本书讲述了这段历程。

早期最著名的有关拥塞控制的工作由两位研究人员：Van Jacobson 和 Mike Karels 提出。相关论文：_Congestion Avoidance and Control_, published in 1988，是网络领域应用的最多的论文之一。论文之所以有这么高的影响力有很多原因：

* 当时拥塞崩溃的确威胁到了初生的互联网，而论文中的工作对于互联网的最终成功具有基础性意义。没有论文中的工作，我们不太可能有如今的全球互联网。
* 另一个原因是，拥塞控制在过去 30 年是一个硕果累累的研究领域。拥塞控制，或者更宽泛的说资源分配，有着广泛的设计空间以及足够的创新领域。几十年来的研究和实现都是基于早期的基础，并且可以合理的推断，针对已有方案的新方法和改进会继续出现，只要互联网还存在。

In this book, we explore the design space for congestion control in the Internet and present a description of the major approaches to managing or avoiding congestion that have been developed over the last three decades.

本书中，我们会探讨互联网中拥塞控制的设计选择，并展示过去 30 年间开发出来的管理拥塞和规避拥塞一些主要方法。

_更多阅读：V. Jacobson._ [_Congestion Avoidance and Control_](https://dl.acm.org/doi/10.1145/52324.52356)_. ACM SIGCOMM ‘88 Symposium, August 1988._
