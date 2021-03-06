# Spanner

### 摘要

Spanner是Google的一个可扩展，多版本控制，全球分布式的同步复制数据库。
它是第一个支持外部一致性事务的全球规模分布式存储系统。
这篇论文主要介绍了Spanner的功能、架构以及这些决定背后的理论依据。
另外，本文也介绍了一个处理时钟不确定性的创新接口。
这个接口的设计及其实现对于支持外部一致性和Spanner的很多关键性功能是至关重要的。
这些关键性功能包括：非阻塞旧数据读取，无锁的只读事务，数据表结构原子更改等。

## 1 介绍

Spanner是由Google设计、开发和部署的一个可扩展的，全球分布式数据库。简单地看，它是一个通过多个分布在不同数据中心的Paxos[21]状态机来存储数据切分的数据库。它利用了Paxos的副本机制来保证全球可用性(Availability)和区域的局部性(Locality)。客户端会自动地进行副本之间的故障转移(failover)。当数据或者服务器数量变化时，Spanner会自动地重新切分和分布数据（可以分步到不同数据中心）来更好地平衡负载和应对可能的错误。Spanner是为了在数百个数据中心中的数百万个数据服务器中存数千亿级别的数据行设计的。

通过州内甚至洲际数据复制，应用程序可以利用Spanner在大范围自然灾害的情况下仍旧保持高可用。Spanner的第一个应用程序是F1 [35]。F1是对Google的广告后台的重写。F1使用了五个分布在美国不同地区的副本。大部分其他的应用大都采用了在同一个地区的三到五个相对独立的副本。也就是说如果可以承受一到两个数据中心的错误，大部分应用相对于更高可用会偏向更低延迟。

尽管Spanner的主要关注在于管理跨数据中心的数据复制，但是我们仍旧尽了非常大的努力利用这个分布式的基础设施来设计和现实关键性的数据库功能。尽管很多项目在顺利的使用Bigtable [9], 但是我们还是不断地收到用户对它在某些情况下很难使用的抱怨：有一些用户需要复杂、可变的数据表结构；有些需要在跨区域复制的情况下仍然达到数据的强一致性。（其他的一些作者也有同样的看法 [37]）尽管Megastore [5]的吞吐量相对较差，很多Google的应用选择了它。因为它提供了半关系型的数据模型并且支持同步复制。通过这些经验，Spanner演化类似于Bigtable的版本控制的键值存储到一个临时性多版本控制数据库。数据被存储在有数据表结构的一个半关系型表内；数据是有版本控制并且每个版本都被自动的标注了在提交时的时间戳；旧版本的数据会被垃圾回收机制根回收规则来回收；应用可以根据时间戳来读取旧的数据。Spanner支持通用的事务并且提供了一个基于SQL的查询语言。

Spanner作为一个全球分布式的数据库提供了很多独特的功能。首先，应用可以细粒度、动态地控制Spanner的复制配置。这些配置包括：数据分配到的数据中心，数据距离用户的距离（控制读取的延时），副本之间的距离（控制写入的延时），维护副本的数量（控制持久性、可用性和读取的吞吐量）。数据还可以被动态、透明地在数据中心之间移动来平衡各个数据中心的资源使用情况。其次, Spanner提供了两个在分布式数据库中非常有挑战性的功能：外部一致[16]的读写以及全球一致的根据时间戳的读取。这些功能使得Spanner可以支持一致的备份、一致的MapReduce操作[12],以及原子性的数据表结构更新。这些操作均可以在全球规模，甚至是有正在进行的事务的情况下完成。

Spanner能够给分布或者非分布的事务分配有意义的全局时间戳，从而使这些功能变得可行。这些时间戳可以反映序列化顺序（Serialization Order）。另外，这些序列化顺序可以满足外部一致性（即线性一致性（linearizability）[20]：如果一个事务T1的提交时间早于另外一个事务T2的开始时间，那么事务T1的提交时间戳小于T2的提交时间戳（译者注：一定注意是T1的最后提交时间早于T2的用户请求开始时间）。Spanner是第一个能在全球规模提供这种保证的系统。

