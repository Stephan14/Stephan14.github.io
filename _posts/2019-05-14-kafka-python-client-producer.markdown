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

## ProducerBatch结构

在介绍`KafkaClient`、`RecordAccumulator`、`Sender`上述三种结构之前，先介绍一下
`ProducerBatch`这个结构，它是生产者通过batch发送数据的基础。

通过函数`try_append`向batch中添加数据：
```
def try_append(self, timestamp_ms, key, value, headers):
    metadata = self.records.append(timestamp_ms, key, value, headers)
    if metadata is None:
        return None

    self.max_record_size = max(self.max_record_size, metadata.size)
    self.last_append = time.time()
    future = FutureRecordMetadata(self.produce_future, metadata.offset,
                                  metadata.timestamp, metadata.crc,
                                  len(key) if key is not None else -1,
                                  len(value) if value is not None else -1,
                                  sum(len(h_key.encode("utf-8")) + len(h_val) for h_key, h_val in headers) if headers else -1)
    return future
```
records变量为MemoryRecordsBuilder的对象，在MemoryRecordsBuilder内部根据不同的版本构建不同的类来存储数据，通过append函数将数据转换成二进制写入到_buffer变量中，当Sender线程发送数据或者丢弃掉过期的数据的时候会对数据`ProducerBatch`中的数据进行压缩。一个produce_future对应多个FutureRecordMetadata对象，FutureRecordMetadata对象返回给上层send函数，用户可以自定义回调函数，回调函数的参数类型为RecordMetadata。当produce_future对象的success或者failure函数被调用的时候，就会触发调用FutureRecordMetadata对象的success或者failure函数，最终触发用户自定义的回调函数。

当用户缓存的`ProduerBatch`被发送并收到发送成功或者发送失败的response的时候，会调用`done`函数，最终调用用户的自定义的回调函数：
```
def done(self, base_offset=None, timestamp_ms=None, exception=None):
    level = logging.DEBUG if exception is None else logging.WARNING
    log.log(level, "Produced messages to topic-partition %s with base offset"
              " %s and error %s.", self.topic_partition, base_offset,
              exception)  # trace
    if self.produce_future.is_done:
        log.warning('Batch is already closed -- ignoring batch.done()')
        return
    elif exception is None:
        self.produce_future.success((base_offset, timestamp_ms))
    else:
        self.produce_future.failure(exception)
```
上述代码中`self.produce_future.success`和`self.produce_future.failure`的调用，会回调`FutureRecordMetadata`中的`_produce_success`和`failure`函数，这两个回调函数的设置可以在`FutureRecordMetadata`的构造函数中看到，代码如下：
```
def __init__(self, produce_future, relative_offset, timestamp_ms, checksum, serialized_key_size, serialized_value_size, serialized_header_size):
    super(FutureRecordMetadata, self).__init__()
    self._produce_future = produce_future
    # packing args as a tuple is a minor speed optimization
    self.args = (relative_offset, timestamp_ms, checksum, serialized_key_size, serialized_value_size, serialized_header_size)
    produce_future.add_callback(self._produce_success)
    produce_future.add_errback(self.failure)
```
其中的`_produce_success`函数会调用`FutureRecordMetadata`的回调函数。

`ProducerBatch`类中还有一个函数`maybe_expire`，这个函数主要是用来判断`ProduerBatch`是不是已经超时了，比如batch已经满了等待时间已经超过了设置的超时时间或者在重试的过程中超时等情况，此时设置error为`KafkaTimeoutError`

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
                            # 如果所有的发送的数据超过max_request_size的大小并且已经从batch中取到数据，则结束数据的获取；如果第一个batch中的数据就达到了max_request_size的大小限制，就只发送这一个batch的数据，从而保证每个request的大小不超过max_request_size的配置
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
                                # 会对ProducerBatch中的数据进行压缩
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
函数`reenqueue`由`_complete_batch`函数在重试的时候调用，重试是按照`ProducerBatch`粒度来重试：
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

