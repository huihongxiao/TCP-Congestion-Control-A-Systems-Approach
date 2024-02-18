# 简介 by me

## 原文信息

**Title**: TCP Congestion Control: A Systems Approach

**Authors**: Larry Peterson, Lawrence Brakmo, and Bruce Davie

**Source**: https://github.com/SystemsApproach/tcpcc

**License**: [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0)



与原文有差异的地方：

* [4.5 NewReno](chapter-4-controlbased-tcp-yong-sai-kong-zhi-suan-fa/4.5-qi-ta-de-xiu-xiu-bu-bu-tcp-newreno.md)，有关NewReno原著介绍的太抽象了，但这又是一个很重要的TCP变种，我通过了一个具体的例子来介绍 NewReno 的传输过程。
* [4.1 TCP 超时时间计算](chapter-4-controlbased-tcp-yong-sai-kong-zhi-suan-fa/4.1-tcp-chao-shi-shi-jian-ji-suan/)，新增了一个子节，补全了Jacobson/Karels 算法，并详细介绍了 RFC6298 和 Linux RTO 的实现方法。
* [2.2 TCP流控](chapter-2-tcp-ji-chu/2.2-ke-kao-de-zi-jie-liu-reliable-bytestream/)，稍微展开介绍了一下如何打开关闭Nagle算法和Delayed ACK

## _声明_

_此次翻译纯属个人爱好，如果涉及到任何版权行为，请联系我，我将删除内容。文中所有内容，与本人现在，之前或者将来的雇佣公司无关，本人保留自省的权利，也就是说你看到的内容也不一定代表本人最新的认知和观点。_
