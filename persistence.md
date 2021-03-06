# 持久化

本文提供对 Redis 持久化(persistence)的技术性描述，适合所有的 Redis 用户来阅读。想获得对 Redis 持久化和持久性保证有更全面的了解，也可以读一下作者的博客文章(地址为 http://antirez.com/post/redis-persistence-demystified.html，译者注)。 

## Redis 持久化(Persistence) 

Redis 提供了不同持久化范围的选项： 

- RDB 持久化以指定的时间间隔执行数据集的即时点(point-in-time)快照。
- AOF 持久化在服务端记录每次收到的写操作，在服务器启动时会重放，以重建原始数据集。命令使用和 Redis 协议一样的格式以追加的方式来记录。当文件太大时 Redis 会在后台重写日志。
- 如果你愿意，你可以完全禁止持久化，如果你只是希望你的数据在服务器运行期间才存在的话。
- 可以在同一个实例上同时支持 AOF 和 RDB。注意，在这种情况下，当 Redis 重启时，AOF 文件会被用于重建原始数据集，因为它被保证是最完整的数据。

理解 RDB 和 AOF 持久化之间的各自优劣 (trade-offs) 是一件非常重要的事情。让我们先从 RDB 开始： 

## RDB 优点(RDB advantages) 

- RDB 是一种表示某个即时点的 Redis 数据的紧凑文件。RDB 文件适合用于备份。例如，你可能想要每小时归档最近 24 小时的 RDB 文件，每天保存近 30 天的 RDB 快照。这允许你很容易的恢复不同版本的数据集以容灾。
- RDB 非常适合于灾难恢复，作为一个紧凑的单一文件，可以被传输到远程的数据中心，或者是 Amazon S3(可能得加密)。
- RDB 最大化了 Redis 的性能，因为 Redis 父进程持久化时唯一需要做的是启动(fork)一个子进程，由子进程完成所有剩余工作。父进程实例不需要执行像磁盘 IO 这样的操作。
- RDB 在重启保存了大数据集的实例时比 AOF 要快。

## RDB 缺点(RDB disadvantages) 

当你需要在 Redis 停止工作(例如停电)时最小化数据丢失，RDB 可能不太好。你可以配置不同的保存点(save point)来保存 RDB 文件(例如，至少 5 分钟和对数据集 100 次写之后，但是你可以有多个保存点)。然而，你通常每隔 5 分钟或更久创建一个 RDB 快照，所以一旦 Redis 因为任何原因没有正确关闭而停止工作，你就得做好最近几分钟数据丢失的准备了。 

RDB 需要经常调用 fork()子进程来持久化到磁盘。如果数据集很大的话，fork()比较耗时，结果就是，当数据集非常大并且 CPU 性能不够强大的话，Redis 会停止服务客户端几毫秒甚至一秒。AOF 也需要 fork()，但是你可以调整多久频率重写日志而不会有损(trade-off)持久性(durability)。 

## AOF 优点(AOF advantages) 

- 使用 AOF Redis 会更具有可持久性(durable)：你可以有很多不同的 fsync 策略：没有 fsync，每秒 fsync，每次请求时 fsync。使用默认的每秒 fsync 策略，写性能也仍然很不错(fsync 是由后台线程完成的，主线程继续努力地执行写请求)，即便你也就仅仅只损失一秒钟的写数据。
- AOF 日志是一个追加文件，所以不需要定位，在断电时也没有损坏问题。即使由于某种原因文件末尾是一个写到一半的命令(磁盘满或者其他原因),redis-check-aof 工具也可以很轻易的修复。
- 当 AOF 文件变得很大时，Redis 会自动在后台进行重写。重写是绝对安全的，因为 Redis 继续往旧的文件中追加，使用创建当前数据集所需的最小操作集合来创建一个全新的文件，一旦第二个文件创建完毕，Redis 就会切换这两个文件，并开始往新文件追加。
- AOF 文件里面包含一个接一个的操作，以易于理解和解析的格式存储。你也可以轻易的导出一个 AOF 文件。例如，即使你不小心错误地使用 FLUSHALL 命令清空一切，如果此时并没有执行重写，你仍然可以保存你的数据集，你只要停止服务器，删除最后一条命令，然后重启 Redis 就可以。


## AOF 缺点(AOF disadvantages) 

- 对同样的数据集，AOF 文件通常要大于等价的 RDB 文件。
- AOF 可能比 RDB 慢，这取决于准确的 fsync 策略。通常 fsync 设置为每秒一次的话性能仍然很高，如果关闭 fsync，即使在很高的负载下也和 RDB 一样的快。不过，即使在很大的写负载情况下，RDB 还是能提供能好的最大延迟保证。
- 在过去，我们经历了一些针对特殊命令(例如，像 BRPOPLPUSH 这样的阻塞命令)的罕见 bug，导致在数据加载时无法恢复到保存时的样子。这些 bug 很罕见，我们也在测试套件中进行了测试，自动随机创造复杂的数据集，然后加载它们以检查一切是否正常，但是，这类 bug 几乎不可能出现在 RDB 持久化中。为了说得更清楚一点：Redis AOF 是通过递增地更新一个已经存在的状态，像 MySQL 或者 MongoDB 一样，而 RDB 快照是一次又一次地从头开始创造一切，概念上更健壮。但是，1)要注意 Redis 每次重写 AOF 时都是以当前数据集中的真实数据从头开始，相对于一直追加的 AOF 文件(或者一次重写读取老的 AOF 文件而不是读内存中的数据)对 bug 的免疫力更强。2)我们还没有收到一份用户在真实世界中检测到崩溃的报告。

