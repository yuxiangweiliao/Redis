# 集群（上）

这篇文档是对 Redis 集群的介绍，没有使用复杂难懂的东西来理解分布式系统的概念。本文提供了如何建立，测试和操作一个集群的相关指导，但没有涉及在 Redis 集群规范（参考本系列其他文章，译者注）中的诸多细节，只是从用户的视角来描述系统是如何运作的。 

注意，如果你打算来一次认真的 Redis 集群的部署，更正式的规范文档（关注本系列文章，译者注）强烈建议你好好读一读。 

Redis 集群当前处于 alpha 阶段，如果你发现任何问题，请联系 Redis 邮件列表，或者在 Redis 的 Github 仓库中开启一个问题（issue）。 

## Redis 集群（Redis Cluster） 

Redis 集群提供一种运行 Redis 的方式，数据被自动的分片到多个 Redis 节点。 

集群不支持处理多个键的命令，因为这需要在 Redis 节点间移动数据，使得 Redis 集群不能提供像 Redis 单点那样的性能，在高负载下会表现得不可预知。 

Redis 集群也提供在网络分割（partitions）期间的一定程度的可用性，这就是在现实中当一些节点失败或者不能通信时能继续进行运转的能力。 

所以，在实践中，你可以从 Redis 集群中得到什么呢？ 

- 在多个节点间自动拆分你的数据集的能力。
- 当部分节点正在经历失败或者不能与集群其他节点通信时继续运转的能力。

## Redis 集群的 TCP 端口（Redis Cluster TCP ports） 

每个 Redis 集群节点需要两个 TCP 连接打开。正常的 TCP 端口用来服务客户端，例如 6379，加 10000 的端口用作数据端口，在上面的例子中就是 16379。 

第二个大一些的端口用于集群总线（bus），也就是使用二进制协议的点到点通信通道。集群总线被节点用于错误检测，配置更新，故障转移授权等等。客户端不应该尝试连接集群总线端口，而应一直与正常的 Redis 命令端口通信，但是要确保在防火墙中打开了这两个端口，否则 Redis 集群的节点不能相互通信。 

命令端口和集群总线端口的偏移量一直固定为 10000。 

注意，为了让 Redis 集群工作正常，对每个节点： 

1. 用于与客户端通信的正常的客户端通信端口（通常为 6379）需要开放给所有需要连接集群的客户端以及其他集群节点（使用客户端端口来进行键迁移）。
2. 集群总线端口（客户端端口加 10000）必须从所有的其他集群节点可达。

如果你不打开这两个 TCP 端口，你的集群就不会像你期待的那样去工作。 

## Redis 集群的数据分片（Redis Cluster data sharding） 

Redis 集群没有使用一致性哈希，而是另外一种不同的分片形式，每个键概念上是被我们称为哈希槽（hash slot）的东西的一部分。 

Redis 集群有 16384 个哈希槽，我们只是使用键的 CRC16 编码对 16384 取模来计算一个指定键所属的哈希槽。 

每一个 Redis 集群中的节点都承担一个哈希槽的子集，例如，你可能有一个 3 个节点的集群，其中： 

- 节点 A 包含从 0 到 5500 的哈希槽。
- 节点 B 包含从 5501 到 11000 的哈希槽。
- 节点 C 包含从 11001 到 16384 的哈希槽。

这可以让在集群中添加和移除节点非常容易。例如，如果我想添加一个新节点 D，我需要从节点 A，B，C 移动一些哈希槽到节点 D。同样地，如果我想从集群中移除节点 A，我只需要移动 A 的哈希槽到 B 和 C。当节点 A 变成空的以后，我就可以从集群中彻底删除它。 

因为从一个节点向另一个节点移动哈希槽并不需要停止操作，所以添加和移除节点，或者改变节点持有的哈希槽百分比，都不需要任何停机时间（downtime）。 

## Redis 集群的主从模型（Redis Cluster master-slave model） 

为了当部分节点失效时，或者无法与大多数节点通信时仍能保持可用，Redis 集群采用每个节点拥有 1（主服务自身）到 N 个副本（N-1 个附加的从服务器）的主从模型。 

在我们的例子中，集群拥有 A，B，C 三个节点，如果节点 B 失效集群将不能继续服务，因为我们不再有办法来服务在 5501-11000 范围内的哈希槽。 

