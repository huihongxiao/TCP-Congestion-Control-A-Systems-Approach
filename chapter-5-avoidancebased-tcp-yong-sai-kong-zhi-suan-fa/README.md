# Chapter 5: Avoidance-Based TCP拥塞控制算法

如果查看TCP拥塞控制的学术文献，1988年和1990年分别引入的原始TCP Tahoe和Reno机制与1994年开始的一波新潮流之间存在明显差距，这一波新潮流的标志是一种称为TCP Vegas的替代方法的引入。它触发了大量的比较性研究和不同的设计，并且持续了 超过25 年。

_更多阅读：L. Brakmo, S. O’Malley and L. Peterson_ [_TCP Vegas: New Technique for Congestion Detection and Avoidance_](https://sites.cs.ucsb.edu/\~almeroth/classes/F05.276/papers/vegas.pdf)_. ACM SIGCOMM ‘94 Symposium. August 1994. (Reprinted in IEEE/ACM Transactions on Networking, October 1995)._

之前的所有 TCP 拥塞控制算法都将丢包看作拥塞信号，并反应式的进行拥塞。TCP Vegas采用的是一种基于规避（avoidance-based）的方法来应对拥塞：它尝试探测吞吐速率的变化，并在拥塞变得严重到导致丢包前，调整发送速率，进而避免丢包的发生。这一章描述通用的“Vegas 策略”，以及基于这个策略的 3 个变种实现。这一类算法的研究，以 Google 推出的 BBR 算法最为出众。
