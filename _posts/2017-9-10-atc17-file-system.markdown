---
layout: default
title:  "收割大型数据中心的空闲存储资源-ATC17论文分析"
date:   2017-09-10
categories: filesystem
---

好久没分析论文了，ATC17论文也出来2个多月，挑几篇相关的阅读下，近期一直在关注分布式的文件系统的思路，正好ATC有一篇：“Scaling Distributed File Systems in Resource-Harvesting Datacenters”，这名字确实不好翻译，核心就是解决大规模数据中心中提升资源利用率的技术方案，详细分析下看看后续是否有价值参考。

## 1 整体介绍

建立大型数据中心非常昂贵，但是很多时候资源利用率又不高，于是各种应用融合在一起使用数据中心资源成为了一种提高利用率的基本方式。将大量的batch工作同时&分时的跑在一个集群中，充分利用空闲的资源；但是这么做之后，会有一些问题要解决，比如：

- 长期运行的服务需要能够被隔离出来，性能不受影响
- batch任务的执行，也不受长期运行的服务的干扰

前人的工作给了一些方案——History-Based Harvesting of Spare Cycles and Storage in Large-Scale Datacenters，OSDI16，但是这个方案扩展性不够，本文主要解决扩展性问题。

当前的主要分布式文件系统扩展性不过几千节点，一般都需要建很多个资源池来分摊负载，此时需要上层业务来进行业务的切割，管理起来有以下几个问题：

- 用户需要自己了解各个集群的分区和名字空间
- 很容易导致各个集群负载不均衡
- 管理员需要人工的管理数据和名字空间的映射关系
- 当负载很复杂的时候，管理员也不知道如何是管理数据

还有一种方式是增强元数据的扩展性，比如GFS的下一代Collossus，不过也有问题：

- 系统会变得非常复杂
- 软件的bugs，失败和管理员误操作会导致非常严重的问题

为了解决以上分布式软件的扩展性问题，论文提出了新的方案：

- 在多个分布式资源池和客户端之间加了一层软件，这个软件可以自动的管理多个子名字空间，最终组成一个巨大的联邦分布式文件系统。
- 核心目标：1）避免多个业务的相互干扰；2）性能优秀，资源利用高，避免负载不平衡

整个方案分2个步骤：

- 将servers分配到不同的子集群，直接使用DHT搞定
- 将目录和文件分配到不同的子集群，并且当某个子集群负载快满的时候，能够进行迁移

实现方法：

- 直接基于HDFS来玩，取了个新名词叫DH-HDFS，选择HDFS主要是因为：1）HDFS用的广泛；2）项目的核心也是想解决分析类负载的问题
- 直接在一个真实的环境中来做实验，在一个巨大的数据中心中，模拟了10个集群；测试结果表现良好，极大的提升了数据的持久度和可用性指标，性能几乎没有损失。
- 当前已经有4个生产环境的使用，最大的集群有超过19k的服务器，6个子集群。
- 当然，这个技术不仅限于HDFS，任何类似的子集群任务切割都会用的上。（比如我们多个CloudFS之间的任务切割也应该用得上）

论文的核心贡献：

- 提出一种扩展分布式文件系统的方法，能够很好的解决海量规模下存储的容量，数据持久度，可用性和性能问题。并且：1）不修改任何现有逻辑，仅仅在客户端和文件系统之间叠加一个中间层；2）一致性hash的方式来创建子集群；3）动态的文件迁移。
- 基于HDFS做了实际的工程实现，并在生产环境使用
- 使用真实负载去测试分析
- 阐述了整个系统的设计经验

## 2 背景

几个定义；

- 首席租户：在大规模数据中心中，大多数服务器分配给原生的，低延迟的负载，由于对于延迟的要求极地，一般直接使用服务器的本地FS，这种负载叫“primary tenant”（首席租户）。
- 二流租户：一些低优先级的业务，比如batch类的分析业务，能够消耗首席租户多余的资源和存储空间
- 首席租户优先级搞，资源不够的时候有限保证他的运行，必要时候可以杀掉二流租户的业务
- 一般来说首席租户拥有这些服务器，并且能够自由的去重新安装业务和格式化数据（为啥没事去格式化硬盘呢？应该是格式化OS盘，导致机器重启）

