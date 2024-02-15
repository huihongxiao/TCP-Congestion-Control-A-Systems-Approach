# Nagle 和 Delayed ACK

Nagle算法的本意是避免大量的小包降低整个网络的效率。因为所有的网络转发参与者，都需要通过解析 packet header 来完成转发，不管是大包还是小包，这里的代价是一样的。相应的，如果一个 packet 的数据越少，同样传输一段数据，需要的 packet 越多，消耗的网络资源也就越多。Nagel 算法利用了 TCP 发送端的 buffer 能力，确保数据量不大时，一个时刻只有一个“in-flight”的 packet；数据量大时，每个包都填满了 MSS。讲道理是个合理的算法。

但是 Nagel 算法也经常引起争议，因为在数据量不大时，必须要等待一个 RTT时间，TCP 的两端才能有一次交互，这相当于增加了应用程序的延时。

Nagel 算法还有个兄弟：Delayed ACK。Delayed ACK 的想法也很简单，因为 ACK 一般来说也是小包，那就攒一攒再回复给发送端，以减少网络中的小包数量。Delayed ACK 里面 Delay 的时间在 Linux 中，位于两个常量之间：[TCP\_DELACK\_MIN](https://github.com/torvalds/linux/blob/master/include/net/tcp.h#L138C9-L138C23)（(unsigned)(HZ/25)）和 [TCP\_DELACK\_MAX](https://github.com/torvalds/linux/blob/master/include/net/tcp.h#L134C9-L134C23)（(unsigned)(HZ/5)）之间，也就是 40ms 和 200ms 之间。具体数值取决于 RTT 的滑动平均值 (详见: [Linux 如何计算RTO](../../chapter-4-controlbased-tcp-yong-sai-kong-zhi-suan-fa/4.1-tcp-chao-shi-shi-jian-ji-suan/linux-ru-he-ji-suan-rto.md))。

虽然它们互为兄弟，但是它们两个在某些场合是冲突的。当数据量不大时，Nagle 会等待 ACK 来触发下一次发送，而 Delayed ACK 会增加 ACK 的返回时间，进而加剧 Nagle 算法带来的延时。

在 Linux 5.15 中，并未提供全局的开关来打开或者关闭 Nagel 算法和 Delayed ACK。而是在每个 TCP socket 中提供了 option 来控制他们。TCP\_NODELAY = 1 等于关闭 Nagel 算法，TCP\_QUICKACK = 1 等于关闭 Delayed ACK。
