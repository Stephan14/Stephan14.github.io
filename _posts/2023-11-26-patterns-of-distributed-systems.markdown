---
layout:     post
title:      "分布式系统模式"
subtitle:   "读书笔记"
date:       2023-11-26 15:35:30
author:     "邹盛富"
header-img: "img/weary-8375125.jpg"
tags:
    - 分布式系统
---

## 一致性内核（Consistent Core）

### 问题
在集群内部中存在部分数据（节点信息、数据分片信息等）需要保证强一致性（线性一致性），通常的方案是使用quorum写入方式，但是其系统的吞会随着节点的增加降低。

### 解决方案

集群内部再组建一个3或5个节点的小集群，通过relase机制在集群内部进行决策。最简单的接口如下：
```
public interface ConsistentCore {
    CompletableFuture put(String key, String value);
    List<String> get(String keyPrefix);
    CompletableFuture registerLease(String name, long ttl);
    void refreshLease(String name);
    void watch(String name, Consumer<WatchEvent> watchCallback);
}
```
#### 元数据存储
可以基于raft来存储


#### 客户端交互
所有的操作都要在领导者上执行，这是至关重要的，因此，客户端程序库需要先找到领导者服务器。要做到这一点，有两种可能的方式:
1. 一致性内核的追随者服务器知道当前的领导者，因此，如果客户端连接追随者，它会返回 领导者的地址。客户端可以直接连接应答中给出的领导者。值得注意的是，客户端尝试连接时，服务器可能正处于领导者选举过程中。在这种情况下，服务器无法返回领导者地址，客户端需要等待片刻，再尝试连接另外的服务器。
2. 服务器实现转发机制，将所有的客户端请求转发给领导者。这样就允许客户端连接任意的服务器。同样，如果服务器处于领导者 选举过程中，客户端需要不断重试，直到领导者选举成功，一个合法的领导者获得确认。

## 追随者读取（Follower Reads）

### 问题
使用领导者和追随者模式时，如果有太多请求发给领导者，它可能会出现过载。此外，在多数据中心的情况下，客户端如果在远程的数据中心，向领导者发送的请求可能会有额外的延迟。

### 解决方案
当写请求要到领导者那去维持一致性，只读请求就会转到最近的追随者。当客户端大多都是只读的，这种做法就特别有用。但是，从追随者那里读取的客户端得到可能是旧值。对于该问题有2种解决方法：
1. 追随者发现自己和领导者之间滞后的数据量比较大时，会返回一个错误给client，client收到此错误的时候会自动连领导者进行读取
2. 每条数据写入时会生成对应的版本戳，这个版本戳可以是高水位标记（High-Water Mark）或是混合时钟（Hybrid Clock）。然后，如果客户端希望稍后读取该值的话，它就把这个版本戳当做读取请求的一部分。如果读取请求到了追随者那里，它就会检查其存储的值，看它是等于或晚于请求的版本戳。如果不是，它就会等待，直到有了最新的版本，再返回该值。通过这种做法，这个客户端总会读取与它写入一直的值——这种做法通常称为“读自己写”的一致性。

## Lamport 时钟（Lamport Clock）

### 问题
当值要在多个服务器上进行存储时，需要有一种方式知道一个值要在另外一个值之前存储。在这种情况下，不能使用系统时间戳，因为时钟不是单调的，两个服务器的时钟时间不应该进行比较。

### 解决方案

```
class LamportClock…

public int tick(int requestTime) {
    latestTime = Integer.max(latestTime, requestTime);
    latestTime++;
    return latestTime;
}

class Server…

public int write(String key, String value, int requestTimestamp) {
    //update own clock to reflect causality
    int writeAtTimestamp = clock.tick(requestTimestamp);
    mvccStore.put(new VersionedKey(key, writeAtTimestamp), value);
    return writeAtTimestamp;
}
```
用于写入值的时间戳会返回给客户端。通过更新自己的时间戳，客户端会跟踪最大的时间戳。它在发出进一步写入请求时会使用这个时间戳。

```
class Client…

LamportClock clock = new LamportClock(1);
public void write() {
    int server1WrittenAt = server1.write("name", "Alice", clock.getLatestTime());
    clock.updateTo(server1WrittenAt);

    int server2WrittenAt = server2.write("title", "Microservices", clock.getLatestTime());
    clock.updateTo(server2WrittenAt);

    assertTrue(server2WrittenAt > server1WrittenAt);
}

```

