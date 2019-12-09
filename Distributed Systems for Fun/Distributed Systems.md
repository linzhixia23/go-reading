原文： http://book.mixu.net/distsys/ebook.html


### 简介

我想写一篇文章介绍当前主流的分布式系统背后的思想，比如：亚马逊的Dynamo，谷歌的Bigtable和MapReduce，Apache的Hadoop等。

这篇文章里，我会尝试用更浅显易懂的方式介绍分布式系统。
对我来说，这包含两层意思：一是，介绍关键的概念，以便你将来能够阅读更加深入的内容；二是，尽可能包含主要的内容，使得你能掌握其中的精髓，而不要深陷到其中的细节中。
现在都2013年了，你可以通过互联网选择性阅读你最感兴趣的内容。

在我看来，大多数的分布式编程其实是在解决两个分布式带来的影响：

* 信息以光速传播
* 任何东西会在任意时候失败 

换句话说，分布式编程的核心是解决距离的问题，以及处理一个以上的东西。
这些限制决定了系统设计的空间，我希望你读完这篇文章后，能够对距离、时间和一致性模型如何相互影响有更好的认识。

这篇文章关注于分布式编程，以及系统概念，这有助于你理解数据中心的商业系统。
试图涵盖所有内容是不切实际的。你只要学习关键的协议和算法，包括一些大学教材没有的东西，如CRDT和CALM理论。

后面的内容是这样组织的：

* 第一节介绍分布式系统的关键术语和概念。它包含高阶的目标，例如：可扩展性、可用性、高性能、延迟和容错性；
这些目标如何难以达成，抽象和模型（如分区和副本）如何一起协同工作等。

* 第二节更加深入介绍抽象。它先介绍Nietzsche quote，然后介绍系统模型，以及典型的系统模型基于的假设。
还会讨论CAP理论，总结FLP的可能结果。然后再回到CAP理论的影响，其中一个就是，我们需要去探索其他的一致性模型。
最后会讨论一些一致性的模型。

* 理解分布式系统的很重要的部分是理解时间和顺序。一定程度上讲，如果我们没有理解对时间建模，我们的系统就会失败。
第三节讨论了时间、顺序和时钟的各种用法。

* 第四节介绍副本（replication）的问题，以及它的两种常用的使用方式。大多数相关的特种都可以在这样一个简单的特征下进行讨论。
接下来讨论保持single-copy一致性的副本的方法，包括从最简单的容错算法2PC到Paxos。

* 第五节讨论弱一致性条件下的副本技术。它介绍一个简单的分区的副本能够达成一致，协同工作的场景。
介绍以亚马逊的Dynamo，作为弱一致性系统设计的案例。最后，讨论两个无序编程（disorderly programming）的观点：CRDT和CALM理论。


### 1、分布式系统概述

分布式编程就是，把单机上解决的同样的问题，放到多机环境下去解决的艺术。

它包含了两个计算机系统需要去解决的基本任务：

* 存储
* 计算

本质上，没有任何事情要强制你使用分布式系统。只要有无限的金钱和R&D的时间，其实并不需要分布式系统。
所有的计算和存储可以在魔法盒里完成--你可以付钱让人帮你设计这样一个高速高可靠的系统。

然而，很少有人会有无限的资源。因此，你需要在投入和收益上找到平衡点。对于小规模的问题，升级硬件是一个可行的方案。
但是，随着问题规模变大，你会发现通过在单点上升级硬件已经不能解决问题，或者是代价太大了。
此时，我们欢迎你来到分布式系统的世界。

（It is a current reality that the best value is in mid-range, commodity hardware - as long as the maintenance costs can be kept down through fault-tolerant software.）
（不会翻译）

计算主要受益于高端硬件，它可以用内部的内存访问替代缓慢的网络访问。
高端硬件的优势受到任务数量的影响，因为它需要节点之间进行大量的通讯。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/1_cluster_size.png)

从上图可以看到，高性能硬件和普通硬件的的差距随着集群的增大而减少，当然，这里假设这两种模式下，内存之间的访问是均等的。

理想情况下，增加一台新的机器会线性地增加系统的性能和容量。
当然，实际上这是不可能的，因为计算机之间是相互分开的，这样会带来额外的开销。数据需要拷贝，计算任务需要调度，等等。
这也是为什么我们要学习分布式算法--它给对特定的问题提出有效的解决方案，同时能够给出指导：什么样是能做到的，什么样是不可能的，以及正确的实现的最小代价。

