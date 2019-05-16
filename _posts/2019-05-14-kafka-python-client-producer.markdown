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
函数`ready`用来判断哪一个broker符合要求可以发送request：
```
def ready(self, cluster):
    ready_nodes = set()
    next_ready_check = 9999999.99
    unknown_leaders_exist = False
    now = time.time()

    # 判断是不是有线程在等待分配内存
    exhausted = bool(self._free.queued() > 0)
    # several threads are accessing self._batches -- to simplify
    # concurrent access, we iterate over a snapshot of partitions
    # and lock each partition separately as needed
    partitions = list(self._batches.keys())
    for tp in partitions:
        # 获取parition的leader
        leader = cluster.leader_for_partition(tp)
        if leader is None or leader == -1:
            unknown_leaders_exist = True
            continue
        elif leader in ready_nodes:
            continue
        # 判断一个parition是不是要求有序发送
        elif tp in self.muted:
            continue

        with self._tp_locks[tp]:
            dq = self._batches[tp]
            if not dq:
                continue
            batch = dq[0]
            retry_backoff = self.config['retry_backoff_ms'] / 1000.0
            linger = self.config['linger_ms'] / 1000.0
            # 判断是不是处于等待重试状态中
            backing_off = bool(batch.attempts > 0 and
                               batch.last_attempt + retry_backoff > now)
            waited_time = now - batch.last_attempt
            time_to_wait = retry_backoff if backing_off else linger
            time_left = max(time_to_wait - waited_time, 0)
            full = bool(len(dq) > 1 or batch.records.is_full())
            expired = bool(waited_time >= time_to_wait)
            # batch已经满了、有线程等待申请内存、实际等待的时间已经达到要求、已经退出了、同步写的情况下，认为可以向改节点发送request
            sendable = (full or expired or exhausted or self._closed or
                        self._flush_in_progress())

            if sendable and not backing_off:
                ready_nodes.add(leader)
            else:
                # Note that this results in a conservative estimate since
                # an un-sendable partition may have a leader that will
                # later be found to have sendable data. However, this is
                # good enough since we'll just wake up and then sleep again
                # for the remaining time.
                next_ready_check = min(time_left, next_ready_check)

    return ready_nodes, next_ready_check, unknown_leaders_exist
```
函数`drain`由sender线程调用获取已经缓存的消息数据：
```
def drain(self, cluster, nodes, max_size):
    if not nodes:
        return {}

    now = time.time()
    batches = {}
    for node_id in nodes:
        size = 0
        # 获取该节点的所有的parition
        partitions = list(cluster.partitions_for_broker(node_id))
        ready = []
        # to make starvation less likely this loop doesn't start at 0
        self._drain_index %= len(partitions)
        start = self._drain_index
        while True:
            tp = partitions[self._drain_index]
            if tp in self._batches and tp not in self.muted:
                with self._tp_locks[tp]:
                    dq = self._batches[tp]
                    if dq:
                        first = dq[0]
                        backoff = (
                            bool(first.attempts > 0) and
                            bool(first.last_attempt +
                                 self.config['retry_backoff_ms'] / 1000.0
                                 > now)
                        )
                        # Only drain the batch if it is not during backoff
                        if not backoff:
                            # 如果所有的发送的数据超过max_request_size的大小并且已经从batch中取到数据，则结束数据的获取；如果第一个batch中的数据就达到了max_request_size的大小限制，就只发送这一个batch的数据
                            if (size + first.records.size_in_bytes() > max_size
                                and len(ready) > 0):
                                # there is a rare case that a single batch
                                # size is larger than the request size due
                                # to compression; in this case we will
                                # still eventually send this batch in a
                                # single request
                                break
                            else:
                                batch = dq.popleft()
                                batch.records.close()
                                size += batch.records.size_in_bytes()
                                ready.append(batch)
                                batch.drained = now

            self._drain_index += 1
            self._drain_index %= len(partitions)
            # 最多是从所有的parition中各获取一个batch的数据发送
            if start == self._drain_index:
                break

        batches[node_id] = ready
    return batches
```
函数`reenqueue`由`_complete_batch`函数在重试的时候调用，重试是按照`ProducerBatch`维度来重试：
```
def reenqueue(self, batch):
    """Re-enqueue the given record batch in the accumulator to retry."""
    now = time.time()
    batch.attempts += 1
    batch.last_attempt = now
    batch.last_append = now
    batch.set_retry()
    assert batch.topic_partition in self._tp_locks, 'TopicPartition not in locks dict'
    assert batch.topic_partition in self._batches, 'TopicPartition not in batches'
    dq = self._batches[batch.topic_partition]
    with self._tp_locks[batch.topic_partition]:
        dq.appendleft(batch)
```