能够提供这些特性的关键在于一个创新的TrueTime的接口和实现。这个接口直接暴露了时钟的不确定性。Spanner关于时间戳的保证也是依赖于接口实现的时钟不确定性的界限。如果这个不确定性很大，那么Spanner会增加延迟来等待不确定性的界限。Google的集群管理系统实现了TrueTime的接口。这个实现采用了多个现代的参考时钟（GPS以及原子钟）来保证了低时钟不确定性（通常小于10毫秒）。（译者注：实际上最大等待时间不超过10毫秒）

章节二主要介绍Spanner的实现，功能以及设计相关的工程决定。章节三介绍了创新的TrueTime接口和其简略的实现。章节四介绍了Spanner如何利用TrueTime接口来实现外部一致事务、无锁只读事务和原子数据表结构更新。章节五提供了Spanner的性能测试情况以及TrueTime的表现情况并且讨论了关于F1的使用经验。章节六、七、八分别对相关工作、今后方向进行了介绍并对全文进行总结。

## 2 实现

本节主要介绍Spanner的结构以及实现原理。其次介绍了用于管理复制和局部性的抽象目录结构,它也是移动数据的基本单元。最后介绍了Spanner的数据模型，Spanner相比于键值存储更类似关系型数据库的原因，以及应用如何控制数据局部性。

每一个Spanner的部署称作为一个universe。一个Spanner universe可以管理全球规模的数据，因此通常只会运行很少的universes。我们目前运行了一个用于测试的universe, 一个用于开发、线上的universe以及一个只针对线上的universe。

Spanner universe是由多个zones组成的。每个zone都类似于一个Bigtable [9]服务器组的部署，也是管理的基本单元。这些zones的集合也是数据被复制的位置的集合。当有新的数据中心投入运行时，Zones可以被添加到在运行的Spanner系统中；当有旧的数据中心被关闭时，Zones可以被从正在运行的系统中移除。Zones还是物理隔离的基本单元，在某些情况下每个数据中心中可能会有多个Zones：比如，属于不同应用的数据必须被分区存储到一个数据中心的不同的服务器集合之中。

图1展示了在一个Spanner universe中的服务器。一个zone包括一个zonemaster以及一百至几千个spanserver。zonemaster负责分配数据给spanservers；spanservers负责把数据提供给用户。客户通过每个zone上的location proxy来找到已经分配为它提供数据的spanserver。目前universe master和placement driver都只有一个。Universe master主要作为一个控制台来显示关于所有zones的信息并且提供互动调试。Placement driver管理数据在zones之间进行分钟级别的移动。它会定期地于spanservers通讯来确定因复制约束变更或者平衡负载而需要被移动的数据。由于文字有限，我们讲仅详细介绍spanserver。

### 2.1 Spanserver软件栈

本节主要关注spanserver的实现,并介绍如何在基于类似Bigtable的tablet上实现复制和分布式事务。图2显示了spanserver的软件栈。在最底部，每个spanserver负责管理成百上千个被称为tablet的数据结构实体。Tablet类似于Bigtable的tablet，因为它也实现了如下的映射关系：
```
(key:string, timestamp:int64) -> string
```
不同于Bigtable, Spanner会分配给tablet数据时间戳。 这也是Spanner相比于键值存更像多版本数据库的重要原因。每一个tablet的状态被存在一个类似B-tree的文件集合和一个write-ahead日志中。文件和日志都被存储在Colossus(即Google File System [15]的继承系统)分布式文件系统中。

