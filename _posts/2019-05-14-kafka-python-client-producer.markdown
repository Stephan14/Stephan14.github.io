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
            # 每个ProducerBatch不是立即发送，需要等待一段时间
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

## Sender结构
`Sender`是一个线程起主要功能是发送producer请求进行生产数据，主要函数`run`如下：
```
def run(self):
    """The main run loop for the sender thread."""
    log.debug("Starting Kafka producer I/O thread.")

    # main loop, runs until close is called
    # 当线程没有关闭的时候一直调用run_once构造request
    while self._running:
        try:
            self.run_once()
        except Exception:
            log.exception("Uncaught error in kafka producer I/O thread")

    log.debug("Beginning shutdown of Kafka producer I/O thread, sending"
              " remaining records.")

    # okay we stopped accepting requests but there may still be
    # requests in the accumulator or waiting for acknowledgment,
    # wait until these are completed.
    # 如果不是强制关闭线程，会一直等待没有发送的数据发送完毕
    while (not self._force_close
           and (self._accumulator.has_unsent()
                or self._client.in_flight_request_count() > 0)):
        try:
            self.run_once()
        except Exception:
            log.exception("Uncaught error in kafka producer I/O thread")
    # 如果是强制关闭，丢弃掉没有发送出去的数据
    if self._force_close:
        # We need to fail all the incomplete batches and wake up the
        # threads waiting on the futures.
        self._accumulator.abort_incomplete_batches()

    try:
        self._client.close()
    except Exception:
        log.exception("Failed to close network client")

    log.debug("Shutdown of Kafka producer I/O thread has completed.")
```
下面介绍一下比较重要的`run_once`函数：
```
def run_once(self):
    """Run a single iteration of sending."""
    while self._topics_to_add:
        self._client.add_topic(self._topics_to_add.pop())

    # get the list of partitions with data ready to send
    # 根据metadata数据判断当前那个节点是可以发送request、以及下次检查的时间和是否有不知道leader的parition
    result = self._accumulator.ready(self._metadata)
    ready_nodes, next_ready_check_delay, unknown_leaders_exist = result

    # if there are any partitions whose leaders are not known yet, force
    # metadata update
    # 如果某些parition找不到leader，会更新metadata
    if unknown_leaders_exist:
        log.debug('Unknown leaders exist, requesting metadata update')
        self._metadata.request_update()

    # remove any nodes we aren't ready to send to
    not_ready_timeout = float('inf')
    for node in list(ready_nodes):
        # 根据请求队列和连接情况以及metadata信息情况判断是不是可以发送请求
        if not self._client.ready(node):
            log.debug('Node %s not ready; delaying produce of accumulated batch', node)
            ready_nodes.remove(node)
            not_ready_timeout = min(not_ready_timeout,
                                    self._client.connection_delay(node))

    # create produce requests
    # 为每个节点发送回去相应的写入数据
    batches_by_node = self._accumulator.drain(
        self._metadata, ready_nodes, self.config['max_request_size'])
    # 如果是在要求保证顺序的情况下，将节点加入muted集合中，等到相应的response收到之后再移除
    if self.config['guarantee_message_order']:
        # Mute all the partitions drained
        for batch_list in six.itervalues(batches_by_node):
            for batch in batch_list:
                self._accumulator.muted.add(batch.topic_partition)

    expired_batches = self._accumulator.abort_expired_batches(
        self.config['request_timeout_ms'], self._metadata)
    for expired_batch in expired_batches:
        self._sensors.record_errors(expired_batch.topic_partition.topic, expired_batch.record_count)

    self._sensors.update_produce_request_metrics(batches_by_node)
    requests = self._create_produce_requests(batches_by_node)
    # If we have any nodes that are ready to send + have sendable data,
    # poll with 0 timeout so this can immediately loop and try sending more
    # data. Otherwise, the timeout is determined by nodes that have
    # partitions with data that isn't yet sendable (e.g. lingering, backing
    # off). Note that this specifically does not include nodes with
    # sendable data that aren't ready to send since they would cause busy
    # looping.
    poll_timeout_ms = min(next_ready_check_delay * 1000, not_ready_timeout)
    if ready_nodes:
        log.debug("Nodes with data ready to send: %s", ready_nodes) # trace
        log.debug("Created %d produce requests: %s", len(requests), requests) # trace
        poll_timeout_ms = 0

    for node_id, request in six.iteritems(requests):
        batches = batches_by_node[node_id]
        log.debug('Sending Produce Request: %r', request)
        (self._client.send(node_id, request)
             .add_callback(
                 self._handle_produce_response, node_id, time.time(), batches)
             .add_errback(
                 self._failed_produce, batches, node_id))

    # if some partitions are already ready to be sent, the select time
    # would be 0; otherwise if some partition already has some data
    # accumulated but not ready yet, the select time will be the time
    # difference between now and its linger expiry time; otherwise the
    # select time will be the time difference between now and the
    # metadata expiry time
    self._client.poll(poll_timeout_ms)

```
