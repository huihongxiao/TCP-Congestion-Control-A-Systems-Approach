# Chapter 5: Avoidance-Based TCP拥塞控制算法

如果查看TCP拥塞控制的学术文献，1988年和1990年分别引入的原始TCP Tahoe和Reno机制，与1994年开始的一波新潮流之间存在明显差距。1994 年这一波新潮流的标志是一种称为TCP Vegas的拥塞控制算法的提出。它在接下来超过 25 年间，触发了大量的比较性研究和不同的设计。

_更多阅读：L. Brakmo, S. O’Malley and L. Peterson_ [_TCP Vegas: New Technique for Congestion Detection and Avoidance_](https://sites.cs.ucsb.edu/\~almeroth/classes/F05.276/papers/vegas.pdf)_. ACM SIGCOMM ‘94 Symposium. August 1994. (Reprinted in IEEE/ACM Transactions on Networking, October 1995)._

之前的所有 TCP 拥塞控制算法都将丢包看作拥塞信号，并反应式的进行拥塞控制（control-based）。TCP Vegas采用的是一种基于规避（avoidance-based）的方法来应对拥塞：它尝试探测吞吐速率的变化，并在拥塞变得严重到导致丢包前，调整发送速率，进而避免丢包的发生。这一章描述通用的“Vegas 策略”，以及基于这个策略的 3 个变种实现。 最后介绍这一类算法最为出众的代表： Google 推出的 BBR 算法。