这篇文档关注于常见的分布式编程和系统，但他们跟商业场景相关联：数据中心。 
例如，我不会讨论特定的问题，像原子网络的配置，或者共享内存设置。
我们关注于探索系统设计的空间，而不是优化特定的设计--后者应该是更加专业的书籍的主题。

#### 我们期望的是：可扩展性以及其他的好处

我的看法是，任何问题都开始于处理大小（原话是，The way I see it, everything starts with the need to deal with size）。

大多数问题在规模小的时候都很简单，但是只要超过了一定大小、容量或者其他物理的限制，一切问题就变得困难。
就好比，举起一块巧克力很容易，但是举起一座山却很难。统计房间里有多少人很容易，统计一个国家有多少人却很难。

所以，任何事情都开始于大小，也就是可扩展性。通俗的讲，在一个高扩展性的系统里，随着它的规模从小到大，它不会因此而恶化。
以下是它的另一个定义：

Scalability is the ability of a system, network, or process, to handle a growing amount of work in a capable manner or its ability to be enlarged to accommodate that growth.

增长指什么？增长可以指大多数的东西（用户数量、电力使用等）。不过，这里有三个有趣的值得关注的地方：

* 大小可扩展：系统运行的速度应该随着增加的节点数线性地增长；增加数据集并不会增加系统的延时。

* 地理可扩展：它可以用多数据中心来减少回应用户请求的时间，同时能用合理的方式处理跨数据中心的延时。

* 管理可扩展：增加更多的节点不应该增加系统管理的代价（例如，管理员与机器的比例）。

当然，在实际系统中，增长会同时出现在不同维度；每一个指标都反映增长的不同方面。

一个扩展性的系统就是能够随着规模的扩大而不断满足用户的需求。
有两个关键的指标：性能和可用性，它们可以用很多方式来评价。

#### 性能&&延迟 

性能的定义：

Performance is characterized by the amount of useful work accomplished by a computer system compared to the time and resources used.

根据上下文，它包含了达到下述一个或多个目标：

* 对给定的一批任务，能够以较短的时间返回请求
* 高吞吐量
* 私用较少的计算资源 

优化这些结果有很多折中的方式。例如，系统可以通过批量处理任务来极少额外开销，以达到高吞吐量。
当然，批量处理会导致单个任务的响应时间变长。

我发现，低延时是性能优化方面最有意思的一部分，因为它跟硬件限制紧密相关。
它跟性能优化的其他方面不同，它很难通过增加财务资源（financial resource）来解决。

对延迟（latency）有很多定义，但我最喜欢的是从词源上的定义：

Latency: The state of being latent; delay, a period between the initiation of something and the occurrence.

那么，"latent"的意思是什么呢？ 

Latent:From Latin latens, latentis, present participle of lateo ("lie hidden"). Existing or present but concealed or inactive.

这个定义非常酷，因为它强调了延迟是某件时间发生的时机和它真正产生影响的时间。

试想，你被病毒感染了，它会把人变成僵尸。它的潜伏期就是你被感染的时间到你成为僵尸的时间。
这就是延迟：一个事情已经发生的时间，到它显现的时间。

假设一个场景，我们的系统做一个high-level的任务：给定一个查询，它会基于数据的所有数据，计算出一个结果。
换一句话说，设想一个分布式系统，它能够基于当前存储的数据计算出一个确定性的结果：

result = query(all data in the system)

因而，影响延迟的并不是旧的数据，而是新的数据对系统产生影响的速度。
例如，延时应该根据从写入数据，到它真正能够被读取到的时间来衡量。

这个定义的另一个关键点是，如果没有任何事情发生，那么它也就没有"延迟周期"（latent period）。如果系统的数据没有发生变化，那么它就没有（也不应该有）延迟的问题。

对分布式系统来说，它有一个无法克服的最小延迟时间：光速决定了信息传递的速度，以及硬件组件在每次操作的延迟。

这个最小延时对你的影响取决于查询的问题本质，以及信息传输的实际物理距离。

#### 可用性&&容错性

可扩展系统的另一个考量是可用性。

Availability:the proportion of time a system is in a functioning condition. If a user cannot access the system, it is said to be unavailable.