当前很多互联网已经是全共享式数据中心了，比如Google，Facebook，各种不同的业务有不同的优先级。

### 2.1 多样的副本放置方式

数据放置的挑战：

- 如果把所有的首席租户都放在一起，峰值的时候，很容易造成不可用
- 如果管理员或者开发者格式化一个OS，这个服务器正好包含一个首席租户的所有数据，那么就会造成短暂的数据丢失

OSDI16那篇论文给了个解决方案：不允许数据的多个副本落在同一个逻辑的或者物理的集群上（比如物理的rack）

这篇论文也是基于此来做，支持在集群创建了一个集群的联邦，并解决联邦之间设备分配，业务负载均衡，数据迁移等。

### 2.2 大规模的分布式存储

- 大规模：当今各种FS扩展性约束在一个NAME node类似的东西，无法无限的扩展，到了一定程度之后就需要上层业务来切割；有些系统通过mount table的方式做了一些客户不感知的切割，但是mount table无法相互之间协同。
- 负载均衡：在大集群中，负载不均衡是正常的情况，
- HDFS联邦：HDFS有一个自带的联邦功能，但是并没有子集群的概念，需要用户手动的管理多个名字空间，而且所有的心跳还是要同步到所有的manager，扩展性也受限。

## 3 联邦架构

### 3.1 总体架构

![]({{ site.baseurl }}/assets/DH-HDFS-1.png)

- 不修改后台分布式存储系统，仅仅将多个分布式文件系统组成联邦
- 每个子集群相互独立，无需感知对方的存在；每份数据只会存在于一个子集群中

整体架构如图1所示：

- 在客户端和子集群的data manager之间插入一个新的软件层
- 这个软件层包括：
  - 多个客户端请求的路由（图中的R）
  - 一个stat store，负责维护全局的mount table（即目录/文件到子集群的映射关系）
  - 一个目录和文件的负载均衡器

### 3.2 客户端路由

- 客户端路由对客户呈现一个统一的名字空间，并且对外呈现的接口与元数据服务器完全一致，客户不感知这个路由层。
- 客户端可以连接到任意的router（通过LB或者其他的方式）
- router通过连接stat stor去确定那个子集群用来存放客户的数据
- 子集群响应存储节点的列表信息，客户端存放数据到存储节点

总体来看，架构比较简单：
- route将client的请求截获，多数只是做个简单的forward到对应的子集群
- 但是有几个复杂的操作需要处理下：比如rename，deletes，folder listing和写操作
  - route对所有删除挂载点的返回失败
  - rename或者删除目录，如果只涉及一个子集群，则直接转发过去；如果涉及多个子集群，就需要stat stor来参与了；并通过后台负载均衡来参与干活
  - 目录listing 需要联系父目录来找到所有子集群中的信息
  - 目录和文件的写操作可能会有短暂的失败（比如负载均衡的时候），主要为了保证数据的一致性
- 为了避免多次联系stat stor，每个route都cache这些映射信息（目录和文件到子集群的映射），此时需要确保映射信息的一致

### 3.3 stat store

主要存放一些映射状态，etcd存这些东西应该比较适合：

- 全局的mount table
  - 即目录和文件到子集群的映射信息，比如/tmp映射到子集群3，则为/3/tmp
  - 只有系统管理员和负载均衡器可以修改这个信息
  - 作者使用zookeeper来存放这个信息
- 所有router的状态
- 访问负载信息，子集群的可用状态，可用空间等
- 负载均衡操作的状态

### 3.4 负载均衡器

负载不均衡的几种情况：

- 负载放置算法在很多负载下不靠谱
- 负载很多时候会有一些波动
- 数据可能会有各种不同

为了解决负载不均衡的问题，需要一个负载均衡器，主要做以下事情：在负载不均衡时，在多个子集群之间迁移数据，需要重点解决几个问题：