最后还有一个`abort_expired_batches`函数，这个函数是用来判断一个本地缓存中还没有发送出去并且等待时间超过`request_timeout_ms`参数的`ProduerBatch`。因为我们通常理解的timeout是客户端发送出去request之后超过一定的时间没有收到response而导致的超时，但是，还有一种情况就是客户端要发送的数据太多了，导致本地缓存积攒的很多的数据，这些数据还没有来的及拷贝发送到socket中，如果这样积攒的时间过长，对于上层的用户来说的也算是timeout。代码中有两个需要特别注意的地方
```
def abort_expired_batches(self, request_timeout_ms, cluster):
    expired_batches = []
    to_remove = []
    count = 0
    for tp in list(self._batches.keys()):
        assert tp in self._tp_locks, 'TopicPartition not in locks dict'

        # We only check if the batch should be expired if the partition
        # does not have a batch in flight. This is to avoid the later
        # batches get expired when an earlier batch is still in progress.
        # This protection only takes effect when user sets
        # max.in.flight.request.per.connection=1. Otherwise the expiration
        # order is not guranteed.
        # 对于那些需要保证顺序的partition的ProducerBatch来说，其不会在考虑范围中
        if tp in self.muted:
            continue

        with self._tp_locks[tp]:
            # iterate over the batches and expire them if they have stayed
            # in accumulator for more than request_timeout_ms
            dq = self._batches[tp]
            for batch in dq:
                # 对于有多个ProducerBatch但是当前parition不是最新的ProducerBatch或者只有一个ProducerBatch，但是这个ProducerBatch的size已经达到了参数配置的要求，认为这个batch已经满了
                is_full = bool(bool(batch != dq[-1]) or batch.records.is_full())
                # check if the batch is expired
                if batch.maybe_expire(request_timeout_ms,
                                      self.config['retry_backoff_ms'],
                                      self.config['linger_ms'],
                                      is_full):
                    expired_batches.append(batch)
                    to_remove.append(batch)
                    count += 1
                    self.deallocate(batch)
                else:
                    # 如果前面的ProducerBatch没有过期，后面的就不用看了
                    # Stop at the first batch that has not expired.
                    break

            # Python does not allow us to mutate the dq during iteration
            # Assuming expired batches are infrequent, this is better than
            # creating a new copy of the deque for iteration on every loop
            if to_remove:
                for batch in to_remove:
                    dq.remove(batch)
                to_remove = []

    if expired_batches:
        log.warning("Expired %d batches in accumulator", count) # trace

    return expired_batches
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
    # 考虑如果集群正在升级重启，会不会导致丢失response，导致一直hang在这？
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
    # 如果是在要求保证顺序的情况下，将partition加入muted集合中，等到相应的response收到之后再移除
    if self.config['guarantee_message_order']:
        # Mute all the partitions drained
        for batch_list in six.itervalues(batches_by_node):
            for batch in batch_list:
                self._accumulator.muted.add(batch.topic_partition)
    # 对于那些缓存在本地超时没有发出去的ProducerBatch丢弃掉,此时需要用户自定义回调函数来处理这种失败的情况               
    expired_batches = self._accumulator.abort_expired_batches(
        self.config['request_timeout_ms'], self._metadata)
    for expired_batch in expired_batches:
        self._sensors.record_errors(expired_batch.topic_partition.topic, expired_batch.record_count)
    # 为每个节点构造一个request
    self._sensors.update_produce_request_metrics(batches_by_node)
    requests = self._create_produce_requests(batches_by_node)
    # If we have any nodes that are ready to send + have sendable data,
    # poll with 0 timeout so this can immediately loop and try sending more
    # data. Otherwise, the timeout is determined by nodes that have
    # partitions with data that isn't yet sendable (e.g. lingering, backing
    # off). Note that this specifically does not include nodes with
    # sendable data that aren't ready to send since they would cause busy
    # looping.
    # next_ready_check_delay 表示batch等待发送的最小时间
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

上面的`_handle_produce_response`函数处理收到的response, 包括回收缓存、处理重试错误，将需要有序的parition从muted集合中移除等操作，对于`ack`为0的情况下不处理收到的error code，也就不会进行重试。而对于`_failed_produce`函数，无论对于参数`ack`设置成什么值，都会根据相应的error code进行相应的重试逻辑


## KafkaClient结构
`KafkaClient`是kafka客户端中非常重要的一个类，consumer和producer都依赖这个类实现一些功能，先来介绍一下期中被使用的次数的函数`poll`,这个函数是尝试读写socket中个数据：
```
def poll(self, timeout_ms=None, future=None):
    if future is not None:
        timeout_ms = 100
    elif timeout_ms is None:
        timeout_ms = self.config['request_timeout_ms']
    elif not isinstance(timeout_ms, (int, float)):
        raise TypeError('Invalid type for timeout: %s' % type(timeout_ms))

    # Loop for futures, break after first loop if None
    responses = []
    while True:
        with self._lock:
            if self._closed:
                break
            # 对于不在broker不在metadata中和端口发生变换的broker进行重连
            # Attempt to complete pending connections
            for node_id in list(self._connecting):
                self._maybe_connect(node_id)

            # Send a metadata request if needed
            metadata_timeout_ms = self._maybe_refresh_metadata()

            # If we got a future that is already done, don't block in _poll
            if future is not None and future.is_done:
                timeout = 0
            else:
                idle_connection_timeout_ms = self._idle_expiry_manager.next_check_ms()
                # 获取最小的timeout时间保证线程阻塞的时间最短
                timeout = min(
                    timeout_ms,
                    metadata_timeout_ms,
                    idle_connection_timeout_ms,
                    self.config['request_timeout_ms'])
                timeout = max(0, timeout / 1000)  # avoid negative timeouts

            self._poll(timeout)

            responses.extend(self._fire_pending_completed_requests())

        # If all we had was a timeout (future is None) - only do one poll
        # If we do have a future, we keep looping until it is done
        if future is None or future.is_done:
            break

    return responses
