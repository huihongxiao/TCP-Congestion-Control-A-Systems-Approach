# Linux 如何计算RTO

（本节内容原文没有，译者自己添加）

[4.1](./) 中介绍了 TCP 是如何计算超时时间 RTO，以及其发展历史，整个过程很有启发性。但是有关[Jacobson/Karels 算法](https://ee.lbl.gov/papers/congavoid.pdf)部分内容并不完整，虽然不影响整体陈述，但是从严谨的角度这一节做一下补充。另外，从实用的角度，这一节也将介绍一下 Linux （基于 5.15 内核）中是如何计算 RTO。

## Jacobson/Karels 算法

首先，4.1 节中提出按照下面的方法计算超时时间 Timeout（即 RTO）

<figure><img src="../../.gitbook/assets/image (2) (1) (1) (1) (1).png" alt="" width="375"><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1) (1).png" alt="" width="375"><figcaption></figcaption></figure>

这里RTT 和其波动值的加权平均（分别对应 EstimatedRTT 和 Deviation）使用了相同的增益值δ = 1/8。这与[Jacobson/Karels 算法](https://ee.lbl.gov/papers/congavoid.pdf)的A.2 Practice 部分的前半段是一样的。但是算法在 A.2 Practice 的后半部分中同时也指出了，为了让波动值能更快的变化，可以给 Deviation一个更大的增益值，例如 1/4。

所以，A.2 Practice 的后半部分给出了另一个计算公式：

```
EstimatedRTT = EstimatedRTT + α * Difference
Deviation = Deviation + β * (|Difference| - Deviation)
Timeout = EstimatedRTT + 4 * Deviation
```

这里α = 1/8，β = 1/4。&#x20;

这很重要，因为这是 Linux 采用的公式，所以是更实用的一部分，但是原文遗漏了这部分。

## RFC6298

网络领域，论文一般不直接走向生产，而是先变成某种标准。论文一半是原理性论述，而在标准中会涵盖大部分的实现细节。基于标准，业界再去具体实现。而 [RFC6298](https://datatracker.ietf.org/doc/html/rfc6298) 就是体现了Jacobson/Karels 算法的最新版本标准。

RFC6298 包含以下内容：

* 按照下面的方法计算 RTTVAR 和 SRTT，这两个变量分别对应上面的 Deviation 和 EstimatedRTT，R'对应上面的 SampleRTT。虽然形式上略有不同，但是略微调整之后不难发现与原算法中A.2 Practice 的后半段是一样的，并且RFC6298 也建议 alpha = 1/8，beta = 1/4。

```
RTTVAR = (1 - beta) * RTTVAR + beta * |SRTT - R'|
SRTT = (1 - alpha) * SRTT + alpha * R'
```

* RTO，即 Timeout 按照下面方式计算。其中 K 等于 4，G 为一次时钟周期的时间，如果时钟中断每秒 1 次，那么 G 等于 1 秒。抛开 G，这里的 RTO 公式与原算法一样。而引入 G 是为了考虑实际系统中时钟的颗粒度。

```
RTO = SRTT + max (G, K*RTTVAR)
```

* 如果计算的 RTO 小于 1 秒，那么向上取整到 1 秒。1 秒是 RFC 所允许的最小 RTO。
* 对于一个 TCP 连接的最初始报文，也就是 SYN，因为还没有对 RTT 进行测量，所以拍脑袋将 RTO 设置为 1 秒。前一版标准RFC2988 中，这个值是 3 秒。
* 当收到第一个 ACK 时，将 SRTT 设置为当前测量的 RTT，RTTVAR 设置为 RTT/2，RTO 按照正常方式计算，也就是 SRTT + 4 \* RTTVAR = 3 \* RTT。此时，所有的变量都有了初始值，之后，SRTT 和 RTTVAR 按照公式进行计算。
* 包含了Karn 算法，即不考虑重传报文的 RTT，除非 Timestamp option 被引入。

从上面内容可以看出，RFC6298 不仅包含了Jacobson/Karels 算法，还包含了各种边界条件的处理，相比原算法，开发人员更容易基于 RFC6298 来实现 RTO 的计算。

## Linux 实现

Linux 的 TCP RTO 计算基于 RFC6298。虽然 RFC6298 已经足够详细的定义了很多细节，但是要落地一个具体的实现，还需要考虑到具体场景以及更多的细节。这里基于 5.15 版本来过一遍Linux 是如何计算其 RTO。其他版本 Linux 实现可能与下面的内容有些出入。

### 数据结构

有关 RTO 计算相关的数据结构在[struck tcp\_sock](https://github.com/torvalds/linux/blob/v5.15/include/linux/tcp.h#L256) 中。

```
struct tcp_sock {
/* RTT measurement */
	u64	tcp_mstamp;	/* most recent packet received/sent */
	u32	srtt_us;	/* smoothed round trip time << 3 in usecs */
	u32	mdev_us;	/* medium deviation			*/
	u32	mdev_max_us;	/* maximal mdev for the last rtt period	*/
	u32	rttvar_us;	/* smoothed mdev_max			*/
	u32	rtt_seq;	/* sequence number to update rttvar	*/
}
```

* tcp\_mstamp 用来记录上一次的时间戳，并用来计算 RTT 差值。
* srtt\_us 保存 SRTT 的值，它的单位是 us。注意，这里存储的是实际值的 8 倍。所以，当这里的值为 8 时，代表 1us。为什么要这样呢？是因为 alpha 为 1/8，而 SRTT 的计算要围绕着 alpha 进行，直接存储 8 倍值可以使得计算更加高效。当然代价是牺牲了代码的可读性。
* mdev\_us，mdev\_max\_us，rttvar\_us 都是用来计算 RTTVAR，且单位是 us。类似的，这里存储的都是实际值的 4 倍。所以当这里的值为 4 时，代表 1us。原因也上一条类似，因为 RTTVAR 的计算要围绕 beta 进行，而 beta 等于 1/4。
* rtt\_seq 也是用来计算 RTTVAR，具体后面介绍。

从数据结构可以看出，Linux 的实现相比于 RFC6298 又考虑了更多的细节且更加复杂。

### 函数调用

有关 RTO 计算的函数执行如下：

[tcp\_ack\_update\_rtt](https://github.com/torvalds/linux/blob/v5.15/net/ipv4/tcp\_input.c#L3058) -> [tcp\_rtt\_estimator](https://github.com/torvalds/linux/blob/v5.15/net/ipv4/tcp\_input.c#L820) -> [tcp\_set\_rto](https://github.com/torvalds/linux/blob/v5.15/net/ipv4/tcp\_input.c#L925)

#### tcp\_ack\_update\_rtt

首先根据 ACK 所包含的信息，例如 Timestamp option，来计算 RTT。之后将计算得到的 RTT作为参数（mrtt\_us）传递给 tcp\_rtt\_estimator。

```
static void tcp_rtt_estimator(struct sock *sk, long mrtt_us)
```

#### tcp\_set\_rto

这个函数会根据已有的信息计算 RTO。

```
static inline u32 __tcp_set_rto(const struct tcp_sock *tp)
{
	return usecs_to_jiffies((tp->srtt_us >> 3) + tp->rttvar_us);
}
```

主体代码如上所示，由于 srtt\_us 是实际值的 8 倍，rttvar\_us是实际值的 4 倍，这里的 rttvar\_us 已经包含了 max(G, 4 \* rttvar\_us) 的逻辑（G 定义在[TCP\_RTO\_MIN](https://github.com/torvalds/linux/blob/v5.15/include/net/tcp.h#L140)）。所以运算的结果就相当于

```
RTO = SRTT + max(G, 4 * RTTVAR)
```

这与RFC6298 的定义是一样的。

#### tcp\_rtt\_estimator

这个函数会计算 RTO 所需的数据。简化版的代码如下：

{% code lineNumbers="true" %}
```
static void tcp_rtt_estimator(struct sock *sk, long mrtt_us)
{
	struct tcp_sock *tp = tcp_sk(sk);
	long m = mrtt_us; /* RTT */
	u32 srtt = tp->srtt_us;
	if (srtt != 0) {
		m -= (srtt >> 3);	/* m is now error in rtt est */
		srtt += m;		/* rtt = 7/8 rtt + 1/8 new */
		if (m < 0) {
			m = -m;		/* m is now abs(error) */
			m -= (tp->mdev_us >> 2);   /* similar update on mdev */
		} else {
			m -= (tp->mdev_us >> 2);   /* similar update on mdev */
		}
		tp->mdev_us += m;		/* mdev = 3/4 mdev + 1/4 new */
		if (tp->mdev_us > tp->mdev_max_us) {
			tp->mdev_max_us = tp->mdev_us;
			if (tp->mdev_max_us > tp->rttvar_us)
				tp->rttvar_us = tp->mdev_max_us;
		}
		if (after(tp->snd_una, tp->rtt_seq)) {
			if (tp->mdev_max_us < tp->rttvar_us)
				tp->rttvar_us -= (tp->rttvar_us - tp->mdev_max_us) >> 2;
			tp->rtt_seq = tp->snd_nxt;
			tp->mdev_max_us = tcp_rto_min_us(sk);
		}
	} else {
		/* no previous measure. */
		srtt = m << 3;		/* take the measured time to be rtt */
		tp->mdev_us = m << 1;	/* make sure rto = 3*rtt */
		tp->rttvar_us = max(tp->mdev_us, tcp_rto_min_us(sk));
		tp->mdev_max_us = tp->rttvar_us;
		tp->rtt_seq = tp->snd_nxt;
	}
	tp->srtt_us = max(1U, srtt);
```
{% endcode %}

* 第 7 行。将测量的 RTT（注，mrtt\_us）与保存的 SRTT 值（注，srtt\_us >> 3，即srtt\_us / 8，为什么除 8？见数据结构处的解释）求差值，相当于得到了 R' - SRTT。
* 第 8 行。相当于 srtt = srtt + (测量 RTT 与 SRTT 的差值)。因为 srtt 为实际 SRTT 值的8 倍，所以相当于 srtt = 7 \* SRTT + 测量 RTT。两边都除以 8，可以得到与 RFC6298 一致的公式。至此，SRTT 的计算完成。

```
新的 SRTT = 7 / 8 旧的 SRTT + 1 / 8 测量的 RTT，即
SRTT = (1 - alpha) * SRTT + alpha * R'
其中 alpha = 1 / 8
```

* 第 9-15 行。计算测量 RTT 与 SRTT 差值的绝对值，并按照下面的公式更新 mdev\_us。

```
新的 mdev = 3 / 4 * 旧的 mdev + 测量 RTT 与 SRTT 差值的绝对值，即
新的实际的 mdev = 3 / 4 * 旧的实际的 mdev + 1 / 4 * 测量 RTT 与 SRTT 差值的绝对值
```

* 第 16-20 行。如果计算的 mdev\_us大于记录的最大 mdev\_max\_us，更新最大值。如果新的 mdev\_max\_us 大于 rttvar\_us，则 rttvar\_us 等于 mdev\_max\_us。在这个逻辑下，rttvar\_us = mdev\_us，相当于与 RFC6298 对应。

```
新的 RTTVAR = (1 - beta) * 旧的 RTTVAR + beta * 测量 RTT 与 SRTT 差值的绝对值
其中 beta = 1 / 4
```

* 第 21-26 行，用到了之前没有详细介绍的tcp\_sock 中的一个字段 rtt\_seq，以及 tcp\_sock 中的其他两个字段 snd\_una 和 snd\_nxt。snd\_una表示当前 TCP socket 期待下一个ACK 中确认的第一个字节序列号；snd\_nxt表示下一个要发送的序列号，rtt\_seq初始被设定为snd\_nxt。在第 21 行，当snd\_una大于rtt\_seq时，表示之前 snd\_nxt 对应的 segment 的 ACK 被收到了。这表示已经经过了一个 RTT。而这部分代码只在mdev\_max\_us小于rttvar\_us时，会减少 rttvar\_us。所以综合来看，第 16-20 行在遇到了大的 mdev\_us 时，会增加 rttvar\_us，这会发生在每次收到 ACK 时。而第 21-26 行，会判断mdev\_max\_us与 rttvar\_us 之间的关系，并尽量减少 rttvar\_us，这只会发生在每个 RTT 时间。更小的 rttvar\_us 表示更少的超时时间，即更激进的重传策略；而更大的 rttvar\_us 代表更加保守。所以这里的效果是每个 ACK 都尽量保守一点，在一个 RTT 的时间点尝试更激进点。这符合 TCP 的设计逻辑：激进的保守，保守的激进。
* 第 27-34 行，处理初始条件，即收到第一个 ACK 时初始化各个变量。这里与 RFC6298 的描述是一样的。

在tcp\_rtt\_estimator中调用了两次tcp\_rto\_min\_us。这个函数的过程略微有点复杂，但是可以认为默认情况下会返回[TCP\_RTO\_MIN](https://github.com/torvalds/linux/blob/v5.15/include/net/tcp.h#L140)，等于((unsigned)(HZ/5))，即 200ms。而 TCP 连接新建时，SYN 包的超时时间定义在[TCP\_TIMEOUT\_INIT](https://github.com/torvalds/linux/blob/v5.15/include/net/tcp.h#L142C9-L142C25)，等于((unsigned)(1\*HZ))，即 1 秒，这与 RFC6298 定义的一样。所以Linux 中， TCP SYN包的超时时间是 1 秒，而数据包的超时时间略大于 200ms。

最后引用tcp\_rtt\_estimator中的一段[注释](https://github.com/torvalds/linux/blob/v5.15/net/ipv4/tcp\_input.c#L835-L840)，写这段代码的人都这么说了，我们研究半天是研究了个寂寞╮(╯▽╰)╭

> ```
>  * Funny. This algorithm seems to be very broken.
>  * These formulae increase RTO, when it should be decreased, increase
>  * too slowly, when it should be increased quickly, decrease too quickly
>  * etc. I guess in BSD RTO takes ONE value, so that it is absolutely
>  * does not matter how to _calculate_ it. Seems, it was trap
>  * that VJ failed to avoid. 8)
> ```