- 确保一致性：负载均衡器需要确保各个场景下的数据的一致性
  - 首先，启动之前，负载均衡器在stat store中申请一个WAL，操作完成之后更新日志表明操作完成；基于日志可以实现回滚；并且可以防止冲突
  - 进行负载均衡操作之前，获取写租约，并且记录待迁移的整个子树状态；租约只是用来解决内部的冲突，对客户端来看，仍然是可以访问的。
  - 负载均衡器开始跨子集群复制数据，复制完成之后，检查数据中间是否修改（检查的过程中，不允许有写入操作）（一小段不可用时间）
    - 如果没有更新，直接修改mount table，负载迁移成功，删除旧的数据
    - 如果有更新，继续复制修改过的数据，在检查中间是否有更新，重复三次如果依然没有成功，放弃迁移
- 完成之后，释放租约

### 3.5 考虑过的其他架构

- 简单的扩展viewFS去实现一个共享的mount table，但是这个实现无法让底层FS不感知，需要修改客户端代码。

## 4 实现联邦的技术细节

## 4.1 服务器到子集群的映射

主要要确保每个子集群的多样性，来确保数据持久度和高可用。核心目标：

- 确保子集群的添加和删除服务器不导致大量的数据重建
- 在一个集群中，提升网络的局部性，有利于性能
- 确保每个首席租户的资源多样性（跨机架，跨服务器），保证数据的高可用和高持久度
- 确保单个子集群内部的负载均衡

为了达成上述目标，总体思路如下：

- 定义每个子集群服务器的上线（比如4000）
- 使用DHT的方式hash rack name，将不同的rack放入不同的子集群（使用虚拟节点的方式来确保DHT负载更加均衡），这样确保网络的局部性
- 每个首席租户的数据会在多个rack之间放置，保证数据持久度和可用性，同时平衡负载

## 4.2 目录和文件到子集群的映射

![]({{ site.baseurl }}/assets/DH-HDFS-2.png)

主要确保负载的均衡。几个核心要点：

- 创建时分配策略：一般情况下，第一层目录分到不同的集群，后续的层次默认和父目录用同一个子集群；这样可能会造成负载不均衡，由后续的动作去解决。
- 重新负载均衡的策略：负载均衡器周期性的启动，检查各个子集群的负载情况（主要分析元数据的负载情况），管理员也可以手动启动负载均衡。
- 重新负载均衡的优化：使用MILP模型来描述负载均衡问题，并使用标准的解决方法。细节暂时不分析了。就是一种比较高级的分析负载的算法。

4.3 作者考虑过的其他技术点

- 将服务器和文件的分布方式统一，而不是分两层：这样搞可能有点复杂
- 服务器分布到子集群的方式：本来考虑随机分配每个服务器，每个首席租户，每个租户组到不同的子集群，但是增加和删除节点时，会有大量的迁移工作
- 分配文件到子集群：考虑过直接使用hash文件名的方式去分布，但是存在几个问题:
  - 每个目录的数据就丢失了局部性
  - 子树重命名就是非常折腾

5. 学习到的一些东西

- 服务器到子集群的映射和启动：一旦集群启动之后（利旧环境），再讲算法调整为DHT会导致大量的数据迁移，为此设计了一个子集群控制器的服务。通过新的方式进行服务器到子集群的映射。

- 文件到子集群映射和用户：在使用DH-HDFS之前，用户会提交batch任务到其中一个子集群，为了平滑的让用户上船；我们的route就不纳管这些batch业务，让他继续用现有方案。

- 跨越多个子集群的大型数据集：即一个目录巨大无比，都跨越了多个子集群，这种情况下直接用hash来将文件分割到不同的子集群，并且不允许重命名之类的操作

- 负载均衡和管理员：当前LB主要有管理员触发，由于目录太复杂，在LB的时候会有一些聚合，比如讲 /tmp/logs, /logs放到一个目录中，简化操作。

- 生产环境的延迟：原生的延迟1ms左右，加了这一层之后延迟变为3ms