每个集群节点都维护着一个 Lamport 时钟的实例,服务器每当进行任何写操作时，服务器都应该让 Lamport 时钟前进。如此一来，服务器可以确保写操作的顺序是在这个请求之后，以及客户端发起请求时服务器端已经执行的任何其他动作之后。服务器会返回一个时间戳，用于将值写回给客户端。稍后，请求的客户端会使用这个时间戳向其它的服务器发起进一步的写操作。如此一来，请求的因果链就得到了维持。*但是只能保证部分有序，对于在不同server上的数据无法比较*

![例子](https://res.cloudinary.com/bytedance14/image/upload/v1703663232/tpigriulylzm7btnrzju.png)

#### 单一服务器/领导者更新值
只有领导者负责递增版本计数器，追随者使用相同的版本号。

![例子](https://res.cloudinary.com/bytedance14/image/upload/v1703663556/mumemrdk5ozfows1siau.png)

## 领导者和追随者（Leader and Followers）

### 问题

当数据在多个服务器上更新时，需要决定何时让客户端看到这些数据。只有写读的Quorum是不够的，因为一些失效的场景会导致客户端看到不一致的数据。单个的服务器并不知道Quorum上其它服务器的数据状态，只有数据是从多台服务器上读取时，才能解决不一致的问题。在某些情况下，这还不够。发送给客户端的数据需要有更强的保证。

### 解决方案

在集群里选出一台服务器成为领导者。领导者负责根据整个集群的行为作出决策，并将决策传给其它所有的服务器。


#### 领导者选举

只有“最新”的服务器才能成为合法的领导者。比如说，在典型的基于共识的系统中，“最新”由两件事定义：

    - 最新的世代时钟（Generation Clock）
    - 预写日志（Write-Ahead Log）的最新日志索引

如果所有的服务器都是最新的，领导者可以根据下面的标准来选：

    - 一些实现特定的标准，比如，哪个服务器评级为更好或有更高的 ID（比如，Zab）
    - 如果要保证注意每台服务器一次只投一票，就看哪台服务器先于其它服务器启动选举

##### 使用外部[线性化]的存储进行领导者选举

在一个数据集群内运行领导者选举，对小集群来说，效果很好。但对那些有数千个节点的大数据集群来说，使用外部存储会更容易一些，比如 Zookeeper 或 etcd （其内部使用了共识，提供了线性化保证）。实现领导者选举要有三个功能：

- compareAndSwap 指令，能够原子化地设置一个键值
- 心跳的实现，如果没有从选举节点收到心跳，将键值做过期处理，以便触发新的选举
- 通知机制，如果一个键值过期，就通知所有感兴趣的服务器

## 世代时钟（Generation Clock）

### 问题

集群余下的部分要能检测出有的请求是来自原有的领导者。原有的领导者本身也要能检测出，它是临时从集群中断开了，然后，采用必要的修正动作，交出领导权。

### 解决方案

维护一个单调递增的数字，表示服务器的世代。每次选出新的领导者，这个世代都应该递增。即便服务器重启，这个世代也应该是可用的，因此，它应该存储在预写日志（Write-Ahead Log）每一个条目里

采用领导者和追随者（Leader and Followers）模式，选举新的领导者选举时，服务器对这个世代的值进行递增。

```
class ReplicationModule…
private void startLeaderElection() {
      replicationState.setGeneration(replicationState.getGeneration() + 1);
      registerSelfVote();
      requestVoteFrom(followers);
}
```

服务器会把世代当做投票请求的一部分发给其它服务器。在这种方式下，经过了成功的领导者选举之后，所有的服务器都有了相同的世代。一旦选出新的领导者，追随者就会被告知新的世代。

```
follower
class ReplicationModule
private void becomeFollower(int leaderId, Long generation) {
      replicationState.setGeneration(generation);
      replicationState.setLeaderId(leaderId);
      transitionTo(ServerRole.FOLLOWING);
}
```
自此之后，领导者会在它发给追随者的每个请求中都包含这个世代信息。它也包含在发给追随者的每个心跳（HeartBeat）消息里，也包含在复制请求中。领导者也会把世代信息持久化到预写日志（Write-Ahead Log）的每一个条目里,如果追随者得到了一个来自已罢免领导的消息，追随者就可以告知其世代过低。当领导者得到了一个失败的应答，它就会变成追随者，期待与新的领导者建立通信。

## 混合时钟

### 问题

采用有版本的值（Versioned Value）时，如果用 Lamport 时钟当做版本，存储版本时，客户端并不知道实际的日期-时间。对于客户端而言，有时需要能够访问到像 01-01-2020 这样采用日期-时间的版本，而不仅仅是像 1、2、3 这样的整数。

### 解决方案

混合时间戳里维护了最新的时间，这个时间戳使用系统时间和整数计数器共同构建

```
class HybridTimestamp…

public class HybridTimestamp implements Comparable<HybridTimestamp> {
    private final long wallClockTime;
    private final int ticks;

    public HybridTimestamp(long systemTime, int ticks) {
        this.wallClockTime = systemTime;
        this.ticks = ticks;
    }

    public static HybridTimestamp fromSystemTime(long systemTime) {
        return new HybridTimestamp(systemTime, -1); //initializing with -1 so that addTicks resets it to 0
    }

    public HybridTimestamp max(HybridTimestamp other) {
        if (this.getWallClockTime() == other.getWallClockTime()) {
            return this.getTicks() > other.getTicks()? this:other;
        }
        return this.getWallClockTime() > other.getWallClockTime()?this:other;
    }

    public long getWallClockTime() {
        return wallClockTime;
    }

    public HybridTimestamp addTicks(int ticks) {
        return new HybridTimestamp(wallClockTime, this.ticks + ticks);
    }

    public int getTicks() {
        return ticks;
    }

    @Override
    public int compareTo(HybridTimestamp other) {
        if (this.wallClockTime == other.wallClockTime) {
            return Integer.compare(this.ticks, other.ticks);
        }
        return Long.compare(this.wallClockTime, other.wallClockTime);
    }

}

public synchronized HybridTimestamp now() {
     long currentTimeMillis = systemClock.currentTimeMillis();
     if (latestTime.getWallClockTime() >= currentTimeMillis) {
            //检查系统时间值是否在往回走，如果是，则递增另一个代表组件逻辑部分的数字，以反映时钟的进展。
          latestTime = latestTime.addTicks(1);
     } else {
         latestTime = new HybridTimestamp(currentTimeMillis, 0);
     }
     return latestTime;
}

```

服务器从客户端收到的每个写请求都会带有一个时间戳。接收的服务器会将自己的时间戳与请求的时间戳进行比较，将二者中较高的一个设置为自己的时间戳。

```
public HybridTimestamp write(String key, String value, HybridTimestamp requestTimestamp) {
      //update own clock to reflect causality
      HybridTimestamp writeAtTimestamp = clock.tick(requestTimestamp);
      mvccStore.put(key, writeAtTimestamp, value);
      return writeAtTimestamp;
}

class HybridClock…

public synchronized HybridTimestamp tick(HybridTimestamp requestTime) {
    long nowMillis = systemClock.currentTimeMillis();
    //set ticks to -1, so that, if this is the max, the next addTicks reset it to zero.
    HybridTimestamp now = HybridTimestamp.fromSystemTime(nowMillis);
    latestTime = max(now, requestTime, latestTime);
    latestTime = latestTime.addTicks(1);
    return latestTime;
}

private HybridTimestamp max(HybridTimestamp ...times) {
    HybridTimestamp maxTime = times[0];
    for (int i = 1; i < times.length; i++) {
        maxTime = maxTime.max(times[i]);
    }
    return maxTime;
}
```
用于写入值的时间戳会返回给客户端。请求的客户端会更新自己的时间戳，然后，在发起进一步的写入时会带上这个时间戳。
```
HybridClock clock = new HybridClock(new SystemClock());

public void write() {
    HybridTimestamp server1WrittenAt = server1.write("key1", "value1", clock.now());
    clock.tick(server1WrittenAt);

    HybridTimestamp server2WrittenAt = server2.write("key2", "value2", clock.now());

    assertTrue(server2WrittenAt.compareTo(server1WrittenAt) > 0);
}
```

## 幂等接收者（Idempotent Receiver）

### 问题
如果服务器已经处理了请求，然后奔溃了，之后，客户端重试时，服务器端会收到客户端的重复请求。

### 解决方案
1. 每个客户端都会收到一个唯一的ID，同时client会保存一个计数器，为每个请求也会分配一个ID
2. 服务器会为每个client创建一个session，保存对应client的请求信息
3. 对于过期的请求, 1)保存处理成功的最大的请求ID，该ID也会转发给其他的服务器，小于该ID的请求都丢掉 2)如果client侧能保证收到上一个请求的response之后才能发送下一个request，则服务器一旦收到这个client的请求可以丢掉之前的所有请求

