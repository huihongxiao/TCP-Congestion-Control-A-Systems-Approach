# Chapter 3: 如何设计拥塞控制 (Design Space)

在理解了 TCP/IP 的基础之后，我们可以继续探索如何设计方法来解决拥塞。但在此之前，我们先回过头来以一个更大的视角来看拥塞这个问题。互联网是一个复杂的组合体，它包括了计算、存储和网络资源，这些资源被数百万用户共享。现在的挑战在于如何分配这些资源，对我们来说，就是如何将交换机容量，缓存空间，链路带宽，分配给端到端的数据流。

因为互联网从最开始就使用一种尽力而为的传输模型，用户（或更准确地说，代表了他们的 TCP）可以自由地发送尽可能多的数据包到网络中，所以也不奇怪互联网最终会遭受“公地悲剧”（注："公地悲剧"是一个由英国经济学家加勒特·哈丁在1968年提出的概念。它描述了当多个个体共同使用有限资源时，由于每个个体追求自身利益而最终导致资源过度利用或破坏的现象。这一概念强调了个体利益与整体利益之间的矛盾，以及缺乏合作机制可能会导致资源无法持续利用的问题。公地悲剧常被用来探讨环境保护和资源管理领域的挑战，强调了合作与监管在解决共同资源管理问题中的重要性）。当用户开始经历拥塞崩溃时，自然的反应是尝试控制它。因此，术语“拥塞控制”应运而生，它可以被视为一种隐式的资源分配机制。它是隐式的，因为控制机制在检测到资源变得稀缺时会做出反应，并努力缓解拥塞。

资源显式的分配给每个数据流明显是另一种可用的网络传输模型，例如，应用程序在发送流量之前可以明确地请求资源。但是互联网协议 IP 在最开始提出时是尽力而为模型，当拥塞成为一个严重问题时，这个方案在当时并不是立即可用的。所以在解决拥塞问题或者资源分配问题时，一部分工作就是将更多的显示资源分配能力集成到互联网的尽力而为的传输模型中去，这其中就包括了提供_Quality-of-Service (QoS)_ 的能力。在理解互联网对拥塞的处理方式时，如果能有这些背景知识将会是有指导意义的。本章第一节便是这样做的，它探讨了本书概述的拥塞控制机制背后一系列设计决策。之后，本章将会定义可以量化评估和比较不同拥塞控制机制的标准。