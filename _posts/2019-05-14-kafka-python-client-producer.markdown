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

## RecordAccumulator结构
`RecordAccumulator`类主要负责在客户端本地缓存发送的数据以及相应的内存管理。其中比较重要的函数又如下几个：

函数`append`负责向本地缓存中追加消息：
```    
def append(self, tp, timestamp_ms, key, value, headers, max_time_to_block_ms, estimated_size=0):
        assert isinstance(tp, TopicPartition), 'not TopicPartition'
        assert not self._closed, 'RecordAccumulator is closed'
        # We keep track of the number of appending thread to make sure we do
        # not miss batches in abortIncompleteBatches().
        self._appends_in_progress.increment()
        try:
            #使用dcl技术判断锁是否存在
            if tp not in self._tp_locks:
                with self._tp_locks[None]:
                    if tp not in self._tp_locks:
                        self._tp_locks[tp] = threading.Lock()
            #每个parition对应一把锁，这就是生产者是线程安全的原因
            with self._tp_locks[tp]:
                # check if we have an in-progress batch
                dq = self._batches[tp]
                if dq:
                    last = dq[-1]
                    # 向ProducerBatch中添加消息，如果添加失败则返回None，从而创建新的ProducerBatch
                    future = last.try_append(timestamp_ms, key, value, headers)
                    if future is not None:
                        batch_is_full = len(dq) > 1 or last.records.is_full()
                        return future, batch_is_full, False

            size = max(self.config['batch_size'], estimated_size)
            log.debug("Allocating a new %d byte message buffer for %s", size, tp) # trace
            buf = self._free.allocate(size, max_time_to_block_ms)
            with self._tp_locks[tp]:
                # Need to check if producer is closed again after grabbing the
                # dequeue lock.
                assert not self._closed, 'RecordAccumulator is closed'
                # 在新建ProducerBatch之前判断是不是由其他线程已经创建过，因为刚才有解锁操作
                if dq:
                    last = dq[-1]
                    future = last.try_append(timestamp_ms, key, value, headers)
                    if future is not None:
                        # Somebody else found us a batch, return the one we
                        # waited for! Hopefully this doesn't happen often...
                        self._free.deallocate(buf)
                        batch_is_full = len(dq) > 1 or last.records.is_full()
                        return future, batch_is_full, False

                records = MemoryRecordsBuilder(
                    self.config['message_version'],
                    self.config['compression_attrs'],
                    self.config['batch_size']
                )

                batch = ProducerBatch(tp, records, buf)
                future = batch.try_append(timestamp_ms, key, value, headers)
                if not future:
                    raise Exception()

                dq.append(batch)
                # 把新建的ProducerBatch放到表示未完成的_incomplete中，当收到produce response时，再从其中删除
                self._incomplete.add(batch)
                batch_is_full = len(dq) > 1 or batch.records.is_full()
                return future, batch_is_full, True
        finally:
            self._appends_in_progress.decrement()


```