分布式系统可以让我们做很多之前在单机系统上做不到的事。例如单机无法做容错，它只有两个状态：失败或者没有。

分布式系统可以基于一堆不可靠的组件，构建一个可靠的系统。

一个没有备份（redundancy）的系统，它的可用性只取决于他们潜在的组件。如果它有备份，那么它就可以容忍部分组件的失败，使得组件更加可靠。
备份可以是任何东西，组件、服务器、数据中心等。

可用性可以公式表示为： Availability = uptime / (uptime + downtime) 

可用性在技术视角更多是关于容错性。因为随着组件的增加，出现错误的概率也会增加，系统应该不能使可用性降低。

例如：

| Availability% | How much downtime is allowed per year? |
| :----: | :----: |
| 90%("one nine") | More than a month |
| 99%("two nines") | Less than 4 days |
| 99.9% ("three nines") | Less than 9 hours |
| 99.99% ("four nines") | Less than an hour |
| 99.999% ("five nines") | ~ 5 minutes |
| 99.9999% ("six nines") | ~ 31 seconds |

可用性的概念其实比运行时间更加宽泛，因为一个服务的可用性还会受到网络中断的影响--尽管它跟系统的可容错性并没有直接的关系，但是这确实会影响到系统的可用性。
但是除了了解系统的每一个特定的细节，我们能做的就是设计一个可容错的系统。

那么，可容错意味着什么呢？ 

Fault tolerance：ability of a system to behave in a well-defined manner once faults occur

可容错性可以归结为：定义你认为会出现的错误，并设计系统或者算法去容忍它们。
你不可能对毫无预期的错误做容错。

#### 哪些因素会阻止我们得到好的结果？

分布式系统会受到两个物理因素的限制：

* 节点数量（它的增加会导致要求的存储和计算能力随之增加）

* 节点之间的距离（信息的传输最快就是光速）

因而，会导致这些限制：

* 增加独立的节点的数量会增加系统出现错误的概率（降低可用性，增加管理成本）

* 增加独立的节点的数量会增加节点直接的通信需求（性能会随着规模增加而降低）

* 增加地理距离，会增加通信的最小延时（特定的操作会降低性能）

除了这些由于物理限制导致的问题，其他都是系统设计的范围。

高性能和高可用是由外部保证（external guarantee）来定义的。
在高阶的视角，你可以把这个保证理解为SLA（service level agreement）：如果我写了一个数据，我在其他地方可以多快访问到它？
在我写了之后，我能得到什么样的持久性（durability）的保证？如果我让系统运行计算，它多久可以返回结果？
当组件失败或者操作失败时，对系统的影响是什么？ 

还有一些其他的准备，没有显示地提出：可理解性（intelligibility）。
这个保证（guarantee）有多少可理解性？当然，对于可理解性并没有简单的度量指标。

（以下这一段不太理解，直接贴原文）。I was kind of tempted to put "intelligibility" under physical limitations. 
After all, it is a hardware limitation in people that we have a hard time understanding anything that involves more moving things than we have fingers. 
That's the difference between an error and an anomaly - an error is incorrect behavior, while an anomaly is unexpected behavior. 
If you were smarter, you'd expect the anomalies to occur.

#### 抽象和建模 

这就是抽象和建模在一起工作的方式：抽象通过把现实世界中与解决问题无关的部分剥离出来，使得问题更容易管理；建模用准确的方式描述分布式系统的关键属性。
我会在下一节讨论很多种模型，例如：

* 系统模型（同步/异步）
* 失败模型（crash失败，数据分片，Byzantine等）
* 一致性模型

一个好的抽象使得系统更加容易理解，因为它获取了跟目标相关的要素。

现实的系统有很多个节点，而我们设计的目标是让它们"像一个系统一样工作"。
通常，大多数熟知的模型代价都很高。例如，在分布式系统上实现一个共享内存。

一个系统如果是弱一致性保证（weker guarantee），它会有更多的自由度，它可能会有更好的性能--不过这个很难从推导来证明。
人们更习惯思考一个单一系统，而不是多个节点的系统。

通过展示内部系统的更多细节，人们可以更好地优化性能。
例如，在列存储中，用户一定程度上如果知道key-value的存储方式和数据局部性（locality），那么就可以对特定查询做出有影响的决策。
系统隐藏一些细节会更容易理解，而暴露更多现实的细节则有利于做出优化。

