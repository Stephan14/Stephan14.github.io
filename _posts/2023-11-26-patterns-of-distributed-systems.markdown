---
layout:     post
title:      "分布式系统模式"
subtitle:   "读书笔记"
date:       2023-11-26 15:35:30
author:     "zoushengfu"
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


### 参考资料
https://github.com/dreamhead/patterns-of-distributed-systems/tree/master