```

对于`_maybe_refresh_metadata`函数中，如果元信息的ttl或者request_timeout_ms大于0，返回其最大值；如果broker连不上了则返回reconnect_backoff_ms；如果连接上了则返回request_timeout_ms，也就意味只有在**metadata数据的元信息已经过期并且没有正在刷新并且请求量最小的broker可以继续发送请求**的情况下才可以继续更新metadata相关的数据。着关于_poll()函数随后介绍一下，其主要功能是通过io多路复用的方式获取socket中数据，将数据以及`Future`对象放到放到`_pending_completion`中，然后通过函数`_fire_pending_completed_requests`将`response`以及相应的`Future`对象的状态置位为success。


下面介绍一下`_poll`函数，这个函数的核心就是调用select函数实现io多路复用，当时可以根据用户的需求配置epoll、poll等函数。代码如下：
```
def _poll(self, timeout):
    processed = set()

    start_select = time.time()
    ready = self._selector.select(timeout)
    end_select = time.time()
    if self._sensors:
        self._sensors.select_time.record((end_select - start_select) * 1000000000)

    for key, events in ready:
        # _wake_r为非阻塞的socket
        # 当想让poll线程不阻塞的时候，通过_wake_r唤醒当前线程,这样就不会阻塞sender线程
        if key.fileobj is self._wake_r:
            self._clear_wake_fd()
            continue
        elif not (events & selectors.EVENT_READ):
            continue
        conn = key.data
        processed.add(conn)

        if not conn.in_flight_requests:
            # if we got an EVENT_READ but there were no in-flight requests, one of
            # two things has happened:
            #
            # 1. The remote end closed the connection (because it died, or because
            #    a firewall timed out, or whatever)
            # 2. The protocol is out of sync.
            #
            # either way, we can no longer safely use this connection
            #
            # Do a 1-byte read to check protocol didnt get out of sync, and then close the conn
            try:
                unexpected_data = key.fileobj.recv(1)
                if unexpected_data:  # anything other than a 0-byte read means protocol issues
                    log.warning('Protocol out of sync on %r, closing', conn)
            except socket.error:
                pass
            conn.close(Errors.KafkaConnectionError('Socket EVENT_READ without in-flight-requests'))
            continue

        self._idle_expiry_manager.update(conn.node_id)
        self._pending_completion.extend(conn.recv())

    # Check for additional pending SSL bytes
    if self.config['security_protocol'] in ('SSL', 'SASL_SSL'):
        for conn in self._conns.values():
            if conn not in processed and conn.connected() and conn._sock.pending():
                self._pending_completion.extend(conn.recv())
    # 判断每个连接中是否有超时的请求，这些请求是通过select发现不了的
    for conn in six.itervalues(self._conns):
        if conn.requests_timed_out():
            log.warning('%s timed out after %s ms. Closing connection.',
                        conn, conn.config['request_timeout_ms'])
            conn.close(error=Errors.RequestTimedOutError(
                'Request timed out after %s ms' %
                conn.config['request_timeout_ms']))

    if self._sensors:
        self._sensors.io_time.record((time.time() - end_select) * 1000000000)

    self._maybe_close_oldest_connection()