一些错误会使得让分布式系统像一个单一系统一样工作变得困难。
网络延时和网络分片意味着系统有时要做一些艰难的决定：是否要继续保持可用性，但是没法做一些关键性的保证（lose some crucial guarantees），还是为了安全性，在一些错误发生时拒绝客户端请求。

我下一节要讲的CAP理论，涉及了上述的一些问题。
最后要说的是，理想的系统要同时满足程序员的需求（清晰的语法）以及商业要求（可用性、一致性、延迟性）。

#### 设计技术：数据分片和副本

数据集以什么样的方式分布式放在不同节点这个问题很重要。
为了应付任何类型的计算，我们需要放置数据，并应用它。

有两种基本的方式可应用于数据集。
一种是，把数据分成多个节点存储，这样有助于并行处理；另一种是，把数据复制到不同节点，以减少客户端和服务端的距离，同时还有利于容错。

分而治之--我的意思是，数据分片和复制（partition and replicate）

下图展示了两者的区别：数据分片是把数据分成相互独立的集合（如A和B），而数据副本则是把数据复制放在不同地方（如C）。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/1_partition_and_replicate.png)

这种one-two punch的方式可以解决任何分布式计算的问题。
当然，其中的trick是你要选择合适的技术用于你具体的实现；有很多算法可实现分片和副本，每个都有不同的限制和优势，你要能根据你的设计目标做出抉择。

#### 数据分片 

数据分片是把数据集分成相互独立的小的集合；它用于减少数据增长带来的影响，因为每一个分片都是数据的子集。

* 数据分片可以提高性能，通过限制要运算的数据规模，以及把相关的数据放在同样的分区。

* 数据分片可以提高可用性，它可以使得个别节点失败而不影响其他节点，以增加系统可以允许节点失败的个数。

数据分片是跟实际应用场景非常相关，所以没有具体的应用，很难明确说明要把数据分成多少份。
这也就是为什么大多数的教材把笔墨放在数据副本上，当然我们也一样。

数据分片主要是基于你的系统主要访问的模式来分片，并处理由不同分片导致的限制。
例如，跨分片访问的效率问题、不同分片增长速度不同的问题。

#### 数据副本（Replication）

数据副本是指在不同机器拷贝相同的数据；这样可以让更多的服务器参与计算。

这里引用 Homer J.Simpson的原话：

To replication! The cause of, and solution to all of life's problems.

数据副本是我们应对延迟（latency）的主要方式。

* 数据副本可以提高性能，因为复制数据可以使用到更多计算资源（computing power）和带宽。

* 数据副本可以提高可用性，它创建数据的拷贝，以增加系统可以允许节点失败的个数。

数据副本主要关于提供了额外的带宽和缓存，以及对一些一致性模型用一些方式来保持一致性。

数据副本让我们实现可扩展性、性能和容错。
担心失去可用性和降低性能？数据副本可以让我们避免单点失败以及瓶颈。
计算速度慢？数据副本可以支持在多个系统上进行计算。
I/O速度慢？把数据复制到本地cache，可以减少延迟，或者利用多个机器来增加吞吐量。

数据副本会引起很多问题，因为不同机器上的相互独立的数据之间要通信同步数据，这就意味着要遵守一个一致性模型。

一致性模型的选择至关重要：好的模型更容易理解它为什么能保证一致性，而且能达到商业和设计上的目标，例如高可用性、强一致性等。

强一致性模型是唯一能够让你在写代码时，可以认为底层数据是没有副本的。
其他的一致性模型，或多或少都把内部的副本的原理暴露给开发者。
不过，弱一致性模型可以提高更低的延时和高可用性，而且容易理解。

#### 延伸阅读

