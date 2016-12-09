---
title: Paxos/Mutil-paxos 算法
date: 2016-04-27 20:48:54
tags: [Distributed,Algorithm]
categories: Algorithm
---


在前几篇博客中，讨论了我在伯克利最后一学期修的分布式系统编程课程中的内容。

<!--more --> 

那时我参与的项目太酷了- 我们组一步步完成了从quorum KVS到分布式锁管理，又实现了Multi-paxos算法作为我们最终提交的项目。这篇博客，我会从普通的Paxos算法说起。

简单来说，Paxos是将一组节点在单个值上达成一致的协议(严格来说，是指一个协议簇，这是 [维基百科](https://en.wikipedia.org/wiki/Paxos_\(computer_science) 上的说法)

### Paxos算法为什么重要？

如果有个固定的的master节点，master节点就可以自己决策并提案值，让其它节点跟着使用这个值就行，那Paxos算法就没有任何用武之地了。倘若决策者的选举失败（有时就会这样），有两个节点提案值呢？

Paxos算法能保证在所有提案的值中选出一个唯一的值。当你想在分布式系统中的一系列值中做决策时，或者说，在这一系列的值中决策出一致的顺序，Paxos算法就派上用场了。

像[Ayende Rahien](https://ayende.com/blog/4496/paxos-enlightment)描述的那样，如果这些值是指一些事件，那在一个集群中所有的机器都能保持相同的状态或者状态的子集。

### Paxos 算法

Paxos算法分两个阶段进行，准备阶段(Prepare phase)与批准阶段(Accept phase)。

声明：下面章节的一些用词部分来源于Leslie Lamport的论文： [Paxo Made Simple](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-%20simple.pdf).


#### 准备阶段

在准备阶段，提案者(proposor)选出一个提案编号 `n` (此编号比提案者之前发送出的任何一个 编号都要大，也与任何其他提案者的编号 不同，将在下面详细讨论)，然后发送一个 `PREPARE` 请求给批准者的多数派。

对批准者(acceptor)来说，如果收到的 `PREPARE` 请求中的提案编号 `n` 比它之前收到的任何提案编号都要大，就会响应一个“承诺"  (promise)，指出之后不会接收任何一个比这个提案编号要小的编号,以及已经收到过的最大提案编号(如果有的话).


开始听起来有点困惑了？

在真正理解前我也读了好几遍，再看看下面的伪代码也许会有所帮助。

```C++

# Proposer
for acceptor in acceptors
  send :prepare_req, next_n()

# Acceptor
if (req.n > highest_proposed_n)
  highest_proposed_n = req.n
  reply :prepare_resp, {
    :n => highest_acc.n,
    :value => highest_acc.value
  }
```

#### 批准阶段

当提案者收到批准者中多数派对 `PREPARE` 请求的响应后，给批准者发送一个含有值 `v` 的 `ACCEPT` 信号，`v` 是收到的响应中最高提案编号中的值，当响应中没有任何批准的值时，`v` 可以是任意值。

当批准者收到 `ACCEPT` 请求时，总会接收这个请求，除非它已经在准备阶段承诺过不再接收任何请求。

```C++

# Proposer
for acceptor in acceptors
  send :accept_req, responses.argmax { |i| i.n }.value }

# Acceptor
if (req.n >= highest_proposed_n)
  highest_acc = {:n => req.n, :value => req.value}
  reply :accept_resp

```


### Mutil-Paxos 算法

上面所讲的是 Paxos 算法。那 Multi-paxos 算法是如何工作的呢？

前面提到，Paxos 算法最主要的应用在于，能让许多参与者在一系列列数值中做出决策。当Paxos算法结束运行一轮后，会生成一个提案值 `v` ，要决策出一系列的提案值来，最简单的方法就是多次运行 Paxos 算法，对不对？

实际上这种情况可以对 Paxos 算法进行优化： 设定一个固定的决策者，可以用来跳过准备阶段。假定这种决策关系能够保持"粘性"，就没必要继续发出提案编号---第一个被发送的提案号始终不会被覆盖，因为这时只有一个决策者。

因此，我们只需要运行一次准备阶段就够了。在 Paxos 算法接下来的几轮中，我们只需要发送 `ACCEPT` 消息，附带一个在原始 `PREPARE` 请求中用过的提案编号 `n` ,和一个额外的参数来表示我们正在运行的轮数序列号。最坏的情况不过是决策关系不是固定的(或者说不是唯一的),这时我们也不必担心，因为算法会自动降级到普通的Paxos算法中(每一轮过程中都有准备阶段和批准阶段）。好酷！

### 其它考虑

#### 永久存储

系统必须保证一些数据能永久存储，以防能从失效中恢复。
尤其是，批准者需要记住它们承诺过要响应哪个 `PREPARE` 请求以及它们响应过的 `ACCEPT` 请求，这样做是为了决策出要承诺哪个提案并能传递一些必要的信息反馈给此提案。

#### 唯一的提案编号 `n`

为了能让算法正常工作，每个提案者必须递增地增加提案编号 `n`，`n` 必须不同于其它任何提案者的编号.

为了满足这个条件，我们为每个提案者分配一个互斥的数值集合，并让它们只能从自己的集合中选取编号（比如说，给每个节点分配一个单独的素数，它们可以选择素数的不同倍数作为一个集合）

或者，我们也可以设定参与者成员的静态属性，为每个节点分配一个从0到 `k` 之间的唯一数值`i`，`k` 是参与者的总数，`n=i+(k*轮数)`.


### 结论

长话短说: Paxos 算法的目标是能让集群中的机器在一个提案值上最终达到统一（或者更有用的情况是，能为一系列数值决策出一致的顺序）。Multi-paxos 算法是对连续许多轮执行 Paxos 算法的一种优化，如果你设定了一个决策者，就可以跳过其中的一个阶段。

希望这篇博客能对你有用。我不是这个领域的专家，所以希望能听到更多的评论与修正。


### Acknowledgement

本文遵守Attribution-NonCommercial-NoDerivatives 4.0 International License (CC BY-NC-ND 4.0)


译文，仅为学习使用，未经博主同意，请勿转载。

原英文地址：http://amberonrails.com/paxosmulti-paxos-algorithm/

本文地址：http://www.chongh.wiki/blog/2016/04/27/paxos-mutilpaxos/

作者 [ 新浪微博: [@diting0x](http://weibo.com/2767520802/profile?rightmod=1&wvr=6&mod=personinfo) && [@睡眼惺忪的小叶先生](http://weibo.com/u/2765244861?topnav=1&wvr=6&topsug=1&is_all=1) ]