## Gossip 传播（Gossip Dissemination）

### 问题
在拥有多个节点的集群中，每个节点都要向集群中的所有其它节点传递其所拥有的元数据信息，无需依赖于共享存储。

### 解决方案
每个集群节点都会调度一个 job 用以定期将其拥有的元数据传输给其他节点,调用调度任务时，它会从元数据集合的服务器列表中随机选取一小群节点。对于接收 Gossip 消息的集群节, 会做以下三件事：
- 传入消息中的值，且不再该节点状态集合中的消息，会merge到本地内存中
- 该节点拥有，但不再传入的 Gossip 消息中的消息，会返回给请求发送方，其会merge到本地内存中
- 节点拥有传入消息的值，这时会选择版本更高的值


## 心跳（HeartBeat）

### 问题

### 解决方案
一个服务器周期性地发送请求给所有其它的服务器，以此表明它依然活跃。选择的请求间隔应该大于服务器间的网络往返的时间。所有的服务器在检查心跳时，都要等待一个超时间隔，超时间隔应该是多个请求间隔。通常来说，超时间隔 > 请求间隔 > 服务器间的网络往返时间

#### 小集群
在所有的共识实现中，心跳是从领导者服务器发给所有追随者服务器的。每次收到心跳，都要记录心跳到达的时间戳。如果固定的时间窗口内没有收到心跳，就可以认为领导者崩溃了，需要选出一个新的服务器成为领导者