但是，如果当我们创建集群后（或者稍后），我们为每一个主服务器添加一个从服务器，这样最终的集群就由主服务器 A，B，C 和从服务器 A1，B1，C1 组成，如果 B 节点失效系统仍能继续服务。 

B1 节点复制 B 节点，于是集群会选举 B1 节点作为新的主服务器，并继续正确的运转。 

## Redis 集群的一致性保证（Redis Cluster consistency guarantees） 

Redis 集群不保证强一致性。实践中，这意味着在特定的条件下，Redis 集群可能会丢掉一些被系统收到的写入请求命令。 

Redis 集群为什么会丢失写请求的第一个原因，是因为采用了异步复制。这意味着在写期间下面的事情发生了： 

- 你的客户端向主服务器 B 写入。
- 主服务器 B 回复 OK 给你的客户端。
- 主服务器 B 传播写入操作到其从服务器 B1，B2 和 B3。

你可以看到，B 在回复客户端之前没有等待从 B1，B2，B3 的确认，因为这是一个过高的延迟代价，所以如果你的客户端写入什么东西，B 确认了这个写操作，但是在发送写操作到其从服务器前崩溃了，其中一个从服务器被提升为主服务器，永久性的丢失了这个写操作。 

这非常类似于在大多数被配置为每秒刷新数据到磁盘的数据库发生的事情一样，这是一个可以根据以往不包括分布式系统的传统数据库系统的经验来推理的场景。同样的，你可以通过在回复客户端之前强制数据库刷新数据到磁盘来改进一致性，但这通常会极大的降低性能。 

基本上，有一个性能和一致性之间的权衡。 

注意：未来，Redis 集群在必要时可能或允许用户执行同步写操作。 

Redis 集群丢失写操作还有另一个场景，发生在网络分割时，客户端与至少包含一个主服务器的少数实例被孤立起来了。 

举个例子，我们的集群由 A，B，C，A1，B1，C1 共 6 个节点组成，3 个主服务器，3 个从服务器。还有一个客户端，我们称为 Z1。 

分割发生以后，有可能分割的一侧是 A，C，A1，B1，C1，分割的另一侧是 B 和 Z1。 

Z1 仍然可以写入到可接受写请求的 B。如果分割在很短的时间内恢复，集群会正常的继续。但是，如果分割持续了足够的时间，B1 在分割的大多数这一侧被提升为主服务器，Z1 发送给 B 的写请求会丢失。 

注意，Z1 发送给 B 的写操作数量有一个最大窗口：如果分割的大多数侧选举一个从服务器为主服务器后过了足够多的时间，少数侧的每一个主服务器节点将停止接受写请求。 

这个时间量是 Redis 集群一个非常重要的配置指令，称为节点超时（node timeout）。 

节点超时时间过后，主服务器节点被认为失效，可以用其一个副本来取代。同样地，节点超时时间过后，主服务器节点还不能感知其它主服务器节点的大多数，则进入错误状态，并停止接受写请求。 

## 创建和使用 Redis 集群（Creating and using a Redis Cluster） 

要创建一个集群，我们要做的第一件事情就是要有若干运行在集群模式下的 Redis 实例。这基本上意味着，集群不是使用正常的 Redis 实例创建的，而是需要配置一种特殊的模式 Redis 实例才会开启集群特定的特性和命令。 

下面是最小的 Redis 集群配置文件： 

```
port 7000  
cluster-enabled yes  
cluster-config-file nodes.conf  
cluster-node-timeout 5000  
appendonly yes  
```

正如你所看到的，简单的 cluster-enabled 指令开启了集群模式。每个实例包含一个保存这个节点配置的文件的路径，默认是 nodes.conf。这个文件不会被用户接触到，启动时由 Redis 集群实例生成，每次在需要时被更新。 

注意，可以正常运转的最小集群需要包含至少 3 个主服务器节点。在你的第一次尝试中，强烈建议开始一个 6 个节点的集群，3 个主服务器，3 个从服务器。 

要这么做，先进入一个新的目录，创建下面这些以端口号来命名的目录，我们后面会在每个目录中运行实例。 

像这样： 

```
mkdir cluster-test  
cd cluster-test  
mkdir 7000 7001 7002 7003 7004 7005  
```

在从 7000 到 7005 的每个目录内创建一个 redis.conf 文件。作为你的配置文件的模板，只使用上面的小例子，但是要确保根据目录名来使用正确的端口号来替换端口号 7000。 

