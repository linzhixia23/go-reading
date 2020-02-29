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

* CA系统并没有区分节点失败和网络失败，因此必须停止接受写入，以避免引入分歧。
它无法确认是远端节点挂了，还是网络连接失败：因此最稳妥的方式就是不给写入。

* CP系统防止分歧的方式是在分区的两边都强制不对称的行为。
它只维护大多数的分区，而让少数的分区不可用，这样在一定程度上保持可用性，而且保证单点复制（single-copy）的一致性。

我会在讨论paxos的章节阐释更多细节。CP系统的要点是，它们把网络分区融合到失败模型（failure model）中，使用像Paxos，Raft这样的算法来区分大多数分区和少数分区（ a majority partition and a minority partition）。
CA系统对分区并不敏感，而且更加常见：它们经常用2PC算法，而且在传统的分布式关系数据库中很常见。

如果分区是必须存在的，那么CAP理论其实就是在可用性和一致性中做出选择。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/2_CAP_choice.png)

我认为从CAP理论应该能推导出4个结论：

第一，许多早期的分布式关系数据库的系统设计并没有考虑到分区容错性（它们是CA设计）。
分区容错性是现代系统很重要的一个属性，因为在系统是物理隔壁的情况下，网络隔离（network partition）的可能性变高了。

第二，在网络隔离（network partition）的情况下，强一致性和高可用之间存在鸿沟。
CAP理论是对强一致保证（strong guarantees）和分布式计算之间的折中的阐释。

在某种意义上，想保证一个由不可靠的网络连接起来的相互独立的节点像一个非分布式系统一样工作，是一件很疯狂的事。

在分区的情况下，强一致性保证要求我们放弃可用性。
这是因为，在两个副本不能通信的情况而又分别在持续接收写请求的情况下，不能保证不会出现分歧。

那我们应该怎么解决呢？使用更强的假设（假设没有分区），或者减弱一致性的保证。
一致性可以跟可用性做折中。如果一致性的要求比"所有节点在相同时间看到的数据都一样"要弱，那么我们就能保证可用性和弱一致性。

第三，在常规操作中，强一致性和性能存在一定冲突。

强一致性要求节点通信，而且对每一个操作达成一致。这会导致常规有高的时延。

如果你的一致性模型不是传统的模型，而是允许副本延迟或者分歧，那么你就可以在常规操作中减少时延，并在存在分区的情况下满足可用性。

当一个操作中涉及到更少的信息和更少的节点，那么它的速度就会更快。
但是要想做到这点，只能是减弱一致性的约束：让一些节点访问频率降低，这也意味着节点可以有旧数据。

这也可能会导致异常。你不再保证能够得到最新的数据值。根据不同的一致性保证，你读到的数据可能是旧的，甚至可能没有更新。

第四，或许并不直接，如果我们想在网络分区的情况下放弃可用性，那么我们应该探索除了强一致性模型之外能够达到我们目的的一致性模型。

例如，用户数据放在多个数据中心，而数据中心的连接已经失效，大多数情况下，我们仍然希望允许用户使用网站/服务。
这意味着调解两个有差异的数据集既是技术的挑战，也是商业的风险。
通常技术挑战和商业风险是可控的，因此最好还是能保证高可用性。

一致性和可用性并不是一个二选一的抉择，除非你限制自己要保证强一致性。
但是，强一致性只是其中一个一致性模型。正如[Brewer本人所说](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)，"三选二"本身是有误导的。

如果你只想这些讨论中得到一个简单的idea，那就是：一致性并不是一个单一的、没有歧义的属性。记住：

ACID一致性 != CAP一致性 != Oatmeal一致性 

一致性模型是一个保证--任何关于数据存储给到使用它的程序的保证。

在CAP理论里，C指的是强一致性，但是一致性并不是指"强一致性"。

我们来看一些其他的一致性模型。

### 强一致性 vs 其他一致性模型 

一致性模型可以分为两类：强一致性和弱一致性。

* 强一致又可以分为：Linearizable consistency 和 Sequential consistency 

* 弱一致性又可分为： Client-centric consistency、Causal consistency 和 Eventual consistency 

强一致性模型保证出现的顺序和更新的可见性等同于non-replicated系统。
弱一致性则没有这样的保证。

需要说明的是，下面的例子并不是全部的样例。在说一次，一致性模型只是定义程序员和系统之前的任意联系，所以它们可以是任何定义。

#### 强一致模型

强一致模型可以进一步分为两个相似，但又有些细微不同的模型：

* Linearizable consistency: Under linearizable consistency, all operations appear to have executed atomically in an order that is consistent with the global real-time ordering of operations. 

* Sequential consistency: Under sequential consistency, all operations appear to have executed atomically in some order that is consistent with the order seen at individual nodes and that is equal at all nodes. 

最大的不同点在于，linearizable consistency要求操作的顺序等同于实际的实时的顺序。
而sequential consistency允许操作重新调整顺序，只要每个节点上的操作顺序报纸不变就可以。
能区分出这两者的方式是，它们是否能够观察到全局的系统的所有输入和时机；从客户端交互的视角，这两者没有太大区别。

这样的区别看起来并不重要，但需要指出的是，sequential consistency does not compose（不知道什么意思。。）。

强一致性模型允许程序员把一台单机服务器换成分布式集群，而不会引起任何问题。

需要注意的是，对于弱一致性模型并没有普遍适用的定义，因为"非强一致性模型"可以是任何类型。

#### client-centric一致性

Client-centric一致性是客户端或者会话的一致性模型。例如，一个client-centric一致性可能保证客户端版本不会读到数据的旧的版本。
它的实现通常是在客户端的函数库构建额外的缓存，这样如果客户端访问到有旧数据的副本服务器，则会返回缓存的数据，而不是从副本返回数据。

Clients may still see older versions of the data, if the replica node they are on does not contain the latest version, but they will never see anomalies where an older version of a value resurfaces (e.g. because they connected to a different replica)

#### 最终一致性

最终一致性是指，如果你不再修改一个值，那么在一个规定的时间之后，所有的副本最终都会达成相同的值。
也就是说，在那个时间内，副本之间的结果是不定义的。因为它很容易满足，因此如果没有额外的补充信息，这样的定义是没有意义的。

说某样东西最终一致，就好比在说"人总是要死"一样。
这是一个非常弱的限制，因此我们希望至少明确以下两点：

首先，"最终"是多久？如果能有一个严格的下界会比较好，或者至少是系统收敛到某个特定值要多长时间。
其次，副本如何对一个值达成一致？一个系统总是返回"42"是最终一致性：所有的副本都对一个值达成一致。
它没有收敛于一个有用的值，因为它总是返回相同的固定值。所以，我们可以用一个更好的方法。
例如，总是取有最大时间戳的值。

所以，当有人说"最终一致性"时，应该用更加精确的术语来描述，例如"eventually last-writer-wins, and read-the-latest-observed-value in the meantime" consistency。
"how"很重要，因为一个不好的方法，可能导致写丢失，例如一个节点的时钟设置不正确，而时间戳又正好使用它。

我将会在弱一致性的章节介绍两个副本复制问题的更多细节。

#### 延伸阅读

* [CAP Twelve Years Later: How the "Rules" Have Changed - Brewer, 2012](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)

