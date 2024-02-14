# 简介

原文信息：

**Title**: TCP Congestion Control: A Systems Approach

**Authors**: Larry Peterson, Lawrence Brakmo, and Bruce Davie

**Source**: https://github.com/SystemsApproach/tcpcc

**License**: [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0)



与原著差异的地方：

* NewReno，原著介绍的太抽象了，我通过了一个具体的例子来介绍 NewReno 的传输过程。
* TCP 超时时间计算，新增了一个子节，补全了Jacobson/Karels 算法，并介绍了 RFC6298 和 Linux RTO 的实现方法。