#### 大集群
给每个进程分配一个怀疑计数，在限定的时间内，如果没有收到该进程的 Gossip 消息，则怀疑计数递增。它可以根据过去的统计信息计算出来，只有在这个怀疑计数到达配置的上限时，才将其标记为失效。

## 高水位标记（High-Water Mark）

### 问题
如果领导者失效了，集群会选出一个新的领导者，客户端在大部分情况下还是能像从前一样继续在集群中工作。但是，还有几件事可能会有问题：
- 在向任意的追随者发送日志条目之前，领导者失效了
- 一部分追随者发送日志条目之后，领导者失效了，日志条目没有发送给大部分的追随者

在这些错误的场景下，一部分追随者的日志中可能会缺失一些条目，一部分追随者则拥有比其它部分多的日志条目。因此，对于每个追随者来说，有一点变得很重要，了解日志中哪个部分是安全的，对客户端是可用的。

### 解决方案
高水位标记就是一个日志文件中的索引，记录了在追随者的 Quorum 中都成功复制的最后一个日志条目。在复制期间，领导者也会把高水位标记传给追随者。对于集群中的所有服务器而言，只有反映的更新小于高水位标记的数据才能传输给客户端。

1. 对于每个日志条目而言，领导者将其追加到本地的预写日志中，然后，发送给所有的追随者。追随者会处理复制请求，将日志条目追加到本地日志中。在成功地追加日志条目之后，它们会把最新的日志条目索引回给领导者。应答中还包括服务器当前的时代时钟（Generation Clock）。
2. 领导者通过查看所有追随者的日志索引和领导者自身的日志，高水位标记是可以计算出来的，选取大多数服务器中可用的索引即可。
3. 领导者会将高水位标记传播给追随者，可能是当做常规心跳的一部分，也可能一个单独的请求。追随者随后据此设置自己的高水位标记。客户端只能读取到高水位标记前的日志条目。超出高水位标记的对客户端是不可见的。


#### 日志截断

暂停之后，重新启动或是重新加入集群，服务器都会先去寻找新的领导者。然后，它会显式地查询当前的高水位标记，将日志截断至高水位标记，然后，从领导者那里获取超过高水位标记的所有条目。类似 RAFT 之类的复制算法有一些方式找出冲突项，比如，查看自己日志里的日志条目，对比请求里的日志条目。如果日志条目拥有相同的索引，但时代时钟（Generation Clock）更低的话，就删除这些条目




## 参考资料
https://github.com/dreamhead/patterns-of-distributed-systems/tree/master