```
上述函数中对已经注册的`socket`(已经被封装成为`SelectorKey`)进行监听，判断这个`socket`是不是已经可读、可写，获取已经就绪的`socket`对应的`broker connection`，对于那些broker的请求队列为空的情况下(通常是因为对端关闭连接或者协议不同步导致的),关闭连接；同时会更新相关的`broker`的连接接受数据的时间，方便后续客户端关闭空闲连接。其中有一个特殊的`socket`为`_wake_r`,其是通过`socketpair`创建的之一，其主要功能是用于`sender`线程和`poll`线程之间进行通信。对于每个`broker`的连接，还会判断其发送的请求有没有超时的，如果超时了关闭连接并且将请求队列上的请求设置为超时状态，会新建连接让其重试。最后，会关闭空闲时间比较长的连接。

上面介绍了从socket中获取数据，下面介绍一个客户端向sokcet中发送数据的函数，对于生产者来说其实只发送两种请求，分别为`MetadataRequset`和`ProdcueRequest`，其最终都是通过调用`KafkaClinet`中的`send`函数实现的：
```
def send(self, node_id, request, wakeup=True):
    conn = self._conns.get(node_id)
    if not conn or not self._can_send_request(node_id):
        self.maybe_connect(node_id, wakeup=wakeup)
        return Future().failure(Errors.NodeNotReadyError(node_id))

    # conn.send will queue the request internally
    # we will need to call send_pending_requests()
    # to trigger network I/O
    future = conn.send(request, blocking=False)

    # Wakeup signal is useful in case another thread is
    # blocked waiting for incoming network traffic while holding
    # the client lock in poll().
    if wakeup:
        self.wakeup()

    return future
```
在通过此函数向一个`broker`发送请求实，首先会判断客户端是不是已经连接上该`broker`或者该发送该broker的请求队列是不是已经是满的了，如果连接不上或者请求队列已经满了，则会进行将该`broker`加入到一个队列中进行排队等待连接，等待在`poll`的时候进行重连，此时，对于请求队列已经满的broker不会进行重连。对于其中的`conn.send(request, blocking=False)`函数，在发送之前会判断与`broker`连接的状态，然后将响应的`request`对象转换成二进制数据放到内存中，再生成一个`correlation_id`，并将`correlation_id`和`future`对象放入到`in_flight_requests`中，以方便在收到`response`之后进行相应的处理。如果`blocking`设置为`True`的情况下，会将刚才提的二进制数据通过`socket`发送出去，如果发送超时会关闭连接,最后是向`socketpair`发送数据，唤醒`poll`线程从而释放锁。