为了支持复制，spanserver在每个tablet上实现了一个Paxos状态机。（最初, Spanner在每个tablet上实现了多个Paxos状态机。这样可以更灵活地配置副本。我们最终放弃了这样的实现，因为它过于复杂。）每一个状态机都在对应的tablet上存储了它的元数据和日志。我们的Paxos实现支持long-lived leaders以及基于时间的leader lease。lease的默认时间是10秒。目前Spanner的实现记录每个Paxos的写入操作两次：一次在tablet的日志中，另一次在Paxos的日志中。我们仅仅是为了方便而这样实现, 我们很可能在今后对它进行改进。（译者注：大部分的已知一致性系统都是存储两次日志数据：在一致性协议中和应用中。合并日志并不是一个简单的工作，因为它还涉及到日志的Compaction等问题。）我们的Paxos实现了pipeline来提升在广域网高延时情况下的吞吐量；写入操作仍旧是按照顺序被Paxos来执行的（我们第四章节依赖这个事实）。

Paxos状态机是用来保证映射副本的一致性。每个副本的键值映射都被存在相应的tablet中。写入操作必须由Paxos协议的leader来发起；读取操作可以访问任意一个足够新（译者注：足够新即replica读取返回的结果是读请求到达时最新的或者更新的。例如读请求到达时刻为T0，读返回时刻为T1。返回的的结果必须保证是一个在T0-T1时间范围内某个在所有副本中最新的值。）的replica（译者注：在讨论读写操作所时，Paxos中的状态机一般被称为replica）。每个Paxos单元是由一组replica构成的（译者注：一般由3到5个replica构成）。

## 5 评测

首先我们会对Spanner三个方面的性能进行测试，数据复制，事务以及可用性。然后我们会展示一些关于TrueTime行为的数据，以及Spanner的首个用户，F1的一些数据。

### 5.1 微基准测试
表3展示了Spanner的一些微基准测试结果。这些结果都是在分时机器上测试得到的，其中每个spanserer运行在4GB内存以及4核（ADM Barcelona 2200MHZ）的调度单元上。各client则是运行在独立的机器上。每个zone包含一个spanserver。所有的client以及zone都被放置在一组网络通信时长小于1ms的数据中心里面。（这样的场景应该很常见：大多数的应用并不需要在全球范围内分布它们的数据。）这个测试的数据库拥有50个Paxos group，每个Paxos group拥有2500个directory。测试的操作包含相互独立的4KB读和写。所有读操作的内容都直接放置在内存上，这样一来，我们测试得到的只是Spanner的调用栈的耗时。另外，在测试前还进行了一轮未经测量的读操作，目的是让位置信息提前得到缓存。
对于延时的实验，为了避免操作在服务端排队，各Client发起的操作要足够少。从单一副本的实验可以看到，commit wait大概是5ms，Paxos的延时大概是9ms。随之副本数量的增加，总体的延时基本保持不变，而与此同时，偏差却越来越小。这是因为随着副本数量的增加，达到quorum所需的时间受到单一缓慢节点的影响也会随之变小。
对于吞吐量的实验，各client需要发起足够多的操作来让各server的CPU达到饱和。由于快照读可以在任何最新的副本上面执行，所以它们的吞吐量随着副本数量的增加而增加。只有一个读取项的只读事务只会在leader执行，这是因为时间戳只能由leader分配。随着副本数量的增加，由于有效spanserver的数量增加，只读事务的吞吐量也随之增加。在只读事务的实验设定中，spanserver的数量与副本的数量等同，且leader被随机分散到各个zone中。写吞吐量也同样受益于同样的实验假象#FIXME（这可以解释为什么从3个副本增加到5各副本时，写吞吐量反而上升了），但是随着副本数量的增加，每次写操作的工作量也会线性增加，最终会把这个收益抵消掉。
表4展示出二段提交可以应用到一个合理数目的参与者集合中。结果总结自一系列的实验，其中实验环境包含3个zone，每个zone包含25个spanserver。就延时而言，无论对于平均值还是99百分位数，50个参与者都是一个合适的数目。而且，当参与者数目达到100以后，延时就开始显著地增长。
