---
layout:     post
title:      "The Tail At Scale"
subtitle:   "论文总结"
date:       2022-06-05 14:25:30
author:     "zoushengfu"
header-img: "img/rainbow-5324147.jpeg"
tags:
    - 分布式系统
---


## 为什么会存在抖动
响应时间的抖动会导致服务的各组件产生高的长尾延迟，而抖动的产生有很多原因，包括
- 共享资源:机器可能被争夺共享资源(如 CPU 核心、处理器缓存、内存带宽和网络带宽)的不同应用所共享，而在同一应用中，不同的请求可能争夺资源。
- 守护程序:后台守护程序可能平均只使用有限的资源，但在运行时可能会产生几毫秒的峰值抖动。
- 全局资源共享:在不同机器上运行的应用程序可能会争抢全局资源(如网络交换机和共享文件系统)。
- 维护活动:后台活动(如分布式文件系统中的数据重建，BigTable等存储系统中的定期日志压缩，以及垃圾收集语言中的定期垃圾收集)会导致周期性的延迟高峰。
- 排队:中间服务器和网络交换机中的多层队列放大了这种可能性。
- 功率限制:现代CPU被设计成可以暂时运行在其平均功率之上(英特尔睿频加速技术)，如果这种活动持续很长时间，之后又会通过节流(throttling)限制发热。
- 垃圾收集:固态存储设备提供了非常快的随机读取访问，但是需要定期对大量的数据块进行垃圾收集，即使是适度的写入活动，也会使读取延迟增加100倍。
- 能源管理:许多类型的设备的省电模式可以节省相当多的能量，但在从非活动模式转为活动模式时，会增加额外的延迟。


## 服务级别降低抖动方案

### 服务分级&&优先级队列
差异化服务类别可以用来优先调度用户正在等待的请求，而不是非交互式请求。保持低级队列较短，以便更高级别的策略更快生效。

### 减少队头阻塞
将需要长时间运行的请求打散成一连串小请求，使其可以与其它短时间任务交错执行；例如，谷歌的网络搜索系统使用这种时间分割，以防止大请求影响到大量其它计算成本较低的大量查询延迟。(队头阻塞还有协议和连接层面的问题，需要通过使用更新的协议来解决，比如h1->h2->h3的升级思路)


### 管理后台活动和同步中断
后台任务可能产生巨大的 CPU、磁盘或网络负载；例子是面向日志的存储系统的日志压缩和垃圾收集语言的垃圾收集器活动。可以结合限流功能，把重量级操作分解成成本较低的操作，并在整体负载较低的时候触发这些操作(比如半夜)，以减少后台活动对交互式请求延迟的影响。


## 请求级别降低抖动方案
### 对冲请求
抑制延迟变化的一个简单方法是向多个副本发出相同的请求(Go并发模式中的orchannel)，并使用首先响应的结果。一旦收到第一个结果，客户端就会取消剩余的未处理请求。不过直接这么实现会造成额外的多倍负载。所以需要考虑优化。

一个方法是推迟发送第二个请求，直到第一个请求到达到95分位数还没有返回。这种方法将额外的负载限制在5%左右，同时大大缩短了长尾时间。

### 捆绑式请求
不像对冲一样等待一段时间发送，而是同时发给多个副本，但告诉副本还有其它的服务也在执行这个请求，副本任务处理完之后，会异步的方式主动请求其它副本取消其正在处理的同一个请求，需要额外的网络网络请求

## 集群级别降低抖动方案

### 微分区
把服务器区分成很多小分区，比如一台大服务器分成20个partition，这样无论是流量调整或者故障恢复都可以快非常多。以细粒度来调整负载便可以尽量降低负载不均导致的延迟影响，需要第三个组件来管理这些分区。

### 选择性的复制
为你检测到的或预测到的会很热的分区增加复制因子。然后，负载均衡器可以帮助分散负载。谷歌的主要网络搜索系统采用了这种方法，在多个微分区中对流行和重要的文件进行额外的复制。(就是热点分区多准备些实例，感觉应该需要按具体业务去做一些预测，可能对读请求实现起来比较简单，对于写请求实现起来比较麻烦)

### 将慢机器置于考察期
当检测到一台慢速机器时，暂时将其排除在操作之外(circuit breaker)。由于缓慢往往是暂时的，监测何时使受影响的系统重新上线。继续向这些被排除的服务器发出影子请求，收集它们的延迟统计数据，以便在问题缓解时将它们重新纳入服务中。(这里就是简单的熔断的实现)


## 实现的思考

### 怎么实现统计PCT95和PCT99
- 先对1s中的请求耗时数据进行统计然后排序（可以使用快排），然后计算出95和99对应的数值，进一步优化思路可以参照[数据流的中位数](https://leetcode.cn/problems/find-median-from-data-stream/)，还可以参照Prometheus中的采用了一种非常巧妙的数据结构来计算gram，其次工业界中常用的算法是基于[T-Digest](https://blog.bcmeng.com/pdf/TDigest.pdf)实现的。

### 怎么接入实际项目中
可以和RPC框架或者线程池结合起来
### 如何实现对冲请求
```
type Sampler struct {
    Sampling(value int64)
    Calculate() int64
}

type Strategy struct {
    Callback(f func(e *Enpoint)Error) bool
    Cancel(e *Endpoint)
}

type HedgedStrategy struct {
    p95Sampler LagtencySampler
    candidates []*Endpoint
}

func (hs *HedgedStrategy)Callback(f func(e *Enpoint)Error) {
    p95 := hs.p95Sampler.Calculate()
    c := time.Ticker(p95)
    pending := make(chan struct{})
    idx := rand.Int(size)
    go func() {
        start := time.Now()
        _ := f(js.candidates[idx])
        end := time.Now()
        hs.p95Sampler(end - start)
        pending <- struct{}
    }

    select {
        case <- c:
        // 耗时超过p95
        if ok := hs.Callback(f); ok {
            hs.Cancel(idx)
        }
        case <- pending:
    }
    return true;
}
```

### 如何多种策略组合使用
```
    s := HedgedStrategy{
        p95Sampler: new(LagtencySampler)
        candidates: make([]*Endpoint, 10)
    }

    o := OtherStrategy{
        p95Sampler: new(LagtencySampler)
        candidates: make([]*Endpoint, 10)
    }

    fun := s.Callback(func(e* Endpoint) {
            err := e.rpcCall(req, response)
            if (err != nil) {

            }
        })

   o.Callback(fun)
```