现在，复制你从 Github 的不稳定分支的最新的源代码编译出来的 redis-server 可执行文件到 cluster-test 目录中，最后在你喜爱的终端应用程序中打开 6 个终端标签。 

像这样在每个标签中启动实例： 

```
cd 7000  
../redis-server ./redis.conf  
```

你可以从每个实例的日志中看到，因为 nodes.conf 文件不存在，每个节点都为自己赋予了一个新 ID。 

```
[82462] 26 Nov 11:56:55.329 * No cluster configuration found, I'm 97a3a64667477371c4479320d683e4c8db5858b1  
```

这个 ID 会一直被这个实例使用，这样实例就有一个在集群上下文中唯一的名字。每个节点使用这个 ID 来记录每个其它节点，而不是靠 IP 和端口。IP 地址和端口可能会变化，但是唯一的节点标识符在节点的整个生命周期中都不会改变。我们称这个标识符为节点 ID（Node ID）。 

## 创建集群（Creating the cluster） 

现在，我们已经有了一些运行中的实例，我们需要创建我们的集群，写一些有意义的配置到节点中。 

这很容易完成，因为我们有称为 redis-trib 的 Redis 集群命令行工具来帮忙，这是一个 Ruby 程序，可以在实例上执行特殊的命令来创建一个新的集群，检查或重分片一个已存在的集群，等等。 

redis-trib 工具在 Redis 源代码分发版本的 src 目录中。要创建你的集群，简单输入： 

```
./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \  
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005  
```

这里使用的命令是 create，因为我们想创建一个新的集群。--replicas 1 选项意思是我们希望每个创建的主服务器有一个从服务器。其他参数是我想用来创建新集群的实例地址列表。 

显然，我们要求的唯一布局就是创建一个拥有 3 个主服务器和 3 个从服务器的集群。 

Redis-trib 会建议你一个配置。输入 yes 接受。集群会被配置和连接在一起，也就是说，实例会被引导为互相之间对话。最后，如果一切顺利你会看到一个类似这样的消息： 

```
[OK] All 16384 slots covered  
```

这表示，16384 个槽中的每一个至少有一个主服务器在处理。 

## 与集群共舞（Playing with the cluste） 

在当前阶段，Redis 集群的一个问题是缺少客户端库的实现。 

据我所知有以下实现： 

- redis-rb-cluster 是我（@antirez）写的 Ruby 实现，作为其他语言的参考。这个是对原先的
- redis-rb 进行了简单的封装，实现了与集群高效对话的最小语义。
- redis-py-cluster 看起来就是 redis-rb-cluster 的 Python 版本。最新没有更新（最后一次提交是 6 个月之前）但是这是一个起点。
- 流行的 Predis 有对 Redis 集群的支持，支持最近有更新，并处于活跃开发状态。
- 最多使用的 Java 客户端 Jedis 最近增加了对 Redis 集群的支持，请查看项目 README 中的 Jedis 集群部分。
- StackExchange.Redis 提供对 C#的支持（应该与大多数.NET 语言工作正常：VB，F#等）。
- Github 上 Redis 仓库的不稳定分支上的 redis-cli 工具实现了一个基本的集群支持，使用-c 启动时切换。

测试 Redis 集群的简单办法就是尝试上面这些客户端，或者只是使用 redis-cli 命令行工具。下面的交互例子使用的是后者： 

```
$ redis-cli -c -p 7000  
redis 127.0.0.1:7000> set foo bar  
-> Redirected to slot [12182] located at 127.0.0.1:7002  
OK  
redis 127.0.0.1:7002> set hello world  
-> Redirected to slot [866] located at 127.0.0.1:7000  
OK  
redis 127.0.0.1:7000> get foo  
-> Redirected to slot [12182] located at 127.0.0.1:7002  
"bar"  
redis 127.0.0.1:7000> get hello  
-> Redirected to slot [866] located at 127.0.0.1:7000  
"world"  
```

redis-cli 的集群支持非常基本，所以总是依赖 Redis 集群节点重定向客户端到正确的节点。一个真正的客户端可以做得更好，缓存哈希槽和节点地址之间的映射，直接使用到正确节点的正确连接。映射只在集群的配置发生某些变化时才重新刷新，例如，故障转移以后，或者系统管理员通过添加或移除节点改变了集群的布局以后。