## 我们该选谁(what) 

通常来说，你应该同时使用这两种持久化方法，以达到和 PostgreSQL 提供的一样的数据安全程度。 

如果你很关注你的数据，但是仍然可以接受灾难时有几分钟的数据丢失，你可以只单独使用 RDB。 

有很多用户单独使用 AOF，但是我们并不鼓励这样，因为时常进行 RDB 快照非常方便于数据库备份，启动速度也较之快，还避免了 AOF 引擎的 bug。 

注意：基于这些原因，将来我们可能会统一 AOF 和 RDB 为一种单一的持久化模型(长远计划)。 

下面的部分将介绍两种持久化模型等多的细节。 

## 快照(Snapshotting) 

默认情况下，Redis 保存数据集快照到磁盘，名为 dump.rdb 的二进制文件。你可以设置让 Redis 在 N 秒内至少有 M 次数据集改动时保存数据集，或者你也可以手动调用 SAVE 或者 BGSAVE 命令。 

例如，这个配置会让 Redis 在每个 60 秒内至少有 1000 次键改动时自动转储数据集到磁盘： 

```
save 60 1000  
```

这种策略被称为快照。 

## 如何工作(How works) 

每当 Redis 需要转储数据集到磁盘时，会发生： 

- Redis 调用 fork()。于是我们有了父子两个进程。
- 子进程开始将数据集写入一个临时 RDB 文件。
- 当子进程完成了新 RDB 文件，替换掉旧文件。

这个方法可以让 Redis 获益于写时复制(copy-on-write)机制。 

## 只追加文件(Append-only file) 

快照并不是非常具有可持久性(durable)。如果你运行 Redis 的电脑停机了，电源线断了，或者你不小心 kill -9 掉你的实例，最近写入 Redis 的数据将会丢失。尽管这个对一些应用程序来说不是什么大事，但是也有一些需要完全可持久性(durability)的场景，在这些场景下可能就不合适了。 

只追加文件是一个替代方案，是 Redis 的完全可持久性策略。在 1.1 版本中就可用了。 

你可以在你的配置文件中开启 AOF： 

```
appendonly yes  
```

从现在开始，每次 Redis 收到修改数据集的命令，将会被追加到 AOF 中。当你重启 Redis 的时候，就会重放(re-play)AOF 文件来重建状态。 

## 日志重写(Log rewriting) 

你可以猜得到，写操作不断执行的时候 AOF 文件会越来越大。例如，如果你增加一个计数器 100 次，你的数据集里只会有一个键存储这最终值，但是却有 100 条记录在 AOF 中。其中 99 条记录在重建当前状态时是不需要的。 

于是 Redis 支持一个有趣的特性：在后台重建 AOF 而不影响服务客户端。每当你发送 BGREWRITEAOF 时，Redis 将会写入一个新的 AOF 文件，包含重建当前内存中数据集所需的最短命令序列。如果你使用的是 Redis 2.2 的 AOF，你需要不时的运行 BGREWRITEAOF 命令。Redis 2.4 可以自动触发日志重写(查看 Redis 2.4 中的示例配置文件以获得更多信息)。 

## AOF 持久性如何(How durable) 

你可以配置多久 Redis 会 fsync 数据到磁盘一次。有三个选项： 

- 每次一个新命令追加到 AOF 文件中时执行 fsync。非常非常慢，但是非常安全。
- 每秒执行 fsync。够快(2.4 版本中差不多和快照一样快)，但是当灾难来临时会丢失 1 秒的数据。
- 从不执行 fsync，直接将你的数据交到操作系统手里。更快，但是更不安全。

建议的(也是默认的)策略是每秒执行一次 fsync。既快，也相当安全。一直执行的策略在实践中非常慢(尽管在 Redis 2.0 中有所改进)，因为没法让 fsync 这个操作本身更快。 

## AOF 损坏了怎么办(corrupted) 

有可能在写 AOF 文件时服务器崩溃(crash)，文件损坏后 Redis 就无法装载了。如果这个发生的话，你可以使用下面的步骤来解决这个问题： 

- 创建 AOF 的一个拷贝用于备份。
- 使用 Redis 自带的 redis-check-aof 工具来修复原文件：
- $ redis-check-aof --fix
- 使用 diff -u 来检查两个文件有什么不同。用修复好的文件来重启服务器。