* [The Datacenter as a Computer - An Introduction to the Design of Warehouse-Scale Machines - Barroso & Hölzle, 2008](https://www.morganclaypool.com/doi/pdf/10.2200/s00193ed1v01y200905cac006)

* [https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing)

* [Notes on Distributed Systems for Young Bloods - Hodges, 2013](https://www.somethingsimilar.com/2013/01/14/notes-on-distributed-systems-for-young-bloods/)


### 2、向上和向下抽象（Up and down the level of abstraction）

在这一节，我们要探索向上和向下的抽象，审视一些理想的模型（CAP和FLP），然后考虑如何在性能上做折中。

如果你熟悉任何编程语言，那么抽象的层次对你来说可能很熟悉。
你可能在一些抽象层级上工作，通过API与低的层级交互，可能提供一些高层级的API或者用户接口。
网络的7个层级就是一个很好的例子。

我说过了，分布式编程就是大多数时间在处理分布式带来的后果。
我们的现实是，分布式系统有很多个相互作用的节点，而我们的愿望是让它们"像一个系统一样工作"，这两者之间有一定鸿沟。
也就是说，我们要在它能够做成什么样，以及它的理解性和性能之间取得平衡。

当我们说X比Y更加抽象时，是什么意思呢？
首先，X在本质上与Y没有不同。事实上是，X移除了Y的一些特点，或者用另一种方式来展示，使得它更加容易管理。
其次，在一定程度上，X比Y更容易理解，当然，这里的假设前提是，X从Y移除的一些特性无关重要。

正如尼采写的：

Every concept originates through our equating what is unequal. 
No leaf ever wholly equals another, and the concept "leaf" is formed through an arbitrary abstraction from these individual differences, through forgetting the distinctions;
and now it gives rise to the idea that in nature there might be something besides the leaves which would be "leaf" - some kind of original form after which all leaves have been woven, marked, copied, colored, curled, and painted,
but by unskilled hands, so that no copy turned out to be a correct, reliable, and faithful image of the original form.

抽象本质上就是假的。每一个环境都是独特的，正如每一个节点。
但是抽象让世界更容易管理：简化问题，使得更容易分析，而且没有忽略任何关键的东西，而结果广泛适用。

如果我们保留的事情是必须的，那么我们从中得到的结果就会是广泛适用的。
这也就是为什么不可能的结果如此重要：它们提取了问题中最简单的形式，并展示说明了在一定限制和假设的情况下，它不可能解决。

抽象是让事物等价，忽略现实中并不本质的东西。
（原话是：All abstractions ignore something in favor of equating things that are in reality unique）。
我们能用的trick就是剥离出不是必须的部分。
那么你怎么知道什么是必须的呢？或许你没办法提前知道。

每次我们从特定的系统中忽略一些特性，就可能会引入新的错误，或者性能问题。
这也就是为什么有时需要换一种思路，重新引入新的硬件特性或者物理特性，使得系统的性能更好。

一个系统模型是我们认为系统中重要的特定的特征；有一个特定的之后，我们就可以再寻找一些不可能的结果和挑战。

#### 系统模型

分布式系统的一个关键的属性就是分布式。更具体地说，分布式程序有以下特性：

* 同时在相互独立的节点上并发运行

* 通过一个非确定性的有信息丢失的网络链接

* 没有共享的内存和共享的时钟

还有很多其他的：

* 每一个节点并发执行程序

* 知识存在本地：节点只有访问本地状态时才时最快的，任何关于全局的信息可能已经过期了

* 节点可以失败，并相互独立地从失败中恢复 

* 信息可以延迟或丢失（很难区分出是节点失败了，还是网络延迟了）

* 时钟不能跨节点同步（本地时间戳不等于全局的真实的时间顺序，而全局的时间是很难观察到的）

一个系统的模型列举了很多关于特定系统设计的假设。

系统模型的定义：

System model : a set of assumptions about the environment and facilities on which a distributed system is implemented

系统模型会有很多关于环境和设备的不同假设。具体包括： 

* 节点的能力，以及它们是如何失败的

* 通信连接是怎么工作的，以及它们是如何失败的 

* 所有系统的属性，例如时间和顺序

一个鲁棒的系统模型是基于最弱的假设的：它的任何算法在任何不同环境下容错性都很高，因为它们做了尽可能少的假设。

另一方面，如果我们做的强假设，我们的系统模型会容易理解。
例如，假设节点不会失败，这就意味着我们的算法不需要处理节点失败。
不过这样的系统模型并不现实，因此很难应用到实践中。

我们来仔细看一下节点的属性、时间、顺序等问题。

#### 系统模型的节点 

节点是指用于计算和存储的host，它们通常有以下特征：

* 能够执行程序

* 能把数据存储在内存（volatile memory）和数据持久化（数据在失败后还能读取）

* 有时钟（可能不能认为它是准确的）

Node执行确定性的算法：本地计算、本地计算之后的状态、以及消息发送唯一取决于收到的消息和收到消息时候的本地状态。

有很多可能的容错模型（failure model）描述节点失败的方式。
实践中，大多数系统会假设一个crash-failure的容错模型：节点只会因为crash导致失败，而且在某些情况下能够恢复。

另一个选择是，假设节点会以任何异常的行为导致失败。
这就是所谓的(Byzantine fault tolerance)[https://en.wikipedia.org/wiki/Byzantine_fault]。
在实际的商业系统很少处理拜占庭错误（Byzantine fault），因为算法要处理任意类型错误的代价太大，而且实现上也非常复杂。
我在这里不讨论这类错误。

#### 系统模型的通信连接

通信连接（communication link）把任意节点连接在一起，并可以让任意两个点直接发送消息。
大多数介绍分布式算法的书籍都会假定，任意两个节点直接都有连接，连接直接的消息传播是FIFO的，它们只能传递发送的消息，而且发送的消息可能丢失。

一些算法假定网络是可靠的：消息永远不会丢失，也不会无限期地延迟。
在现实的一些情景下，这样的假设是合理的，但是从通用的角度，最好是认为网络不可靠，会有消息丢失和消息延迟的限制。

当网络出现失败，而节点还在运行时，就会出现网络隔离（network partition）。
当它出现的时候，消息可能会丢失或者延迟，直到网络隔离修复。
被分离的节点可能会被一些客户端访问，所以它跟crash的节点是不一样的。
下面的图展示了节点失败和网络隔离的区别。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/2_node_failure_vs_network_partition.png)

关于网路连接，很少有更多的假设了。
我们可以假设网络是单向传播的，或者我们可以假设不同连接之间有不同的传输代价。
但是，在商业系统中，很少关心这些，除非是长距离的连接（WAN延迟），所以我在这里不会讨论这些；
对代价和拓扑结构做更加细致的建模，可以用更高的复杂度来获得更好的优化。

#### 时间/顺序的假设

物理上分离的一个结果是，每一个节点都以各自的方式进行工作。这是不可避免的，因为信息最多只能以光速传递。
如果节点之间的距离都各自不同，那么从一个节点出发的信息，会在不同的时间到达其他节点，而且顺序也可能不同。

时间的假设（timing assumption）是现实中很容易考虑到的问题。（原文是：Timing assumptions are a convenient shorthand for capturing assumptions about the extent to which we take this reality into account）
两个主要的选择是：

* 同步系统模型：进程以lock-step方式处理；它们有一个已知的信息传输的最大时延；每一个进程都有一个准确的时钟。

* 异步系统模型：没有时间的假设；每一个进程以相互独立的速率运行；它们对信息传播时延没有限制；没有时钟。

同步系统模型对时间和顺序强加了很多限制。
它们本质上假设节点的情况相同：发送的信息总是在特定的最大传输时延里收到，以lock-step方式处理。
这样处理比较方便，因为它允许系统设计者对时间和顺序做出假设，而异步的系统模型则做不到。

异步性不是一个假设：它只是假设你不能太依赖时机。（原话是：Asynchronicity is a non-assumption: it just assumes that you can't rely on timing ）

用同步系统模型更容易解决问题，它对执行速度、最大消息传输时延以及时钟的准确性等做了假设，这有助于你解决问题，因为你可以基于这些假设进行操作，也可以假设一些失败的场景永远不会发生。

当然，一些同步系统模型的假设并不是特别现实。
现实世界的网络总是有失败的，而且对消息延时并没有严格的界限。
现实世界的系统最好是部分同步：它们可能有时准确工作，并有一些上界，但有时消息会永久延迟，时钟同步也会失效。
我不会讨论同步系统的算法，但你可能会在一些介绍性的书籍遇到它们，因为它们更容易进行分析（但在实际系统中并不现实）。

#### 一致性问题

这篇文档的剩余部分，我们会改变系统模型的不同参数。
下面，我们来看看改变两个系统属性如何会影响系统设计：

* 是否网络分离包含在错误模型（failure model）中

* 同步vs异步的时间假设

我们会讨论两个不可达结果（impossibility result）。

当然，为了更好地展开讨论，我们需要引入要解决的问题。这个问题就是[一致性问题](https://en.wikipedia.org/wiki/Consensus_%28computer_science%29)。

几个计算机（或者节点）如果同意某个值，就说明它们达成一致。更准确地说：

* 同意（agreement）：每一个正确的进程必须同意相同的值；

* 诚实（integrity）：每一个正确的进程的决定做多只有一个值，如果它要决定某个值，则这个值之前必须由其他进程提议

* 结束（termination）：所有进程最终做出一个决定

* 有效性（validity）：如果所有正确的进行提议同样的值V，则所有正确的进行决定V。

一致性问题是很多商业分布式系统的核心。毕竟，我们希望利用分布式系统的可靠性和性能，但又不希望去处理因分布式带来的结果（如：节点不一致等），
因此解决一致性问题有助于解决一些相关的、更加高阶的问题，例如原子广播和原子提交。

#### 两个不可达结果（impossibility result）

第一个不可达结果是FLP，它跟分布式算法设计相关；第二个是CAP理论，它跟分布式的实践者相关--给那些需要选择不同的系统设计，但又不需要关系算法设计的人。

#### FLP不可达结果

我只会简单地总结FLP不可达结果，尽管它在学术圈非常重要。
FLP通过异步系统模型来检查一致性问题。
它假设唯一失败的方式是crash、网络是可靠的、消息的延迟没有界限（bound）——这个也是异步系统模型的典型时间假设。

基于这些假设，FLP认为"在异步系统中，不存在一个决定的（deterministic）算法适用于一致性问题，因为受到失败限制，即便消息不会丢失、最多只有一个进程失败、进程只会因为crash而失败"。

（译者注：由Fischer、Lynch和Patterson三位科学家于1985年发表的论文《Impossibility of Distributed Consensus with One Faulty Process》指出：
在网络可靠，但允许节点失效--即便只有一个的最小化异步模型系统中，不存在一个可以解决一致性问题的确定性共识算法。
原话是，No completely asynchronous consensus protocol can tolerate even a single unannounced process death）

这个结果意味着，即便是在一个最小的、消息不会延迟的系统模型中，也不可能解决一致性问题。
它的论述是这样的，如果这个共识算法存在，那么就可以设计出这样的执行机制：它会延迟消息传递--异步系统模型允许这样，于是会导致在一段时间内状态是不确定的。
因此，这个算法是不存在的。
（原话是，The argument is that if such an algorithm existed, 
then one could devise an execution of that algorithm in which it would remain undecided ("bivalent") for an arbitrary amount of time by delaying message delivery - which is allowed in the asynchronous system model. Thus, such an algorithm cannot exist.）


这个不可达结果是重要的，因为它强调了异步系统模型的折中：当不保证信息传递的上下界时，共识性算法必须放弃安全性或者活跃性（liveness）。
（原话是，algorithms that solve the consensus problem must either give up safety or liveness when the guarantees regarding bounds on message delivery do not hold）

这个洞察对算法设计者尤其重要，因为它给可以用异步系统模型解决的问题加了一个严格的限制。
CAP理论则跟实践者更加相关：它基于的假设稍微不同（网络失败而不是节点失败），而且更方便实践者在系统设计中做出选择。

#### CAP理论

CAP理论最早是计算机科学家Eric Brewer做的一个假设。它是一个常见的、且相对公平的方式来思考系统设计时做的保证。

这个理论表示以下三个属性，只能同时满足两个：

* 一致性：所有节点在相同时间看到的数据是一样的

* 可用性：失败的节点不会影响其他节点的工作

* 分区容错性：系统在网络失败导致消息丢失的情况下，仍然能够继续工作

我们可以画一个图来展示满足任意两个条件的系统：

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/2_CAP_intersection.png)

注意到，中间的部分（也就是满足三个条件的部分）是不可达到的。
这样，我们得到三个不同的系统类型：

* CA（一致性+可用性）：例子是完整的严格仲裁协议（full strict quorum protocols），如：2PC（two-phase commit）

* CP（一致性+分区容错性）： 例子是多数仲裁协议（majority quorum protocol），即少量的分区不可用，如 paxos

* AP（可用性+分区容错性）：例子是使用冲突决议（conflict resolution）的协议，如 Dynamo

CA和CP提供了相同的一致性模型：强一致性。
唯一的区别是，CA系统不能允许任何节点失败；而CP系统可以允许一个有2f+1个节点的非拜占庭失败模型（ non-Byzantine failure model）里，最多f个节点失败。
理由很简单：


 