* [Replicated Data Consistency Explained Through Baseball - Terry, 2011](http://pages.cs.wisc.edu/~remzi/Classes/739/Papers/Bart/ConsistencyAndBaseballReport.pdf)

* [Life Beyond Distributed Transactions: an Apostate's Opinion - Helland, 2007](https://cs.brown.edu/courses/cs227/archives/2012/papers/weaker/cidr07p15.pdf)

* [If you have too much data, then 'good enough' is good enough](https://dl.acm.org/citation.cfm?id=1953140)

* [Building on Quicksand - Helland & Campbell, 2009](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.195.6558&rep=rep1&type=pdf)

### 3、时序

什么是顺序（order），为什么它很重要？ 

我意思是，为什么我们最初总是会对order感到迷惑？为什么我们要关心A在B之前发生？
为什么我们不关心其他的属性，例如"颜色"？

让我们回到分布式系统的定义来回答这些问题。

你或许还记得，我把分布式编程描述为使用多台计算机来解决在单机上的相同问题的艺术。

这其实也就是对顺序感到困惑的关键。任何一个时间只能做一件事情的系统，会产生所有操作的顺序。
正如人穿过一扇门，每一个操作都会按顺序来执行。这个也就是我们努力去维持的本质的编程模型。

传统的模型是：单一程序，单一处理，单一内存空间运行在一个CPU上。
操作系统的抽象没有考虑一个事实，即存在多个CPU和多个程序，因而计算机上的内存实际上是多个程序共享的。
我没有说线程编程和面向事件编程不存在；只是它们是在"1/1/1"模型上的特别抽象。
程序执行是按照一定顺序的：从顶层开始，走到底层结束。

顺序属性获得很大关注，因为最容易定义正确的方式就是，"系统像单机一样运行"。
这通常意味着：a、我们运行相同的操作；b、我们用相同的顺序运行-即便它们可能在多机上。

在实际中，分布式程序运行在多个节点上，有着多个CPU，以及多个操作流输入。
你可以分配一个总的顺序，但它需要准确的时钟，或者是某个通信方式。
你可以用完全准确的时钟来给每一个操作打上时间戳，用它来指出一个完整的顺序。
或者也可以用某种方式，来给总的顺序来标记序列。

#### 整体序列和偏序（total and partial order）

对分布式系统来说，偏序（partial order）是最自然的。不管是网络还是各相互独立的节点，都不会保证任何的相对顺序。
但是对每一个节点来说，你能够观察到一个本地的序列。

总序（total order）定义了集合里的每一个元素的二元关系，并成为序列。

两个不同的元素如果一个大于另一个，那么它们就是可以比较的。
以下的声明在总序和偏序中，对于任意a、b和c都是成立的：

If a ≤ b and b ≤ a then a = b (antisymmetry);
If a ≤ b and b ≤ c then a ≤ c (transitivity);

但是，总序表示所有：

a ≤ b or b ≤ a (totality) for all a, b in X

而偏序则只是reflexive（反射？）：

a ≤ a (reflexivity) for all a in X

注意到，totality包含了reflexivity；所以偏序是总序的一个弱的变体。
换句话说，偏序中一些元素不可比。

Git的分支则是一个很好的例子。Git的版本控制允许你从一个base分支创建多个分支--例如从master分支。
每一个分支代表它们从共同祖先拉出来的代码演化记录：

[ branch A (1,2,0)]  [ master (3,0,0) ]  [ branch B (1,0,2) ]

[ branch A (1,1,0)]  [ master (2,0,0) ]  [ branch B (1,0,1) ]
          
                  \  [ master (1,0,0) ]  /

分支A和B都是从共同的祖先衍化出来的，但是他俩之间并没有一个严格的序列：
它们代表着不同的历史记录，而不能reduce成为一个单一的历史线（single lenear history），除非是通过额外的操作（如merge）。
当然，你可以把所有的提交按任意顺序串起来，但是这样强制地添加原本不存在的序列，可能会导致信息丢失。

在一个单一节点的系统，总序是实际需要的：指令的执行和消息的传递，都以特定的、可观察的顺序进行。
我们要依赖总序--这样使得程序的执行可预见。
这样的总序可以保持在分布式系统，但是需要一定的代价：通信代价很昂贵，而且时间的同步很困难且脆弱。

#### 什么是时间？（What is time？）
                  
时间是顺序的来源-它允许定义操作的顺序-同时，它也能提供一个人能够理解的解释。

在某种意义上，时间就是一种整型计数器。而刚好大多数计算机都有一个专门的time sensor，也就是时钟。                  

时间戳是一个简便的值，用于记录宇宙从开始到现在的状态--如果某件事情在某个时间戳开始，则它可能收到这个时间之前的任何事件的影响。
这个idea可以泛化为一个casual clock显示地记录cause，而不是简单地假设在某个时间戳之前的事件都是相关的。
当然，常用的假设是我们应该只关心特定系统的状态，而不是全部。

假设时间在任何地方都以相同的速度进行--这是一个很大的假设，这样时间和时间戳在程序就有了解释性，即：

* 顺序（Order）

* 持续时间（Duration）

* 解释性（Interpretation）

顺序。 当我说到时间是顺序的来源，我真正的意思是：

* 可以给无序的时间贴上标签，以便能够有序化；

* 可以用时间戳来强制特定的操作顺序，或者传递消息；

* 可以用时间戳来决定某个时间是否在时间顺序上，要先于其他时间

解释性。时间可以做为普适性的可以比较的值。时间戳的值可以解释为日期，这方便人阅读。
给定宕机发生的时间戳，你就可以说出它是上周六发生的。

持续时间。它是用时间来衡量的，跟现实世界有一些联系。
算法通常不会关系时钟的绝对时间，但它们用持续时间（duration）来衡量调用。
特别的是，用于等待的时间可以提供一些关于系统是否隔离或者是否高延迟的线索。

本质上，分布式系统的组件并没有一个可预测的行为。
它们并不会保证任何特定的顺序、处理的速度、或者不会延迟。
每个节点可能有本地的执行顺序，但是它们各自的本地顺序（local order）是相互独立的。

强制顺序（imposing order）可能是一个减少可能执行空间和并发的方式。
对人类来说，如果时间能够以任何顺序发生，是很难去理性思考它们的--因为这中间有太多需要考虑的组合方式。

#### 时间是否在任何地方都以相同的速率流逝？ 

我们每个人都基于自己的经验，对时间有自己直觉的概念。
不幸的是，这种直觉的时间的概念更容易描述全序（total order），而不是偏序（partial order）。
描述一个事件在另一个事件之后发生很容易，但描述并行很困难。
理解消息的单个顺序比描述它们以不同的顺序和不同的延迟时间到达容易。

尽管如此，我们在实现分布式系统的时候，尽量避免对时间和顺序做出强假设，因为假设越强，则系统对"time sensor"就越脆弱。
此外，强制顺序会提高代价。我们能忍受更多短暂的不确定性，我们就能够更好的利用分布式计算。

对于，"时间是否在任何地方都以相同的速率流逝"这个问题，三个常用的回答是：

* "全局时钟（global clock）" ：yes
* "本地时钟（local clock）" ：no，but
* "无时钟（No clock）" ：no！

这对应我在第二节提到的三个关于时间的假设：同步系统模型有全局时钟，部分同步模型有本地时钟，异步系统模型不使用时钟。
我们接下来讨论更多细节。

#### "全局时钟"的假设

全局时钟假设是指，有一个绝对精准的时钟，而且每个人都能够访问到它。这也是我们习惯的思考时间的方式，因为在人类的交互中，对时间的细小不同可以忽略。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/3_clock_assumption_global.png)

全局时钟是全序（total order）的一个来源（它指所有节点的每一个操作，即便这些节点之间没有通信）。

然而，这只是一个理想的情况：在现实中，时钟的同步只可能会限制准确精度。

假设时钟在分布式的节点中绝对地同步，这也意味着时钟以相同的值开始，而且从不出现偏差。
这是一个很好的假设，因为这样你可以随意使用时间戳来决定一个全局的顺序，不过这是一个不小的挑战，而且可能导致异常。
这样的失败，在很多不同情景都会出现-例如用户不小心改变了本地机器的时间、过期的机器加入了集群、同步的时钟发生了一点偏移，以及其他很难去跟踪的异常。

不过，仍然有一些实时的系统采用这样的假设。
Facebook的[Cassandra系统](https://en.wikipedia.org/wiki/Apache_Cassandra)就是一个假设时钟同步的例子。
它们用时间戳来缓解写的冲突-有最新的时间戳的写请求赢得写的资格。
这就意味着，如果时钟出现了偏移，新的数据可能会被忽略，或者被老的数据覆盖写。
另一个有意思的例子是Google的[Spanner系统](https://research.google/pubs/pub39966/)：文章描述了它们的TrueTime API，它用同步时间，但同时也会预估时钟偏移的最坏的情况。

#### "本地时钟"的假设

第二个，也可能是更受欢迎的假设，则是每一个机器有自己的时钟，但没有全局时钟。
这也就意味着，你不能用本地时钟的顺序来决定一个远端的时间戳出现在本地时间戳的前面还是后面；换句话说，你不能比较两台不同机器的时间戳。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/3_clock_assumption_local.png)

本地时间假设相对更接近于现实世界。
它用到了偏序：每个系统的事件是有序的，但不能只用一个时钟来对不同平台的事件做排序。

然而，你可以用时间戳来对单机上的事件进行排序；
你可以在单机上使用超时，只要你保证不会导致时钟jump around。
当然，在终端用户的机器上，可能会有更多的假设：例如，用户在查询操作系统的时间控制器时，可能会不小心把时间更改到了另外的值。

#### "无时钟"的假设

最后，介绍逻辑时间（logical time）的概念。
我们不再使用时钟，而且用另一种方式来追踪因果关系。
记住，时间戳只是一种最简便的用于记录世界某个时刻状态的方式--我们同样可以用计数器和通信（counter and communication）来决定一个事件是否发生在另一个事件之前、之后或者同时发生。

用这种方式，我们就可以决定不同机器的事件的发生顺序；当然，这种方式并不能记录事件间隔和使用超时（因为它们并没有"time sensor"）。
这是偏序：事件在单机上可以用计数器来排序，而且不需要通信，但是对不同系统的事件排序，则需要信息交换。

关于分布式系统引用最高的一篇文章是Lamport的[time，clocks and the ordering of events](http://lamport.azurewebsites.net/pubs/time-clocks.pdf?ranmid=24542&raneaid=je6nubpobpq&ransiteid=je6nubpobpq-dwftub0_oz6mfeweghll0w&epi=je6nubpobpq-dwftub0_oz6mfeweghll0w&irgwc=1&ocid=aid2000142_aff_7593_1243925&tduid=(ir__ve1i3hs2q9kftw2mkk0sohzz0u2xlkfh1y19bzhl00)(7593)(1243925)(je6nubpobpq-dwftub0_oz6mfeweghll0w)()&irclickid=_ve1i3hs2q9kftw2mkk0sohzz0u2xlkfh1y19bzhl00)。
向量时钟（vector clock）是这个概念的泛化，它可以用于追溯事件的因果，而不需要使用时钟。
Cassandra的Riak和Linkedin的Voldemort使用向量时钟（vector clock），而不是假设每一个节点都连接到一个绝对准确的全局时钟。
这可以使系统避免前面提到的因时钟不准确所导致的问题。

当不使用时钟时，事件在不同机器上的顺序的最大精度（maximum precision）的上界就由通信延时（communication latency）决定了。

#### 时间如何在分布式系统中使用？ 

时间的定义是什么？ 

1、时间可以定义跨系统的顺序

2、时间可以定义算法的边界条件（boundary condition）

事件的顺序在分布式系统中很重要，因为很多分布式系统的属性是根据操作/事件的顺序来定义的：

* 系统的正确性（correctness）依赖于正确的事件顺序，例如分布式数据库的序列化。

* 顺序可以在资源争夺的时候做为tie breaker，例如一个widget有两个顺序时，可以实现第一个，并取消第二个。

一个全局的时钟可以允许两台不同机器的操作按顺序进行，而不需要机器之间直接通信。
没有全局时钟的话，我们需要通过通信来决定顺序。

时间还可以用于定义算法的边界条件-具体说，区分"高延时"和"节点或网络连接宕机"。
这是一个非常重要的使用样例；在大多数实际系统中，会使用超时来确定一个远端机器是否失败，或者它的延时超过了一般的经验值。
在算法里，这个叫错误检测（failure detector）；我会在后面讨论。

#### 向量时钟

前面的时候，我们讨论了分布式系统上不同节点的处理速率的各种假设。
假如我们没有一个准确的同步时钟，或者说，我们要求我们的系统不对时间的同步敏感，那么，我们如何定义事情的顺序？

Lamport时钟和向量时钟可以替代物理时钟，它们使用计数器和通信来决定分布式系统的事件的顺序。
这些时钟提供了一个不同节点之间可以相互比较的计数器。

Lamport时钟很简单：每个处理进程根据下面规则来维护一个计数器： 

* 任何时候，一个进程进行工作，就增加计数器；

* 任何时钟，一个进程发送消息，就带上计数器；

* 当收到一个消息，则把计数器设置为 max（本地计数器，收到的计数器+1） 

用代码表示如下：

```
function LamportClock() {
  this.value = 1;
}

LamportClock.prototype.get = function() {
  return this.value;
}

LamportClock.prototype.increment = function() {
  this.value++;
}

LamportClock.prototype.merge = function(other) {
  this.value = Math.max(this.value, other.value) + 1;
}
```

Lamport时钟允许计数器在不同系统之间比较，但需要注意：Lamport时钟定义的是偏序。
如果timestamp(a) < timestamp(b): 

* a可能在b之前发生

* a可能无法跟b进行比较

这就是时钟一致性条件（clock consistency condition）：如果一个事件在另一个事件之前到来，则该事件的逻辑时钟在其他事件之前。
如果a和b来自于相同的随机历史事件（causal history），则要么两个时间戳的值由同一个进程产生；要么b是回应a带过来的消息，而a是发生在b之前的。

直觉上，这是因为Lamport时钟只能携带一个时间线的历史记录；因此，比较两个从来没有进行过通信的系统的Lamport时间戳，可能会导致两个并行的事件看起来是有序的--而实际上它们没有。

设想一个系统在初始化之后，便分成了两个相互独立的子系统，而且相互之间不再通信。

每个相互独立的系统的所有事件，如果a在b之前发生，则ts(a) < ts(b)；但如果你比较两个相互独立的系统，你很难说出它们的相对顺序。
当系统的每个部分都给事件分配了时间戳，这些时间戳之间并没有任何关系。
因而，两个事件可能会看起来是有序的，而实际上它们并不相关。

不过，这仍然是一个很有用的属性：对单机来说，任何消息在ts(a)时间发出，而在ts(b)时间收到，则ts(b)> ts(a)。

向量时钟是Lamport时钟的扩展，它包含了N个逻辑时钟的数组 [ t1, t2, ... ] - 每个节点都有一个。
此时不再是更新共同的计数器，而是每个节点只更新它自己的逻辑时钟。以下是更新规则：

* 任何时候，一个进程进行工作，都会增加向量中它的逻辑时钟的值

* 任何时候，一个进程发送消息，都要携带整个逻辑时钟的向量

* 当收到消息的时候：a、更新向量中的每个节点 max(local,received); b、增加向量中代表该节点的逻辑时钟的值。

同样的，附上代码：

```

function VectorClock(value) {
  // expressed as a hash keyed by node id: e.g. { node1: 1, node2: 3 }
  this.value = value || {};
}

VectorClock.prototype.get = function() {
  return this.value;
};

VectorClock.prototype.increment = function(nodeId) {
  if(typeof this.value[nodeId] == 'undefined') {
    this.value[nodeId] = 1;
  } else {
    this.value[nodeId]++;
  }
};

VectorClock.prototype.merge = function(other) {
  var result = {}, last,
      a = this.value,
      b = other.value;
  // This filters out duplicate keys in the hash
  (Object.keys(a)
    .concat(b))
    .sort()
    .filter(function(key) {
      var isDuplicate = (key == last);
      last = key;
      return !isDuplicate;
    }).forEach(function(key) {
      result[key] = Math.max(a[key] || 0, b[key] || 0);
    });
  this.value = result;
};

```

下图展示了时钟向量： 

 ![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/3_vector_clock.png)

上面的三个节点(A,B,C)都保存向量时钟的状态。当事件发生时，它们用向量时钟的当前值来标记时间戳。
自己验证一下向量时钟，如{A:2,B:4,C:1}可以让我们更清楚事件如何影响消息。

向量时钟的一个问题是，它们每个节点都需要一个数据项，这样对于大的系统，这可能会变得很大。
很多技术已经用于减少向量时钟的大小：使用周期性的垃圾回收，或者限制大小来减少精度。

我们已经看到了不使用物理时钟来追溯顺序和因果关系。
现在，再来看持续时间(time duration)如何用于截断。

#### 失败探测（截止时间）

正如我前面提到的，等待的总时长可以用于判断系统是否网络分离（partition），或者是处于高时延。
在这个情况下，我们不需要假设一个精确的全局时钟--只要有一个足够稳定的本地时钟就可以。

对于一个在节点上运行的程序来说，它如何知道远端的节点失败了呢？
在缺乏足够精确的信息的情况下，我们可以推断，在超过一段合理的时间之后，没有回应的远端节点就是失败了。

那么，什么是"合理的时间"呢？这取决于本地和远端节点的延时。
与其显式地用特定的算法计算特定的值，不如用一种合适的方式来抽象它。

失败探测是一种抽象确切时间的方式。失败探测是通过心跳信息和记时器来实现的。
进程交换心跳信息。如果没有在超时之前收到应答消息，那么该进程就要怀疑其他进程出现了问题。

[Chandra等人](https://www.cs.utexas.edu/~lorenzo/corsi/cs380d/papers/p225-chandra.pdf)在解决一致性问题的情景下，讨论过错误探测的问题
--这个是大多数副本问题（replication problem）的基础，而副本问题则是解决在在一个延时，而且有网络分离的环境下，如何达成一致。

它们用两个属性来描述失败探测，完整性和准确性。

* 强完整性。每一个崩溃的进程最终都会被每一个准确的进程监测到。

* 弱完整性。每一个崩溃的进程最终会被一些准确的进程监测到。

* 强准确性。没有任何疑似的进程。

* 弱准确性。一些进程永远不会成为疑似进程。

完整性比准确性更容易达成；所有主要的失败探测都实现了它--你所需要做的就是不会永远地等待某个疑似进程。
Chandra指出，一个弱完整性的失败探测器可以转化成强完整性（只要把疑似进程广播出去），这就使得我们只要关注于准确性属性。

避免误判疑似进程是一个很难的问题，除非你能假设信息的延时有一个hard maximum（注：理解应该是强制的最大的超时时间）。
这个假设可以在一个同步模型中做出，于是在这个系统中，失败探测就可以做到强准确性。
在没有对消息延时做强制界限的系统模型中，失败探测最好情况是达到最终准确。
（Under system models that do not impose hard bounds on message delay, failure detection can at best be eventually accurate）。

Chandra等人指出，即便是一个非常弱的失败探测器（最终弱准确性和弱完整性）都能用于解决一致性问题。
下图展示了系统模型和问题可解性的关系：

 ![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/3_chandra_failure_detectors.png)

正如上面所看到的，在异步系统只能够，如果没有失败探测器（failure detector），则特定的问题无法解决。
这是因为，如果没有失败探测，那就没办法知道一个远端节点是否已经崩溃，还是只是处于比较高延迟。
这个区别对于任何single-copy一致性的系统来说很重要：失败的节点可以忽略，因为它们不会导致歧义，但是网络分离的节点（partitioned node）如果忽略了则会导致不安全。

那么如何实现一个失败探测器呢？概念上来讲，实现一个简单的失败探测器并不难，只要超过超时时间，就可以认为是失败。
但是这里最有意思的点在于，如果判断一个远端节点是失败（而不是延迟）。

理想情况下，我们希望失败探测能够适应于网络的情况，避免以hardcode的方式设置超时时间。
例如，Cassandra使用Accrual Failure Detector的方式，来给定一个0到1直接的可疑值，而不是单纯的二元判断的结果。
这允许应用能够在精确判断和尽早发现错误之间做出抉择。

#### 时间，顺序和性能

在前面，我略微提过，顺序是有代价的。

如果你写一个分布式系统，你大概会有一个以上的计算机。在现实世界中，大多数情况是偏序而不是总序。
你可以把偏序转化成总序，但是这个需要通信、等待，以及增加一些限制，要求大多数计算机在特定时间点如何工作。

时间和顺序通常会在一起讨论，单独时间这个属性本身并没有太大用处。
算法并不太在意时间，它们更关注更多抽象的属性：

* 事件的随机顺序（casual ordering）

* 失败探测（例如，消息传输的近似上界）

* 一致性快照（consistent snapshot）（例如，在某个节点检验系统状态的能力；在这里不进行讨论）

强制一个全序是可能的，但是代价太大。它需要你以相同的速度处理（也就是最低的速度）。
通常，最简单的保证所有事件都以特定的顺序传递的方式是，提名一个单点节点，让所有的操作都要经过它。

是否时间/顺序/同步是真的需要的呢？这要看情况。在一些情况下，我们希望系统的每一个中间操作是一致的。
例如，在大多数情况下，我们希望数据库回复的信息包含所有信息，而尽量避免系统返回不一致的结果。

但是，在另一些case，我们可能不需要太多时间/顺序/同步。
例如，你跑一个运行很长时间的计算，而且结果出来之前，并不在乎中间结果--此时，你并不需要太多的同步，因为你能保证答案是正确的。

当只有一小部分case真正在乎最终结果的情况下，同步通常是所有操作之间一个比较生硬的工具。
那么，什么时候顺序需要去保证准确性呢？CALM理论提供了一个回到--我将会在最后一章讨论这个问题。

在其他情况，能接受的其他解法就是使用估计--也就是基于系统的节点的信息。
特别的，在网络分离的情况下，一个节点需要在只能访问一部分机器的情况下，响应请求。
在这样的情况下，终端用户其实不能真正区分出，相对容易获得的较新结果和保证准确但计算代价昂贵的结果之间的区别。
以较小的代价，得到一个接近准确的结果，是可以接受的。


#### 延伸阅读

* [Time, Clocks and Ordering of Events in a Distributed System - Leslie Lamport, 1978](http://lamport.azurewebsites.net/pubs/time-clocks.pdf?ranmid=24542&raneaid=je6nubpobpq&ransiteid=je6nubpobpq-omavjvcatwjgr8iucc7i2a&epi=je6nubpobpq-omavjvcatwjgr8iucc7i2a&irgwc=1&ocid=aid2000142_aff_7593_1243925&tduid=(ir__ve1i3hs2q9kftw2mkk0sohzz0u2xlheyfa19bzhp00)(7593)(1243925)(je6nubpobpq-omavjvcatwjgr8iucc7i2a)()&irclickid=_ve1i3hs2q9kftw2mkk0sohzz0u2xlheyfa19bzhp00)
 
* [Unreliable failure detectors and reliable distributed systems - Chandra and Toueg](https://dl.acm.org/doi/pdf/10.1145/226643.226647?download=true)
 
* [Latency- and Bandwidth-Minimizing Optimal Failure Detectors - So & Sirer, 2007](http://www.cs.cornell.edu/people/egs/sqrt-s/doc/TR2006-2025.pdf)
 
* [The failure detector abstraction, Freiling, Guerraoui & Kuznetsov, 2011](https://dl.acm.org/doi/pdf/10.1145/1883612.1883616?download=true)

* [Consistent global states of distributed systems: Fundamental concepts and mechanisms, Ozalp Babaogly and Keith Marzullo, 1993](http://www.cs.utexas.edu/~lorenzo/corsi/cs380d/papers/chapt4.pdf)

* [Distributed snapshots: Determining global states of distributed systems, K. Mani Chandy and Leslie Lamport, 1985](https://dl.acm.org/doi/pdf/10.1145/214451.214456?download=true)

* [Detecting Causal Relationships in Distributed Computations: In Search of the Holy Grail - Schwarz & Mattern, 1994](http://www.vs.inf.ethz.ch/publ/papers/holygrail.pdf)

* [Understanding the Limitations of Causally and Totally Ordered Communication - Cheriton & Skeen, 1993](https://dl.acm.org/doi/pdf/10.1145/168619.168623?download=true)


### 4. 复制集（Replication)

Replication是众多分布式系统问题中的一个。
我关注它胜于其他问题，如：leader选举、失败探测、共享资源互斥（mutual exclusion）、一致性和全局快照等，是因为它通常是人们最感兴趣的话题。
例如，两个并行的数据库的不同就在于它们的复制集特征。
此外，replication跟很多子问题相关，例如leader选举、错误探测、一致性等。

Replication是一个群组的通信问题。如何协议和通信（arrangement and communication）能够提供我们期望的性能和可用性？
我们如何在网络分离和节点同步失败的情况下，保证容错性、持久性和非歧义性？

同时，有很多方式实现replication。我这里介绍的只是high-level的模式。
我的目标是探索设计的空间，而不是解释特定的算法。

我们先来定义，replication是什么样的。我们假定有一些初始的数据库，客户端发送一些请求来改变数据库的状态。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/4_replication_both.png)
 
 协议和通信的模式可以分为下面几个阶段：
 
 * 1、请求-客户端发送请求到服务端
 
 * 2、同步-replication的同步部分开始
 
 * 3、回应-返回应答给客户端
 
 * 4、异步-replication的异步部分开始

这个模型大致基于[这篇文章](http://www-users.cselabs.umn.edu/classes/Spring-2018/csci8980/Papers/ProcessReplication/Understanding-Replication-icdcs2000.pdf)。
注意，任务的每一个部分的信息交换取决于特定的算法：我有意尝试跳过它，而不必讨论特定的算法。

给定了这些状态，我们能用什么样的通信模式呢？我们选用的模式的性能和可用性是什么样的呢？

#### 同步复制集

第一个模式是同步复制集(synchronous replication)。我们来看下它是什么样的：

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/4_replication_sync.png)

这里，我们可以看到三个不同的阶段：首先，客户端发送请求。
接着，复制集同步(synchronous portion of replication)开始，也就是，客户端阻塞，等待系统响应。

在同步阶段，第一台服务器联系其他两台服务器，一直等待收到其他服务器的回应。
最后，它发送应答给客户端通知结果（例如，成功或失败）。

整个过程看起来很直接。我们如何在不涉及同步阶段的算法细节情况下，讨论这个特定的协议和通信模式呢？
首先，这是一个write N-of-N方法：在一个响应返回之前，它会被系统的每一个服务器看见和确认。

对于N-of-N方法，系统不能容忍任何服务器的缺失。当一个服务器缺失，系统则不能写给所有节点，则它不能再进行处理。
它或许能提供数据的read-only访问，但修改则是不允许的。

这种协议方式能提供非常强健的持久性（durability)：当客户端收到请求返回后，它能确定所有的服务器都收到、存储和确认了它的请求。
如果一个确认的更新要丢失的话，除非所有N个副本都丢失，这是你能做出的最好的保证。

#### 异步复制集

我们来把它跟异步复制集进行对比。正如你猜想的那样，它跟同步复制集是相反的：

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/4_replication_async.png)

这里，master(/leader/coordinator)节点会发送响应节点给客户端。
它或许最好会把更新存储在本地，但它不会做太多的工作来进行同步，而客户端也不需要强制等待服务器之间多轮的通信。

接着，复制集异步(asynchronous portion of the replication)开始。
这时，master用某些通信模式联系其他服务器，其他服务器更新它们的数据副本。通信方式取决于使用的算法。

我们如何在不涉及算法细节的情况下来讨论这个特定的协议方式呢？
这是一个write 1-of-N方法：响应立刻返回，而更新会在稍后的时间传播。

在性能的角度，这意味这系统可以更快：客户端不需要花更多时间等待系统内部来完成它们的工作。
系统也更加能够忍受网络延时：因为内部延时的抖动不会在客户端造成更多的等待时间。

这种协议方式只能提供弱的耐久性的保证。如果没有任何的错误，数据最终会复制到所有N台机器。
当然，如果唯一有这个数据的服务器在这一切发生之后丢失了数据，那么这个数据就会永远丢失。

对于1-of-N的方法，只要至少还有一个节点是正常的，系统就还能使用。（这个只是理论上的，实际上它的负载可能会很高）。
一个像这样的纯lazy的方法并不提供持久性和一致性的保证；你允许往系统写数据，但是在错误发生时，它不保证你读回来的就是你写入的数据。

最后，需要注意的是，passive replication并不保证系统里的所有节点都保持相同的状态。
如果你接受数据在多个地方，而且不要求这些节点异步一致，你可能会面临数据不一致的风险：从不同地方地点读到的数据不一样（尤其是在节点失败并重启之后），以及不能强制全局限制（因为需要跟每一个节点通信）。

我没有提到过读操作的通信模式（相对于写），因为读其实是用了写的模式：对于读来说，它实际上联系的节点更少。
我们在后面讨论quorum的时候会再讲到。

我们已经讨论了两种基本的协议方法，而没有讨论特定的算法。
我们已经能够讨论通信模式（communication pattern），以及它们的性能、持久性保证和可用性特征。

#### 主要复制集方法概览

看了两个基本的replication方法：同步和异步，我们再来看主要的replication算法。

有很多对replication技术进行分类的方式。我想介绍的是这两种：

* 防止歧义的replication方法（单copy系统）

* 有歧义风险的replication方法（多master系统）

第一组的方法使得系统具有"像单一系统一样工作"的属性。特别是，当部分错误出现时，系统保证其single copy特征激活。
此外，系统保证副本总是达成一致的。这也就是一致性问题（consensus problme）。

如果几个都对一些值达成一致，那么就认为它们是一致的。更正式地，

* Agreement:每一个正确的进程必须认同相同的值；（Agreement: Every correct process must agree on the same value）

* Integrity：每一个正确的进程决定至多一个值，而且如果它决定某些值，它必须由其他进程提议；（Integrity: Every correct process decides at most one value, and if it decides some value, then it must have been proposed by some process）

* Termination: 所有进程最终达成决定；（Termination: All processes eventually reach a decision）

* Validity: 如果所有正确的进程提议相同的值V，则所有正确的进程决定V；（Validity: If all correct processes propose the same value V, then all correct processes decide V）

资源争夺、leader选举、消息多播和原子广播是一致性问题的更多例子。
副本系统要达到单点复制的一致性，需要以某种方式解决一致性的问题。

保持single-copy一致性的复制集算法包括：

* 1n messages (asynchronous primary/backup)

* 2n messages (synchronous primary/backup)

* 4n messages (2-phase commit, Multi-Paxos)

* 6n messages (3-phase commit, Paxos with repeated leader election)

这些算法的错误容忍性（fault tolerance）是不一样的。
我根据它们在执行算法时的消息交换数量把它们简单地做了分类，因为我认为能回到这个问题会很有意思："如果我们增加交换的数据量，要额外付出什么代价？"

下图描述了不同选择的不同考量：

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/4_aspect_of_different_algorithm.png)

上图的一致性、延迟性、吞吐量、数据损失和故障转移又可以追溯到两种不同的复制集方法：同步复制集和异步复制集。
当你等待，你的性能会比较差，但是会有较强的保证。
在我们讨论分区容忍性时，2PC和quorum系统的吞吐量区别会变得明显。

在上图中，强制弱一致性的算法都被归为一类（"gossip"）。然而，我会再后面讨论更多细节。
"transaction"行指的不仅仅是全global predicate evaluation，实际上，它在弱一致性系统中并不支持。

需要指出的是，强调弱一致性的系统，对算法的泛化能力要求不高，这样在技术上就会有更多选择。
因为系统不要求single-copy，则它可以表现得像是有多个节点的分布式系统，它可以更关注于让提供一种方式让使用者更能理解系统的特性。

例如：

* client-centric一致性模型在允许分歧的情况下，使用更加容易的一致性保证

* CRDT探索特定状态的semilattice属性以及基于操作的数据类型

* 合流分析根据计算的单调性信息来最大化地探索无序（disorder）

* PSB（probabilistically bounded staleness）使用模拟和从现实收集数据的方式来刻画partial quorum system 的预期行为。

我会较详细地讨论这些；首先，先来看下复制集算法如何保证single-copy一致性。

### 主备复制集

主备复制集大概是最常用的复制集方法，也是最基础的算法。所有的更新在主机器，其操作日志通过网络传输到备用机的副本。
它有两个变体：

* 异步主备复制集

* 同步主备复制集

同步版本需要两个消息（"更新"+"确认收到"），而异步的版本则只需要一个（"更新"）。

P/B非常常见。例如，在MySQL中，复制集用的就是其异步的版本。MongoDB同样也用P/B（加上一些额外的流程进行故障转移。
所有的操作都在一个master服务器上，它会序列化成本地日志，这些日志会被异步复制给备份的服务器。

正如我们前面在异步复制集时讨论的，任何异步复制集算法只能提供弱耐久性保证（weak durability gurantee）。
在MySQL复制集本身就意味着滞后复制：异步的备份（backup）总是落后主机器（primary）至少一个操作。
如果主机器挂了，没有同步到备份机器的更新就会丢失。

P/B复制集的同步变体保证写操作已经保存在节点，然后才返回给客户端-代价是，需要等待其他副本写完之后的回应。
然而，需要注意的是，即便是这样的方式，也是提供了弱保证。
考虑以下简单的失败场景：

* 主机器接收写请求，然后发送到备份机器

* 备份机器持久化，然后确认写操作

* 主机器在发送确认操作给客户端之前就挂了

客户端会认为确认失败了，而备份机器实际已经提交了；如果备份机器替代了主机器，那么就会不准确了。
这时，可能需要手动清理来协调失败的主机器或者出现歧义的备份机器。

当然，我在这里是做了简化。当所有的主备复制集算法遵循同样的消息模式，它们对失败的处理、副本离线化等都是不一样的。
当时，在这种模式下，它们是没法避免这种主机器的碰巧失败导致的问题。

这种基于log-shipping/主/备方式的关键就是，它们只能提供"最大努力"（best-effort）的保证（例如，它们会很容易丢失更新或者进行不正确的更新，如果节点在不凑巧的时间失败的话）。
此外，P/B模式很容易导致split-brain，即：由于网络临时的问题导致备份机器加入时，会导致主备机器在相同的时间都是激活的。

为了避免因为不凑巧的失败导致一致性保证被破坏；我们需要加入额外一轮的消息传递，这也就是两阶段提交协议（2PC）。

#### 两阶段提交（2PC）

两阶段提交（2PC）是用于学多经典关系数据库的协议。例如，MySQL集群用2PC提供了同步复制集（synchronous replication）。
下图展示了消息的流动：

[ Coordinator ] -> OK to commit?     [ Peers ]

                <- Yes / No

[ Coordinator ] -> Commit / Rollback [ Peers ]

                <- ACK

在第一阶段（投票），仲裁者（coordinator）发送更新到所有的参与者（participant）。
每一个参与者处理更新，然后投票提交或者是终止（abort）。
当投票为提交时，参与者把更新存储在一个临时区（也就是write-ahead log）。
在第二个阶段完整之前，这个更新都认为是临时的。

在第二阶段（决定），仲裁者决定结果并通知每一个参与者。如果所有的参与者都投票提交，则改更新会从临时区取出，并永久生效。

有这样第二个阶段是很有用的，因为它允许系统在一个节点失败时回滚一个更新。
相反地，在第一阶段，在一些节点失败而一些节点成功时，并没有步骤回滚操作，这回导致副本出现歧义。

2PC很容易阻塞，因为单个节点失败会阻塞处理过程，直到节点恢复。
由于有了第二个阶段通知系统状态，使得恢复成为可能。注意，2PC假设数据在每一个节点的稳定存储时不再丢失，以及节点不会再crash。
其实，即便数据在稳定存储里，它也仍然会在crash时被破坏。

在节点失败之后的恢复处理过程细节非常复杂，因为我不会讲这些细节。其主要任务是确保：写到磁盘的数据是持久的，以及做出正确的恢复决定。

正如我们在CAP那章学到的，2PC是CA-它并不是分区容错的。
2PC提供的失败模型并没有包括网络分区；它提供的恢复方式是失败节点一直等到网络分离恢复。
如果一个仲裁者失败了，它并没有一个安全的方式来提拔另一个仲裁者；除非是人工干预。
2PC同样对延迟敏感，因为它的写方式是N-of-N，也就是它要等到最慢的节点确认，写才算完成。

2PC在性能和容错方面有较好的平衡，这也就是它为何在关系数据库中这么流行。
然而，较新的系统通常使用分区容错的一致性算法，因为这些算法能够从临时的网络分离中自动恢复，同时也会较好地处理增加的节点之间的网络延时。

下面来看分区容错的一致性算法。

#### 分区容错一致性算法（Partition tolerant consensus algorithms）

分区容错一致性算法是保持single-copy一直的算法。（原文是：Partition tolerant consensus algorithms are as far as we're going to go in terms of fault-tolerant algorithms that maintain single-copy consistency）

当谈到分区容错一致性算法，最知名的就是Paxos算法。当然，它以难以实现和解释著称，于是，我主要关注Raft，它更容易教学和实现。
我们先来看下网络分区（network partition）以及分区容错算法一般有哪些特点。

##### 什么是网络分区？

网络分区（network partition）是指网络连接其他一个或多个节点失败。而节点本身还是处于active状态，这样它们还能从客户端接收请求。
正如我前面在讨论CAP时提过的，网络分区是一定会发生的，并不是所有系统都能优雅地处理它。

网络分区这个问题很tricky，因为在一个网络分区中，不可能区分出远端节点失败了还是不可达了。
如果网络分区出现，但没有节点失败，则此时系统就被分成了两个部分，而这两个部分都在同时运行。
下图展示了为什么网络分区看起来像是节点失败。

2个节点的系统，失败vs网络分区。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/4_system_of_2_node.png)

3个节点的系统，失败vs网络分区。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/4_system_of_3_node.png)

一个强制single-copy一致性的系统必须要有方法打破对称性：否则，它会分割成两个系统，彼此之前不同，这样就不再是single copy了。

single-copy一致性系统，其网络分区容忍性要求，在一个网络分区中，只能有一个分区是active的，因为在网络分区中，歧义是没有办法避免的。

##### 少数服从多数决定（majority decision）

这是为什么分区容错一致性算法依赖于大多数投票。
要求大多数投票-而不是全部投票（如2PC）-来对更新达成一致，会允许少数节点宕机、处理慢、或者由于网络问题导致不可达。
只要 （N/2+1）-of-N 个节点是正常和可达的，这个系统就会持续工作。

分区容错一致性算法使用奇数个节点（如3、5、7个等）。如果只有2个节点，在失败之后，它不能区分出大多数。
例如，如果它有3个节点，则系统能够容忍1个节点失败；如果有5个节点，则能容忍2个节点失败。

当网络分区出现时，分区的行为是不对称的。一个分区会包含大多数节点。
少数节点的分区就会停止处理，避免在网络分区时出现不一致，但是大多数节点的分区仍然可以继续工作。
这保证了系统的single copy状态是active的。

少数服从多数还能容忍disagreement：如果出现了失败，则节点的投票会不同。
但它只有一个majority decision，这样临时的disagreement至多会阻止协议继续进行（放弃了liveness），但它不会破坏single-copy一致性的原则。

##### 角色

构建系统可能有两种方式：所有节点都有相同的职责；一些节点有不同的角色。

对于复制集的一致性算法，每个节点有各自的角色是一个相对更优的选择。
有一个固定的leader或者master服务器是一种可以使系统更加高效的优化方式，因为所有的更新操作都要经过该服务器。
非leader节点只需要把请求转发给leader。

注意，有不同的角色并不会阻碍系统从leader（或其他角色）的失败中恢复。
因为正常操作时角色固定并不意味着在失败之后它不会被重新分配其他角色（例如，通过leader选举阶段）。
解决可以反复使用上一次leader选举的结果，直到出现节点失败或者是网络分区。

Paxos和Raft都使用不同的解决角色。尤其是，它们有一个laeder（在paxos里叫"proposer"）节点在正常操作时进行协调。
在正常操作时，其余的节点叫follower（在paxos里叫"acceptor"或者"voter"）。

##### epoch

每一个正常操作的周期在Paxos和Raft里叫做一个epoch（在raft里叫"term"）。
在每个epoch里，只有一个节点指定为leader。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/4_epoch.png)

在一次成功的选举之后，会由同一个leader进行协调，直到该epoch结束。
正如上图所示，一些选举可能会失败，这会导致该epoch立即结束。

epoch像是一个逻辑时钟，当过期的节点开始通信时，其他节点能够识别--过期的节点会有更小的epoch数字，这样它们的命令就可以被忽略。

##### 通过竞争的leader变更

在正常操作时，分区容忍一致性算法很简单。正如我们前面看到的，如果我们不在乎容错，我们只要用2PC。
大多数的复杂性来于确保需要做出一致性决定时，它不会丢失数据，以及在节点失败或者网络变化时，协议能够处理好leader变更。

所有节点开始都是follower；一个节点最初会被选为leader。在正常操作时，leader需要保持一个心跳，以便follower检测leader是否失败，或者网络分区。

当一个节点检测到leader不再回应时，它会进入中间状态（在Raft里叫"candidate"），此时它把epoch/term增加1，并开始leader选举，竞争成为leader。

为了成为leader，该节点必须获得大多数投票。最简单的投票方式是，把票投给最先来的；这样，leader最终会被选举出来。
在尝试等待时间里再加上一个随机时间值，会减少同时尝试竞选leader的节点数量。

##### 一个epoch里提议的数值

在每一个epoch里，leader提议一个值进行投票。每个epoch里，每个提议会有一个严格自增的唯一的数字标识。
如果一个特定的值收到多个提议，则follow只会投票给第一个提议。

##### 正常操作

在正常的操作流程里，所有提议都要经过leader节点。当一个客户端提交一个提议，leader会联系所有节点进行仲裁。
如果没有竞争的提议，则leader会提议该值。如果大多数follow接收该值，则可以认为这个值被接受了。

因为有可能另一个节点也会成为leader，我们需要保证，一旦单个提议被接受了，它的值就再也不会改变。
否则，一个已经被接受的提议，可能会被一个竞争的leader更改。Lamport是这样描述的：

```
P2: If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.
```

确保这个提议要求，follower和proposer都被算法限制为，每改变一个值都应该被大多数节点接受。
注意，"the value can never change"指提议的单个处理的值。
一个典型的复制集算法会同时运行多个，但在讨论单一运行会使得问题变得简单。
我们希望避免决策历史被改变或覆盖。

为了具备这样的属性，提议者必须首先询问follower它们（最大数字）接受的提议和具体值。
如果提议者发现该提议已经存在，它必须结束这次提议的执行，而不是做出它自己的提议。
Lamport是这样描述的：

```
P2b. If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposer has value v.
```

更具体地，

```
P2c. For any v and n, if a proposal with value v and number n is issued [by a leader], then there is a set S consisting of a majority of acceptors [followers] such that either (a) no acceptor in S has accepted any proposal numbered less than n, or (b) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the followers in S.
```

这个是Paxos算法的核心，同时算法也从它衍生而来。提议的值在协议的第二阶段之前不会被选择。
提议者必须有时重传之前做的决定以确保安全性，直到它们知道它们可以强制自己的提议值。

如果多个先前的提议存在，则选择最大数字值的协议。提议这只有在没有其他竞争的提议时，才能尝试强制它们自己的值。

为了避免提议时出现冲突，提议者要求follower不要接受比当前数字还低的提议。

把这些规则合在一起，就可以看到在Paxos中要做出决定，需要两轮的通信：

[ Proposer ] -> Prepare(n)                                [ Followers ]

             <- Promise(n; previous proposal number

                and previous value if accepted a

                proposal in the past)

[ Proposer ] -> AcceptRequest(n, own value or the value   [ Followers ]

                associated with the highest proposal number

                reported by the followers)

                <- Accepted(n, value)

准备阶段允许提议者获得任何竞争的或者之前的提议。第二阶段是新的值或者之前接受的值进行提议。
在一些case中--例如，两个提议者在同一个时间激活（dueling）；消息丢失；大多数节点都失败--则没有提议被大多数节点接受。
但这个是可接受的，因为决定规则对什么值应该被提议里，会收敛于一个值。

特别的，根据FLP impossibility结果，我们能做到最好的是：解决一致性问题的算法，当消息传递的bound没有保证时，必须放弃安全性或者活跃性（liveness）。
Paxos放弃了活跃性：它可能会永久延迟做出决定，直到没有leader竞争，而且大多数节点接受一个提议。
相比于破坏安全性保证，这个选择会更好。

当然，实现这个算法比它描述的更难。它们有很多小的点，而代码实现难度却很大，即便是由专家来做。
它们有这些问题：

* 实践优化：a、通过领导合约（leadership lease）来避免重复的leader选举；b、在进入leader身份不改变的稳定状态时，避免重复提议消息；

* 避免follower和proposer在稳定存储里不会丢失item，以及稳定存储里的结果不会丢失（如：磁盘损坏等）

* 集群的成员能以安全的方式变更

Google的文章[Paxos Made Live](https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf)对这些挑战提供更多细节。

#### 分区容错的一致性算法：Paxos，Raft和ZAB 

希望这里能给你一些分区容错一致性算法如何工作的大致感觉。
我希望你能读延伸阅读里的一些文章，以了解不同算法的细节。

Paxos。它是构建强一直分区容错副本系统的最重要的算法。它应用在大多数Google的系统里，包括[BigTable](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/68a74a85e1662fe02ff3967497f31fda7f32225c.pdf) 
/[Megastore](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36971.pdf)
里用到的[Chubby锁管理](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/c64be13661eaea41dcc4fdd569be4858963b0bd3.pdf)，谷歌文件系统，以及[Spanner](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/65b514eda12d025585183a641b5a9e096a3c4be5.pdf)。

Paxos是以希腊一个岛屿命名的，它最早是由Lamport在1998年的一篇名为"The Part-Time Parliament"里介绍的。
它被认为很难实现，于是有很多公司根据自身的实践提供了很多更加实际的细节。
你或许希望能希望读到Lamport对此的评论，可以看[这里](http://lamport.azurewebsites.net/pubs/pubs.html?irgwc=1&ocid=aid2000142_aff_7593_1243925&tduid=(ir__ve1i3hs2q9kftw2mkk0sohzz0u2xlmbw9r19bzhl00)(7593)(1243925)(je6nubpobpq-wto.0vuc6qpfrn5ur.j2da)()&irclickid=_ve1i3hs2q9kftw2mkk0sohzz0u2xlmbw9r19bzhl00#lamport-paxos?ranMID=24542&ranEAID=je6NUbpObpQ&ranSiteID=je6NUbpObpQ-wTO.0VUC6qpFRN5Ur.J2DA&epi=je6NUbpObpQ-wTO.0VUC6qpFRN5Ur.J2DA)
和[这里](http://lamport.azurewebsites.net/pubs/pubs.html?irgwc=1&ocid=aid2000142_aff_7593_1243925&tduid=(ir__ve1i3hs2q9kftw2mkk0sohzz0u2xlmb3ob19bzhl00)(7593)(1243925)(je6nubpobpq-kyjvcxbn_0t9drwmezcmxq)()&irclickid=_ve1i3hs2q9kftw2mkk0sohzz0u2xlmb3ob19bzhl00#paxos-simple?ranMID=24542&ranEAID=je6NUbpObpQ&ranSiteID=je6NUbpObpQ-kyJvcXbn_0t9dRwMEzCMXQ&epi=je6NUbpObpQ-kyJvcXbn_0t9dRwMEzCMXQ)。

Paxos在做决定的时候是单轮的（single round），但实际在实现的时候，希望高效运行多轮一致性算法。
因而又衍生出了许多[该协议的扩展](http://lamport.azurewebsites.net/pubs/pubs.html?irgwc=1&ocid=aid2000142_aff_7593_1243925&tduid=(ir__ve1i3hs2q9kftw2mkk0sohzz0u2xlmb3ob19bzhl00)(7593)(1243925)(je6nubpobpq-kyjvcxbn_0t9drwmezcmxq)()&irclickid=_ve1i3hs2q9kftw2mkk0sohzz0u2xlmb3ob19bzhl00#paxos-simple?ranMID=24542&ranEAID=je6NUbpObpQ&ranSiteID=je6NUbpObpQ-kyJvcXbn_0t9dRwMEzCMXQ&epi=je6NUbpObpQ-kyJvcXbn_0t9dRwMEzCMXQ)。
此外，还有一些实践挑战，例如如何优化集群的成员变更。

ZAB. ZAB-Zookeeper原子广播协议用于Apache Zookeeper项目。
Zookeeper提供了很多分布式系统的协调调度原语(coordination primitives)，它用于很多Hadoop-centric的分布式系统项目，如：HBase，Storm，Kafka等。
Zookeeper是基于开源社区的Chubby版本。纯技术来讲，原子广播（atomic broadcase）跟纯一致性问题是不同的，但它仍然归于保证强一致性的分区容忍性算法。

Raft. Raft是最近（2013）年的算法。它最初设计是为了比Paxos更容易用于教学，而且它提供了相同的保证。
该算法最大的不同是，它描述了一个集群成员变更的机制。它的开源项目版本是[etcd](https://github.com/etcd-io/etcd)。

#### 强一致性的复制集方法

在这一节里，我们看了一些保证强一致性的复制集方法。
首先对比了同步和异步，然后探讨了更多的算法，它们的失败的复杂度是不断增加的。
以下是算法的一些关键特性。

##### Primary/Backup

* Single, static master

* Replicated log, slaves are not involved in executing operations

* No bounds on replication delay

* Not partition tolerant

* Manual/ad-hoc failover, not fault tolerant, "hot backup"

##### 2PC

* Unanimous vote: commit or abort

* Static master 

* 2PC cannot survive simultaneous failure of the coordinator and a node during a commit

* Not partition tolerant, tail latency sensitive

##### Paxos 

* Majority vote

* Dynamic master

* Robust to n/2-1 simultaneous failures as part of protocol

* Less sensitive to tail latency

#### 延伸阅读

##### 主备/2PC

* [Replication techniques for availability - Robbert van Renesse & Rachid Guerraoui, 2010](https://www.researchgate.net/profile/Robbert_Van_Renesse/publication/221029788_Replication_Techniques_for_Availability/links/0c96052b26a4fc846a000000.pdf)

* [Concurrency Control and Recovery in Database Systems](https://courses.cs.washington.edu/courses/cse551/09au/papers/CSE550BHG-Ch7.pdf)

##### Paxos

* [The Part-Time Parliament - Leslie Lamport](http://lamport.azurewebsites.net/pubs/lamport-paxos.pdf?ranmid=24542&raneaid=je6nubpobpq&ransiteid=je6nubpobpq-a0jo7p2g.7n1_ktmvl9yja&epi=je6nubpobpq-a0jo7p2g.7n1_ktmvl9yja&irgwc=1&ocid=aid2000142_aff_7593_1243925&tduid=(ir__ve1i3hs2q9kftw2mkk0sohzz0u2xlmbloi19bzhl00)(7593)(1243925)(je6nubpobpq-a0jo7p2g.7n1_ktmvl9yja)()&irclickid=_ve1i3hs2q9kftw2mkk0sohzz0u2xlmbloi19bzhl00)

* [Paxos Made Simple - Leslie Lamport, 2001](http://lamport.azurewebsites.net/pubs/paxos-simple.pdf)

* [Paxos Made Live - An Engineering Perspective - Chandra et al](http://www8.cs.umu.se/kurser/5DV153/HT14/literature/chandra2006paxos.pdf)

* [Paxos Made Practical - Mazieres, 2007](http://read.seas.harvard.edu/~kohler/class/08w-dsi/mazieres07paxos.pdf)

* [Revisiting the Paxos Algorithm - Lynch et al](https://groups.csail.mit.edu/tds/papers/DePrisco/WDAG97.pdf)

* [How to build a highly available system with consensus - Butler Lampson](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.72.5429&rep=rep1&type=pdf)

* [Reconfiguring a State Machine - Lamport et al - changing cluster membership](https://lamport.azurewebsites.net/pubs/reconfiguration-tutorial.pdf)

* [Implementing Fault-Tolerant Services Using the State Machine Approach: a Tutorial - Fred Schneider](https://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf)

##### Raft和ZAB 

* [In Search of an Understandable Consensus Algorithm, Diego Ongaro, John Ousterhout, 2013](https://raft.github.io/raft.pdf)

* [Raft Lecture - User Study](https://www.youtube.com/watch?v=YbZ3zDzDnrw)

* [A simple totally ordered broadcast protocol - Junqueira, Reed, 2008](https://www.datadoghq.com/pdf/zab.totally-ordered-broadcast-protocol.2008.pdf)

* [ZooKeeper Atomic Broadcast - Reed, 2011](https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=5958223)


### 5. 复制集：弱一致性模型协议

现在，我们已经看了一些强制single-copy一致性的协议，现在我们把目光转到现实的一个情景中--它并不要求single-copy一致性。

总的来说，很难找出一个单一维度的定义允许副本有分歧的协议。
大多数这样的协议是高可用的，而且其关键不仅仅是终端用户是否发现其保证、抽象或者是API是他们想要的，尽管事实是，在节点/网络出现错误时，副本是有分歧的。

为什么弱一致性系统并没有很流行？ 

正如我在介绍里说的，我认为大多数分布式编程在解决分布式带来的后果时，有两个隐含的假设：

* 信息以光速传播

* 相互独立的事件会相互独立地失败

在单节点上计算是容易的，因为每件事都以一个可预测的全局顺序发生。在分布式系统上计算是复杂的，因为没有全局的顺序。

很早之前，我们就通过引入全局的顺序（global total order）来解决这个问题。
我已经讨论了很多方法，通过创造顺序来达到强一致性 ，尽管本质上没有出现全序（ there is no naturally occurring total order）。

当然，这里的问题是，强制顺序的代价是很大的。
This breaks down in particular with large scale internet systems, where a system needs to remain available。（break down在这里怎么理解）
一个强一致性的系统并没有表现地像一个分布式系统：它像一个单一系统，而在出现分区时，它会影响可用性。

此外，对于每一个操作，通常必须要接触大多数节点，而且通常不止一次，而是两次（正如在2PC里面讨论的）。
这对于物理上分布式、而又需要对全局用户提供足够的性能的系统来说，是很痛苦的。

因此，默认表现得像一个单一系统，或许并不是我们所期望的。

或许，我们需要的系统是，我们不需要用到太多的协调调度，而且能够返回一个"可以用"（usable）的值。
相比于只有一个唯一的真值，我们可以允许不同副本之间有差异--不仅可以保持效率，而且容忍分区--同时尝试找到一个方式在某种程度上解决分歧。

最终一致性表达了这个思路：节点可以在某些时间彼此不同，但最终它们会达成统一值。

对于提供最终一致性的系统，它们有两种类型的设计：

概率保证的最终一致性（Eventual consistency with probabilistic guarantees）。
这类型能够在随后检测出写冲突，但是并不保证结果等于以某种正确的顺序执行得到的结果。
换句话说，更新冲突有时会导致用旧的值覆盖了新的值，而且一些错误可能会出现在正常的操作中。

近些年，提供single-copy一致性的最有影响力的系统是亚马逊的Dynamo，我们在稍后把它做为一个提供了概率保证的最终一致性的系统的例子。

强保证的最终一致性（Eventual consistency with strong guarantees）。
这类型保证最终结果收敛到一个值，它等于以某种正确顺序执行得到的结果。
换句话说，这类系统并不产生任何异常结果；不需要任何协调，你就可以构建相同服务的副本，这些副本可以以任何方式通信，以任何顺序收到更新消息，而它们最终会对最后结果达成一致，只要它们看到的是相同的信息。

CRDT（convergent replicated data types）是一种数据类型，它们保证收敛到相同的值，不管是网络延迟、分区还是消息乱序。
它们证明是收敛的，但是数据类型必须按CRDT限制的来实现。

CALM也是另一个选择：它等同于逻辑单调一致（原话是，it equates logical monotonicity with convergence）
如果我们能确认某件事是逻辑单调的，则它可以安全地运行而不需要协调。
合流分析（confluence analysis）--用Bloom编程语言实现的--可以用于指导编程者决定，何时何地用强一致系统的协调技术（coordination technique），何时又能安全地不使用协调技术。

#### 协调不同的操作顺序

一个不强制single-copy一致性的系统是什么样的？我们通过一些具体的实例来看。

或许该系统最明显的一个特征是，它允许副本彼此不同。这意味着，它没有强制定一个通信的模式：副本可以彼此分离，能够继续工作，并且能够接受写请求。

假设一个系统有3个副本，彼此之间相互分离。例如，每一个副本可以是三个不同的数据中心，由于某种原因没法相互通信。
每个副本在分区中都保持可用，接受来自客户端的读写请求：

[Clients]   - > [A]

--- Partition ---

[Clients]   - > [B]

--- Partition ---

[Clients]   - > [C]

过一段时间之后，物理分区修复了，副本服务器开始交互信息。他们由于已经从不同服务器接收不同的更新请求，因而他们彼此之间存在差异，因而需要某种协调。
我们期望的是，所有副本都收敛到相同的结果。

[A] \
    --> [merge]
[B] /     |
          |
[C] ----[merge]---> result

另一种理解弱一致性的方式是，假设一些服务器以某种顺序给两个副本发送消息。
由于没有一致性协议，强制一个唯一的总序（total order），因而消息在两个副本的顺序可以不同：

[Clients]  --> [A]  1, 2, 3

[Clients]  --> [B]  2, 3, 1

这也就是本质上为什么我们需要一致性协议。例如，我们希望链接一个字符串，而三个操作的消息如下：

1: { operation: concat('Hello ') }

2: { operation: concat('World') }

3: { operation: concat('!') }

如果没有一致性卸掉，A会输出"Hello World!"，而B会输出"World!Hello"。

A: concat(concat(concat('', 'Hello '), 'World'), '!') = 'Hello World!'

B: concat(concat(concat('', 'World'), '!'), 'Hello ') = 'World!Hello '

这当然是不对的。我们希望的是两个不同的副本收敛到相同的结果。

记住这两个例子。我们先看Amazon的Dynamo系统，来建立一个基础的概念，然后介绍一些有意思的构建弱一致性系统的方法，例如CRDT和CALM理论。

#### Amazon的Dynamo 

Amazon的Dynam系统可能是最知名的提供弱一致性、高可用性的系统了。
它是现实很多其他系统的基础，包括Linkin的Voldemort，Facebook的Cassandra和Basho的Riak。

Dynamo是最终一致性，高可用的key-value存储。
key-value存储就像一个大的哈希表，客户端可以通过设置（key，value）来设置一个值，并可以通过key来检索。
Dynamo集群包括N个节点；每个节点有一些key的节点来存储。

Dynamo对可用性的要求高于一致性；它并不保证single-copy一致性。
相反的，在写的时候，副本之间可以不同；在读的时候，会有一个读协调阶段（read reconciliation phase）可以在返回客户端之前尝试解决不同副本之间的不一致。

对Amazon的许多特性来说，避免断电（outage）比保证数据绝对的一直更重要，因为断电会导致商业和信誉上损失。
此外，如果数据不是非常重要，弱一致性系统相比于传统的RDBMS，能够以更小的代价来提供更好的性能和更高的可用性。 

由于Dynamo是一个完整的系统设计，出了核心的副本任务，它还有很多不同的部分。
下图阐释了一些任务；最重要的是，一个写操作如何路由到一个节点，并写到不同的副本。 

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/5_dynamo_write_process.png)

在看了写操作最初如何被接受的，我们再来看冲突检测，以及异步副本同步任务（asynchronous replica synchronization task）。
副本同步任务保证即便节点失败了，它也能很快跟上来。

#### 一致性哈希

不管是读还是写，我们需要做的第一件事是确定数据在系统的什么位置。
这需要某种key-to-node的映射。

在Dynamo中，key通过[一致性哈希](https://github.com/mixu/vnodehash)的方式映射到节点（在这里我们不讨论细节）。
它的主要思路是，在客户端做简单的计算，然后把一个key映射到一些节点中的目标节点。
这意味着，客户端本地保存key，而不需要查询系统来获得每一个key的位置；这样会节省系统资源，因为哈希通常会比RPC更快。

#### 部分仲裁 

当我们知道一个key存在什么地方，我们需要做的是持久化（persist）它。
这是一个同步的任务；我们需要马上把这个值写到多个节点的原因是为了提供持久性（例如，在节点立即失败时进行保护）。

像Paxos和Raft一样，Dynamo使用副本仲裁（quorum for replication）。
然而，Dynamo的仲裁是部分仲裁（partial quorum）而不是严格仲裁（strict quorum）。

正式地讲，一个严格的仲裁系统是任何两个仲裁集相互重叠。
（Informally, a strict quorum system is a quorum system with the property that any two quorums in the quorum system overlap）。
它在更新之前需要大多数的投票，这样保证了只有一个历史记录提交，因为大多数仲裁（majority quorum）的机制使得必须至少有一个节点重叠。
这也是Paxos所依赖的属性。

而部分仲裁并没有这个属性；这意味着"少数服从多数"并不是必须的，而不同的仲裁集可能对同样的数据有不同的版本。
用户可以选择节点数量来写或者读：

* 用户可以选择W-of-N节点要求写成功；

* 用户可以在读的时候，指定R-of-N；

W和R分别指定了写和读的时候要多少个节点。写的时候要求更多节点，可能会慢一点，但是增加了数据不会丢失的概率；
从更多的节点读数据，增加了读到最新数据的概率。

通常推荐的是R+W>N，因为这意味着读写仲裁至少有一个节点重叠--使得返回错误数据的几率降低。
一个典型的配置是N=3；这意味着用户需要在以下值做出决定：

R = 1, W = 3;   R = 2, W = 2 or R = 3, W = 1

更通用的是，假设 R+W >N： 

* R=1,W=N : 读很快，但是写很慢；

* R=N，W=1 ： 写得快，但是读得慢；

* R=N/2， W/2+1 ： 两者平衡。 

N很少大于3，因为保存很多份数据的拷贝，成本很高。

正如我之前提到的，Dynamo那篇论文对很多类似的系统设计很有启发。
他们基于数据副本的方法，使用了部分仲裁，但对N，W和R有不同的默认值：

* Basho's Riak (N = 3, R = 2, W = 2 default) 

* Linkedin's Voldemort (N = 2 or 3, R = 1, W = 1 default) 

* Apache's Cassandra (N = 3, R = 1, W = 1 default)

这里还有一个细节：当发送一个读或写请求，是所有N个节点要回应（Riak），还是只要有最少的节点回应（如 R或；Voldemort）。
"send-to-all"的方法更快，而且对延时不敏感（因为，它只要等待N里面最快的W或R个节点）；
而"send-to-minimum"方法对延时更加敏感（因为，单点的通信延迟会导致操作延迟），但它也是很高效的（更少的信息和链接）。

当R+W>N时，会发生什么？更具体地说，通常把这个结果称为"强一致性"。

#### 是否R+W>N等同于"强一致性"？ 

不。

一个系统的 R + W > N 能探测读/写冲突，因为任何读仲裁和写仲裁都有一个共同的成员。
例如，至少一个节点在两个仲裁中： 

   1     2   N/2+1     N/2+2    N

  [...] [R]  [R + W]   [W]    [...]
  
这保证了写的操作，会被后续一个读操作看到。 
当然，这里的前提是节点数N不再变化。而这个是Dynamo不具备的，因为在Dynamo中在节点失败时，集群的成员是会改变的。

Dynamo设计为总是可以写的。它有一个机制处理节点错误：在之前的服务器宕机时，增加一个不同的、不相关的服务器到节点集中，这个服务器负责一部分特定的key。
这意味着，仲裁协议不再保证总是有overlap。
即便是 R=W=N也不能满足，因为当仲裁的大小等于N，在失败时，仲裁的节点会改变。
具体地说，在一个分区中，如果有足够数量的节点不可达，Dynamo会增加新的节点，它虽然不相关，但是可以访问。

此外，Dynamo并不像一个强一致性的模型一样去处理分区：也就是，写能在两边的分区进行，这意味着，至少在某段时间里，系统并没有显示得像一个single-copy。
因而，说R+W>N是"强一致性"是有误导的；它只是在概率上--而这跟强一致性的定义是不一样的。 

#### 冲突检测和读修复 

一个允许副本差异的系统必须要有一种方式来最终协调两个不同的值。
正如前面介绍的，采用的方式是在读的时候检测冲突，然后采用一些冲突解决方法。那么，该如何做？ 

大致来说，是通过元数据来追踪数据的因果历史（casual history）。
客户端从系统读取数据时，必须保存元数据的信息，而写入数据库时，必须返回元数据的值。 

我们已经有一个方法来做这个：向量始终可以用于表示一个值的历史。
特别的，这就是Dynamo最初用于检测冲突的设计。

然而，使用向量时钟并不是唯一的选择。如果你看很多实际的系统设计，你可以大致推断出它们如何使用元数据。

_没有元数据_。当一个系统不跟着元数据，而只返回值，它实际上对并发写并没有做太多的事情。
通常的规则是，采用最后的写结果，换句话说，如果两个writer同时写，只会保留最慢的writer的结果。

_时间戳_。名义上是采用有最大时间戳的值。然而，如果时间没有很好地同步，一些奇怪的情况就会出现，如在系统出错时，会写旧的数据，或者是最快的时钟会覆写最新的值。
Facebook的Cassandra是Dynamo的变体，它使用的就是时间戳而不是向量时钟。

_版本号_。版本号或许能避免使用时间戳遇到的一些问题。注意的是，在多个历史值发生时，最基本的准确追踪的机制是向量时钟，而不是版本号。 

_向量时钟_。使用向量时钟，并发数据和旧的数据会被检测出来。进行读修复（read repair）因此称为可能，尽管有时我们会需要问客户端来决定一个值。
这是因为如果变化是并发的，我们其实对数据一无所知，因此最好是询问，而不是随机丢弃。

当读一个值的时候，客户端联系N个节点中的R个，询问它们一个key的最新值。
它会获得所有的应答，并丢弃严格意义上的旧值（使用向量时钟来检测）。
如果只有唯一的向量时钟+值，它会返回。如果有多个向量时钟+值，则所有的值都会返回。

显然，读修复会返回多个值。这意味着客户端/应用开发者必须处理这样的情况，根据一些特定的规则选一个值。

特别地，实际的向量时钟系统的一个关键部分是，时钟不能永远地增长，因此它需要一个过程（procedure）来安全地进行垃圾回收，以平衡容错和存储要求。

#### 副本同步：gossip and merkle tree 

既然Dynamo系统设计是节点容错和网络分区，它需要处理节点在分区之后，重新加入集群的问题，以及失败节点被替代或部分恢复。

副本同步用于在失败之后，使得节点同步到最新，以及周期性地同步副本。

Gossip是一个概率的技术用于同步副本。它的通信模式（例如节点之间相互联系）不会提前决定。
相反的，节点以一定概率p尝试相互同步数据。每t秒，每一个节点会选一个节点来通信。
这提供了异步任务之外的一种额外的机制来使得副本更新。 

Gossip是一种可扩展的，但它只能提供概率的保证。

为了使副本同步时信息交换有效率，Dynamo使用了一种叫做Merkle tree的技术。
这里我不会展开讲。它的关键思路是，数据存储可以哈希成不同的粒度：用哈希代表整个内容，一般的key值，四分之一key值等。

通过保存这个粒度的哈希，节点可以更高效地对比他们的数据存储。
当一个节点的唯一key有不同值时，它们只要交换一些必要的信息，就可以使得副本更新。

#### Dynamo实践：probabilistically bounded staleness（PBS）

Dynamo系统设计包括以下：

* 一致性哈希决定key的位置

* 读写的部分仲裁 

* 通过向量时钟来解决冲突检测和读修复 

* gossip用于副本同步 

我们如何特征化这样的系统的行为？Bailis等人提出了一种PBS的方法，用模拟和从现实系统收集的数据来特征化这样系统所预期的行为。

PBS使用关于anti-entropy的信息来估计不一致性度（degree of inconsistency ），用网络延时和本地处理延时来估计读的一致性等级。
它已经在Cassandra中实现，它的时间信息通过其他消息携带，并基于蒙特卡洛模拟的方式来进行估计。

在这篇文章中，在普通操作中，最终一致性数据存储通常更快，而且会在10-100毫秒达到读一直的状态。
下面的表展示了在不同R和W的情况下，以99.9%概率达到读一致性的数据。

![image](https://github.com/linzhixia23/go-reading/blob/master/Distributed%20Systems%20for%20Fun/pic/5_PBS_table.png)

更多细节可以在[PBS](http://pbs.cs.berkeley.edu/)的官网查询。 