## 如何工作(How works) 

日志重写采用了和快照一样的写时复制机制。下面是过程： 

- Redis 调用 fork()。于是我们有了父子两个进程。
- 子进程开始向一个临时文件中写 AOF。
- 父进程在一个内存缓冲区中积累新的变更(同时将新的变更写入旧的 AOF 文件，所以即使重写失败我们也安全)。
- 当子进程完成重写文件，父进程收到一个信号，追加内存缓冲区到子进程创建的文件末尾。
- 搞定！现在 Redis 自动重命名旧文件为新的，然后开始追加新数据到新文件。

## 如何从 RDB 切换到 AOF(How switch) 

在 Redis 2.2 及以上版本中非常简单，也不需要重启。 

- 备份你最新的 dump.rdb 文件。
- 把备份文件放到一个安全的地方。
- 发送以下两个命令：
- redis-cli config set appendonly yes
- redis-cli config set save ""
- 确保你的数据库含有其包含的相同的键的数量。
- 确保写被正确的追加到 AOF 文件。

第一个 CONFIG 命令开启 AOF。Redis 会阻塞以生成初始转储文件，然后打开文件准备写，开始追加写操作。

第二个 CONFIG 命令用于关闭快照持久化。这一步是可选的，如果你想同时开启这两种持久化方法。 

重要：记得编辑你的 redis.conf 文件来开启 AOF，否则当你重启服务器时，你的配置修改将会丢失，服务器又会使用旧的配置。 

此处省略一万字。。。。。。原文此处介绍 2.0 老版本怎么操作。 

## AOF 和 RDB 的相互作用(Interactions) 

Redis 2.4 及以后的版本中，不允许在 RDB 快照操作运行过程中触发 AOF 重写，也不允许在 AOF 重写运行过程中运行 BGSAVE。这防止了两个 Redis 后台进程同时对磁盘进行繁重的 IO 操作。 

当在快照运行的过程中，用户使用 BGREWRITEAOF 显式请求日志重写操作的话，服务器会答复一个 OK 状态码，告诉用户这个操作已经被安排调度，等到快照完成时开始重写。 

Redis 在同时开启 AOF 和 RDB 的情况下重启，会使用 AOF 文件来重建原始数据集，因为通常 AOF 文件是保存数据最完整的。 

## 备份数据(Backing up) 

开始这一部分之前，请务必牢记：一定要备份你的数据库。磁盘损坏，云中实例丢失，等等：没有备份意味着数据丢失的巨大风险。 

Redis 对数据备份非常友好，因为你可以在数据库运行时拷贝 RDB 文件：RDB 文件一旦生成就不会被修改，文件生成到一个临时文件中，当新的快照完成后，将自动使用 rename(2) 原子性的修改文件名为目标文件。 

这意味着，在服务器运行时拷贝 RDB 文件是完全安全的。以下是我们的建议： 

- 创建一个定时任务(cron job)，每隔一个小时创建一个 RDB 快照到一个目录，每天的快照放在另外一个目录。
- 每次定时脚本运行时，务必使用 find 命令来删除旧的快照：例如，你可以保存最近 48 小时内的每小时快照，一到两个月的内的每天快照。注意命名快照时加上日期时间信息。
- 至少每天一次将你的 RDB 快照传输到你的数据中心之外，或者至少传输到运行你的 Redis 实例的物理机之外。

## 灾难恢复(Disaster recovery) 

在 Redis 中灾难恢复基本上就是指备份，以及将这些备份传输到外部的多个数据中心。这样即使一些灾难性的事件影响到运行 Redis 和生成快照的主数据中心，数据也是安全的。 

由于许多 Redis 用户都是启动阶段的屌丝，没有太多钱花，我们会介绍一些最有意思的灾难恢复技术，而不用太多的花销。 

- Amazon S3 和一些类似的服务是帮助你灾难恢复系统的一个好办法。只需要将你的每日或每小时的 RDB 快照以加密的方式传输到 S3。你可以使用 gpg -c 来加密你的数据(以对称加密模式)。确保将你的密码保存在不同的安全地方(例如给一份到你的组织中的最重要的人)。推荐使用多个存储服务来改进数据安全。
- 使用 SCP(SSH 的组成部分)来传输你的快照到远程服务器。这是一种相当简单和安全的方式：在远离你的位置搞一个小的 VPS，安装 ssh，生成一个无口令的 ssh 客户端 key，并将其添加到你的 VPS 上的 authorized_keys 文件中。你就可以自动的传输备份文件了。为了达到好的效果，最好是至少从不同的提供商那搞两个 VPS。

要知道这种系统如果没有正确的处理会很容易失败。至少一定要确保传输完成后验证文件的大小 (要匹配你拷贝的文件)，如果你使用 VPS 的话，可以使用 SHA1 摘要。 

你还需要一个某种独立的告警系统，在某些原因导致的传输备份过程不正常时告警。 