---
layout:     post
title:      "kafka python 客户端分析"
subtitle:   "生产者源码分析"
date:       2019-01-14 22:24:30
author:     "邹盛富"
header-img: "img/sunset-3325080_1920.jpg"
tags:
    - kafka
---

## 组成结构

在`KafkaProducer`类的构造函数中可以发现，`KafkaProducer`主要由一下几部分组成：
- `KafkaClient`:包含管理与kafka broker之间的连接，并发送相应的request
- `RecordAccumulator`:包含写入数据的批量管理
- `Sender`:是个线程，主要负责发送produce request以及获取response
