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

正如 Nietzsche 写的：

Every concept originates through our equating what is unequal. 
No leaf ever wholly equals another, and the concept "leaf" is formed through an arbitrary abstraction from these individual differences, through forgetting the distinctions;
and now it gives rise to the idea that in nature there might be something besides the leaves which would be "leaf" - some kind of original form after which all leaves have been woven, marked, copied, colored, curled, and painted,
but by unskilled hands, so that no copy turned out to be a correct, reliable, and faithful image of the original form.

抽象本质上就是假的。每一个环境都是独特的，正如每一个节点。
但是抽象让世界更容易管理：简化问题，使得更容易分析，而且没有忽略任何关键的东西，而结果广泛适用。


