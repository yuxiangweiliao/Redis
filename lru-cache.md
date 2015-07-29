# 使用 Redis 作为 LRU 缓存

当 Redis 作为缓存使用时，当你添加新的数据时，有时候很方便使 Redis 自动回收老的数据。这种行为在开发者社区中众所周知，因为这是流行的 memcached 系统的默认行为。 

LRU 实际上是被唯一支持的数据移除方法。本文内容将包含 Redis 的 maxmemory 指令，用于限制内存使用到一个固定的容量，也包含深入探讨 Redis 使用的 LRU 算法，一个近似准确的 LRU。 

## maxmemory 配置指令(configuration directive) 

maxmemory 配置指令是用来配置 Redis 为数据集使用指定的内存容量大小。可以使用 redis.conf 文件来设置配置指令，或者之后在运行时使用 CONFIG SET 命令。 

例如，为了配置内存限制为 100MB，可以在 redis.conf 文件中使用以下指令 

```
maxmemory 100mb  
```

设置 maxmemory 为 0，表示没有内存限制。这是 64 位系统的默认行为，32 位的系统则使用 3G 大小作为隐式的内存限制。 

当指定的内存容量到达时，需要选择不同的行为，即策略。Redis 可以只为命令返回错误，这样将占用更多的内存，或者每次添加新数据时，回收掉一些旧的数据以避免内存限制。 

## 回收策略(Eviction policies) 

当 maxmemory 限制到达的时候，Redis 将采取的准确行为是由 maxmemory-policy 配置指令配置的。 

以下策略可用： 

- noeviction：当到达内存限制时返回错误。当客户端尝试执行命令时会导致更多内存占用(大多数写命令，除了 DEL 和一些例外)。
- allkeys-lru：回收最近最少使用(LRU)的键，为新数据腾出空间。
- volatile-lru：回收最近最少使用(LRU)的键，但是只回收有设置过期的键，为新数据腾出空间。
- allkeys-random：回收随机的键，为新数据腾出空间。
- volatile-random：回收随机的键，但是只回收有设置过期的键，为新数据腾出空间。
- volatile-ttl：回收有设置过期的键，尝试先回收离 TTL 最短时间的键，为新数据腾出空间。

当没有满足前提条件的话，volatile-lru，volatile-random 和 volatile-ttl 策略就表现得和 noeviction 一样了。 

选择正确的回收策略是很重要的，取决于你的应用程序的访问模式，但是，你可以在程序运行时重新配置策略，使用 INFO 输出来监控缓存命中和错过的次数，以调优你的设置。 

一般经验规则： 

- M如果你期待你的用户请求呈现幂律分布(power-law distribution)，也就是，你期待一部分子集元素被访问得远比其他元素多，可以使用 allkeys-lru 策略。在你不确定时这是一个好的选择。
- 如果你是循环周期的访问，所有的键被连续扫描，或者你期待请求正常分布(每个元素以相同的概率被访问)，可以使用 allkeys-random 策略。
- 如果你想能给 Redis 提供建议，通过使用你创建缓存对象的时候设置的 TTL 值，确定哪些对象应该被过期，你可以使用 volatile-ttl 策略。

当你想使用单个实例来实现缓存和持久化一些键，allkeys-lru 和 volatile-random 策略会很有用。但是，通常最好是运行两个 Redis 实例来解决这个问题。 

另外值得注意的是，为键设置过期时间需要消耗内存，所以使用像 allkeys-lru 这样的策略会更高效，因为在内存压力下没有必要为键的回收设置过期时间。 

## 回收过程 (Eviction process) 

理解回收的过程是这么运作的非常的重要： 

- 一个客户端运行一个新命令，添加了新数据。
- Redis 检查内存使用情况，如果大于 maxmemory 限制，根据策略来回收键。
- 一个新的命令被执行，如此等等。

我们通过检查，然后回收键以返回到限制以下，来连续不断的穿越内存限制的边界。 

如果一个命令导致大量的内存被占用 (像一个很大的集合交集保存到一个新的键)，一会功夫内存限制就会被这个明显的内存量所超越。 

## 近似的 LRU 算法(Approximated LRU algorithm) 

Redis 的 LRU 算法不是一个精确的实现。这意味着 Redis 不能选择最佳候选键来回收，也就是最久钱被访问的那些键。相反，会尝试运营一个近似的 LRU 算法，通过采样一小部分键，然后在采样键中回收最适合(拥有最久访问时间)的那个。 

然而，从 Redis3.0(当前还是 beta 版本)开始，算法被改进为持有回收候选键的一个池子。这改善了算法的性能，使得更接近于真实的 LRU 算法的行为 

Redis 的 LRU 算法有一点很重要，你可以调整算法的精度，通过改变每次回收时检查的采样数量。这个参数可以通过如下配置指令： 

```
maxmemory-samples 5  
```

Redis 没有使用真实的 LRU 实现的原因，是因为这会消耗更多的内存。然而，近似值对使用 Redis 的应用来说基本上也是等价的。下面的图形对比，为 Redis 使用的 LRU 近似值和真实 LRU 之间的比较。 

用于测试生成上面图像的 Redis 服务被填充了指定数量的键。键被从头访问到尾，所以第一个键是 LRU 算法的最佳候选回收键。然后，再新添加 50% 的键，强制一般的旧键被回收。 

你可以从图中看到三种不同的原点，形成三个不同的带。 

- 浅灰色带是被回收的对象
- 灰色带是没有被回收的对象
- 绿色带是被添加的对象

在理论的 LRU 实现中，我们期待看到的是，在旧键中第一半会过期。而 Redis 的 LRU 算法则只是概率性的过期这些旧键。 

你可以看到，同样采用 5 个采样，Redis 3.0 表现得比 Redis 2.8 要好，Redis 2.8 中最近被访问的对象之间的对象仍然被保留。在 Redis 3.0 中使用 10 为采样大小，近似值已经非常接近理论性能。 

注意，LRU 只是一个预言指定键在未来如何被访问的模式。另外，如果你的数据访问模式非常接近幂律，大多数的访问都将集中在一个集合中，LRU 近似算法将能处理得很好。 

在模拟实验的过程中，我们发现使用幂律访问模式，真实的 LRU 算法和 Redis 的近似算法之间的差异非常小，或者根本就没有。 

然而，你可以提高采样大小到 10，这会消耗额外的 CPU，来更加近似于真实的 LRU 算法，看看这会不会使你的缓存错失率有差异。 

使用 CONFIG SET maxmemory-samples <count>命令在生产环境上试验各种不同的采样大小值是很简单的。