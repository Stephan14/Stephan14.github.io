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

每个集群节点都维护着一个 Lamport 时钟的实例,服务器每当进行任何写操作时，服务器都应该让 Lamport 时钟前进。如此一来，服务器可以确保写操作的顺序是在这个请求之后，以及客户端发起请求时服务器端已经执行的任何其他动作之后。服务器会返回一个时间戳，用于将值写回给客户端。稍后，请求的客户端会使用这个时间戳向其它的服务器发起进一步的写操作。如此一来，请求的因果链就得到了维持。*但是只能保证部分有序，对于在不同server上的数据无法比较*
![](https://github.com/dreamhead/patterns-of-distributed-systems/blob/master/image/lamport-clock-request-sequence.png)

#### 单一服务器/领导者更新值
只有领导者负责递增版本计数器，追随者使用相同的版本号。
![](https://github.com/dreamhead/patterns-of-distributed-systems/blob/master/image/single-servergroup-kvstore.png)

##  

### 问题

### 解决方案



## 幂等接收者（Idempotent Receiver）

### 问题
如果服务器已经处理了请求，然后奔溃了，之后，客户端重试时，服务器端会收到客户端的重复请求。

### 解决方案
1. 每个客户端都会收到一个唯一的ID，同时client会保存一个计数器，为每个请求也会分配一个ID
2. 服务器会为每个client创建一个session，保存对应client的请求信息
3. 对于过期的请求, 1)保存处理成功的最大的请求ID，该ID也会转发给其他的服务器，小于该ID的请求都丢掉 2)如果client侧能保证收到上一个请求的response之后才能发送下一个request，则服务器一旦收到这个client的请求可以丢掉之前的所有请求

## 参考资料
https://github.com/dreamhead/patterns-of-distributed-systems/tree/master