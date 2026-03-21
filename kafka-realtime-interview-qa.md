# Kafka — Real-Time Deep Interview Questions & Answers

> Scenario-based questions asked at Netflix, Uber, LinkedIn, Confluent, PayPal, etc.

---

## SECTION 1 — Kafka Internals & Architecture

---

### Q1. A message is produced to Kafka. Walk me through exactly what happens internally until the consumer reads it.

**Answer:**

**Producer Side:**
1. Producer calls `send(ProducerRecord)`.
2. Serializer converts key/value to bytes.
3. Partitioner determines which partition — default: hash(key) % numPartitions, or round-robin if no key.
4. Message goes into an in-memory **RecordAccumulator** (per partition batch buffer).
5. **Sender thread** (background) picks up batches when `batch.size` is reached OR `linger.ms` elapses.
6. Sender opens a TCP connection to the **Leader broker** of that partition.
7. Broker writes the message to the **partition log file** (append-only on disk).
8. If `acks=all`, broker waits for all **ISR (In-Sync Replicas)** to acknowledge before responding.
9. Follower brokers **pull** the data from the leader (replication).
10. Producer receives acknowledgment (or retries on failure).

**Broker Storage:**
- Message is appended to a `.log` segment file in the partition directory.
- An `.index` file maps offsets to byte positions for fast seeks.
- A `.timeindex` file maps timestamps to offsets.

**Consumer Side:**
1. Consumer sends `FETCH` request to the **Leader broker** (or follower if rack-aware fetch enabled).
2. Broker finds the offset in the index file, reads from log segment.
3. Uses **zero-copy** (`sendfile` syscall) to transfer bytes from disk to network socket without copying to user space — this is why Kafka is fast.
4. Consumer deserializes, processes, and **commits the offset** back to the `__consumer_offsets` internal topic.

---

### Q2. What is ISR (In-Sync Replicas) and what happens when a replica falls out of ISR?

**Answer:**
ISR is the set of replicas that are fully caught up with the leader — within `replica.lag.time.max.ms` (default 30 seconds).

**When a replica falls out of ISR:**
- It's removed from the ISR list by the leader (communicated to Zookeeper/KRaft).
- The leader continues serving reads/writes with the remaining ISR.
- The lagging replica keeps fetching from the leader to catch up.
- Once it catches up, it's added back to ISR.

**Why this matters for producers:**
- With `acks=all`, the producer waits for ALL ISR members to acknowledge.
- If ISR shrinks to just the leader, `acks=all` only needs the leader to acknowledge — **your durability guarantee is weakened without you knowing**.
- To prevent this, set `min.insync.replicas=2` on the topic:
  - If ISR drops below 2, the broker throws `NotEnoughReplicasException` instead of accepting writes.
  - Combined with `acks=all`, this gives you true durability.

**Production scenario:**
```
Cluster: 3 brokers, replication-factor=3, min.insync.replicas=2
Broker 2 goes down → ISR = {1, 3}
Producer with acks=all can still write (ISR ≥ min.insync.replicas)
Broker 3 also goes down → ISR = {1}
Producer gets NotEnoughReplicasException — correct! Preventing data loss.
```

---

### Q3. What is the difference between `acks=0`, `acks=1`, and `acks=all`? Which one would cause duplicate messages and which would cause data loss?

**Answer:**

| Setting | Meaning | Data Loss Risk | Duplicate Risk | Latency |
|---|---|---|---|---|
| `acks=0` | Fire-and-forget, no ack waited | HIGH — broker crash = lost | Low (no retry needed) | Lowest |
| `acks=1` | Leader ack only | MEDIUM — leader crash before replication = lost | YES — leader acks but crashes before replication, producer retries | Low |
| `acks=all` (-1) | All ISR ack | LOW (with min.insync.replicas) | YES — if leader crashes after writing but before acking producer | Higher |

**How `acks=1` causes data loss:**
1. Producer sends message.
2. Leader writes to disk, sends `ack`.
3. Leader crashes before followers replicate.
4. New leader elected from followers — **message is gone**.
5. Producer thinks it succeeded — no retry.

**How `acks=all` causes duplicates:**
1. Producer sends message.
2. Leader + all ISR write the message.
3. Leader sends ack — but ack is lost in network.
4. Producer times out, retries — **same message written twice**.
5. Fix: Enable idempotent producer — `enable.idempotence=true`.

**`enable.idempotence=true`** assigns a sequence number to each message per partition. Broker deduplicates retries automatically. This also requires `acks=all`, `retries > 0`, and `max.in.flight.requests.per.connection ≤ 5`.

---

### Q4. You have a topic with 6 partitions and 3 consumer instances. One consumer is getting 4 partitions and others get 1 each. Why and what's the problem?

**Answer:**
This is a **partition assignment imbalance** caused by how the partition assignment strategy works.

**Default: `RangeAssignor`** — assigns contiguous ranges per topic. With 6 partitions and 3 consumers:
- Consumer 1: partitions 0, 1
- Consumer 2: partitions 2, 3
- Consumer 3: partitions 4, 5
This is actually balanced for one topic, but if you subscribe to **multiple topics**, RangeAssignor assigns per-topic ranges, leading to imbalance.

**With 2 topics × 6 partitions each + 3 consumers using RangeAssignor:**
- Consumer 1 gets topic-A partitions {0,1} + topic-B partitions {0,1} = 4 partitions
- Consumer 2 gets topic-A {2,3} + topic-B {2,3} = 4
- Consumer 3 gets topic-A {4,5} + topic-B {4,5} = 4
Actually balanced here but with odd counts it breaks.

**Real cause of 4-1-1:** Likely a **rebalance happened mid-consumption** and one consumer joined late, getting extra partitions before the group stabilized.

**Fixes:**
1. Use `StickyAssignor` — minimizes reassignment on rebalance:
   ```properties
   partition.assignment.strategy=org.apache.kafka.clients.consumer.StickyAssignor
   ```
2. Use `CooperativeStickyAssignor` — incremental rebalancing, consumers don't stop processing during rebalance:
   ```properties
   partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
   ```
3. Ensure consumer instances are stable — frequent restarts trigger rebalances.

---

### Q5. What is a consumer group rebalance? What are the performance implications and how do you minimize them?

**Answer:**
A rebalance is when Kafka redistributes partition ownership among consumers in a group. During a **stop-the-world rebalance** (default), ALL consumers stop processing while partitions are reassigned.

**Triggers:**
- Consumer joins the group (new instance scaled up).
- Consumer leaves (instance crashed, scaled down, or `session.timeout.ms` exceeded).
- Consumer fails to send heartbeat within `session.timeout.ms`.
- Topic partition count changed.
- Consumer subscribes to new topics.

**Performance implications:**
- During rebalance (seconds to minutes in large groups), **zero messages processed** — lag grows.
- Uncommitted offsets are reset to last committed — **messages reprocessed** after rebalance.
- Rebalance cost scales with number of consumers and partitions.

**Minimization strategies:**

1. **Tune timeouts to avoid false rebalances:**
```properties
session.timeout.ms=45000          # how long broker waits before considering consumer dead
heartbeat.interval.ms=15000       # how often consumer sends heartbeat (1/3 of session.timeout)
max.poll.interval.ms=600000       # max time between polls before considered dead
```

2. **Use `CooperativeStickyAssignor`** — only revoked partitions stop; others continue processing:
```properties
partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

3. **Static group membership** — consumer gets a fixed ID, can rejoin without triggering rebalance:
```properties
group.instance.id=consumer-instance-1  # unique per instance
```
If this consumer restarts within `session.timeout.ms`, it resumes without rebalance.

4. **Don't do long processing between polls** — if processing takes longer than `max.poll.interval.ms`, consumer is considered dead → rebalance triggered:
```java
// Reduce batch size so processing finishes quickly
max.poll.records=100  // default 500
```

---

## SECTION 2 — Producer Deep Dive

---

### Q6. You're getting `org.apache.kafka.common.errors.TimeoutException: Expiring 5 record(s) for topic-partition` in production. What causes this and how do you fix it?

**Answer:**
This means records sat in the `RecordAccumulator` buffer for longer than `delivery.timeout.ms` (default 120 seconds) without being sent or acknowledged.

**Root causes:**

1. **Broker is slow or unreachable** — sender can't get an ack in time.
   - Check broker metrics: request latency, disk I/O, GC pauses.

2. **Buffer is full** — `buffer.memory` (default 32MB) is exhausted:
   - Too many partitions, each with its own batch.
   - Producer sending faster than broker can accept.
   - Check `buffer-available-bytes` metric.

3. **`max.block.ms` reached** — producer blocked trying to get buffer space and gave up.

4. **Metadata fetch timeout** — broker list is wrong or brokers are down.

**Fixes:**
```properties
# Increase buffer
buffer.memory=67108864        # 64MB

# Increase delivery timeout (give more time for retries)
delivery.timeout.ms=300000    # 5 minutes

# Increase request timeout
request.timeout.ms=60000

# Reduce record size / use compression
compression.type=snappy

# For throughput: batch more aggressively
batch.size=65536              # 64KB
linger.ms=20                  # wait 20ms to fill batch
```

**Monitoring to prevent this:**
```java
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        metrics.counter("kafka.producer.error",
            "topic", record.topic(),
            "error", exception.getClass().getSimpleName()
        ).increment();
    }
});
```

---

### Q7. Explain exactly-once semantics (EOS) in Kafka. How does it actually work under the hood?

**Answer:**
EOS means each message is processed and its effect is visible exactly once — no duplicates, no data loss.

**Three levels:**
- **At-most-once** — may lose messages (acks=0, no retries).
- **At-least-once** — may duplicate (acks=all, retries enabled).
- **Exactly-once** — neither loss nor duplication.

**Building blocks:**

**1. Idempotent Producer** (`enable.idempotence=true`):
- Each producer instance gets a **Producer ID (PID)** from the broker.
- Each message gets a **sequence number** per partition.
- Broker deduplicates: if it sees PID + sequence already written, it discards the duplicate.
- Only handles duplicates within one producer session (PID resets on restart).

**2. Transactions** (for cross-partition atomic writes):
```java
producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("topic-A", key, value));
    producer.send(new ProducerRecord<>("topic-B", key, value));
    producer.commitTransaction();  // atomic: both written or neither
} catch (Exception e) {
    producer.abortTransaction();   // rolls back both
}
```

Under the hood:
- Broker has a **Transaction Coordinator** (a broker managing the `__transaction_state` internal topic).
- `beginTransaction()` tells the coordinator.
- Each write is tagged with the transaction ID.
- On commit: coordinator writes a commit marker to all affected partitions.
- Messages are only visible to consumers after the commit marker is written.

**3. Exactly-once in Kafka Streams (consume-process-produce):**
```java
props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
```
Kafka Streams wraps the entire consume → process → produce cycle in a transaction:
- Consumes from input topic.
- Processes.
- Produces to output topic AND commits offset — all in one atomic transaction.

**Limitation:** EOS only works within Kafka. If you write to a database AND Kafka, you need the Outbox pattern.

---

### Q8. You need to guarantee message ordering in Kafka. What are all the things you can accidentally break it with?

**Answer:**
Kafka guarantees ordering **within a partition only**. Here's every way you can break it:

**1. Multiple partitions with no key:**
```java
// Round-robin → messages go to different partitions → no order guarantee
producer.send(new ProducerRecord<>("orders", null, orderEvent));
// Fix: use a meaningful key
producer.send(new ProducerRecord<>("orders", orderId.toString(), orderEvent));
```

**2. `max.in.flight.requests.per.connection > 1` with retries:**
```
Batch 1 sent → fails
Batch 2 sent → succeeds
Batch 1 retried → succeeds
Result in partition: [Batch 2, Batch 1] — WRONG ORDER
```
Fix: `max.in.flight.requests.per.connection=1` (kills throughput) OR `enable.idempotence=true` (handles reordering with sequence numbers, allows up to 5 in-flight).

**3. Consumer processing in parallel:**
```java
// Pulling from one partition but processing in thread pool = order lost
for (ConsumerRecord<String, String> record : records) {
    executor.submit(() -> process(record)); // parallel = no order
}
// Fix: process sequentially or partition by key within thread pool
```

**4. Topic key changes:** If you add partitions to an existing topic, `hash(key) % numPartitions` changes — same key goes to different partition than before.

**5. Multiple consumer instances for same key's partition:**
One partition → one consumer instance. But if that consumer uses parallelism internally, ordering is lost.

**6. Producer `linger.ms` + `batch.size` can reorder across batches if network is lossy and retries happen.**

**Design for ordering:**
- Use the entity ID as the message key (orderId, userId, accountId).
- Set `enable.idempotence=true`.
- Process one partition → one thread.
- Never add partitions to a running topic if ordering is critical — create a new topic.

---

## SECTION 3 — Consumer Deep Dive

---

### Q9. Consumer lag is growing despite consumers running. You see `records-lag-max=500000`. How do you debug and fix this?

**Answer:**
Lag means consumers are falling behind producers. `records-lag-max=500000` means the slowest partition is 500K messages behind.

**Debugging steps:**

**Step 1 — Identify which consumers/partitions are lagging:**
```bash
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group my-group
# Shows: partition, current-offset, log-end-offset, lag, consumer-host
```

**Step 2 — Check if it's a consumption problem or a production spike:**
- Is the producer rate suddenly increased? Check producer metrics.
- Is consumer throughput stable or dropped?

**Step 3 — Root causes:**
1. **Processing is too slow** — each message takes too long:
   ```java
   // Are you making synchronous DB/HTTP calls per message?
   consumer.poll(Duration.ofMillis(100));
   for (record : records) {
       httpClient.post(record); // synchronous = slow
   }
   ```
   Fix: Batch DB writes, use async HTTP client.

2. **Consumer is constantly rebalancing** — check for frequent rebalances in logs.

3. **GC pauses** — long GC pauses trigger session timeout → rebalance.

4. **Single-threaded consumer can't keep up** — need more consumers.

5. **`max.poll.records` too low** — consuming 10 records per poll when you can handle 500:
   ```properties
   max.poll.records=500
   ```

**Scaling fixes:**
- Add more consumer instances (up to partition count).
- If already at partition count, increase partition count (careful with ordering).
- Process records in parallel within consumer using `CompletableFuture` (if ordering not required).

**Spring Kafka parallel processing:**
```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConcurrency(6); // 6 threads, one per partition
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
    return factory;
}
```

---

### Q10. Your consumer committed an offset but crashed before processing the message. The message is now lost. How do you prevent this?

**Answer:**
This is the **commit-before-process** problem — happens with `enable.auto.commit=true` (default).

**Default auto-commit behavior:**
```properties
enable.auto.commit=true
auto.commit.interval.ms=5000  # commits every 5 seconds
```
Auto-commit happens in `poll()` — it commits offsets from the PREVIOUS poll, but if your processing takes more than 5 seconds, it may commit offsets for records not yet fully processed.

**Fix: Manual commit with process-before-commit:**
```java
@KafkaListener(topics = "orders")
public void consume(ConsumerRecord<String, Order> record,
                    Acknowledgment acknowledgment) {
    try {
        orderService.process(record.value());  // process FIRST
        acknowledgment.acknowledge();           // THEN commit
    } catch (Exception e) {
        // Don't ack — message will be redelivered after rebalance
        // But you now get at-least-once, not exactly-once
    }
}
```
```java
factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
```

**AckMode options in Spring Kafka:**
| Mode | When committed |
|---|---|
| `RECORD` | After each record processed |
| `BATCH` | After each `poll()` batch processed |
| `MANUAL` | When `ack.acknowledge()` called (at next `poll()`) |
| `MANUAL_IMMEDIATE` | When `ack.acknowledge()` called, immediately |
| `COUNT` | After N records |
| `TIME` | After T milliseconds |

**Important:** Even with `MANUAL`, you get **at-least-once** — if consumer crashes after processing but before committing, message is reprocessed. For exactly-once, you need the transactional approach.

---

### Q11. What is the difference between `seek`, `seekToBeginning`, `seekToEnd`, and when would you use each in production?

**Answer:**
All three manually control where a consumer reads from, bypassing committed offsets.

```java
// Seek to specific offset
consumer.seek(new TopicPartition("orders", 0), 1000L);
// Start reading from offset 1000 on partition 0

// Seek to beginning (replay all messages)
consumer.seekToBeginning(consumer.assignment());

// Seek to end (skip all existing messages, only new ones)
consumer.seekToEnd(consumer.assignment());

// Seek by timestamp (very useful in production)
Map<TopicPartition, Long> timestamps = partitions.stream()
    .collect(Collectors.toMap(tp -> tp, tp -> startTime.toEpochMilli()));
Map<TopicPartition, OffsetAndTimestamp> offsets = consumer.offsetsForTimes(timestamps);
offsets.forEach((tp, offsetAndTimestamp) -> {
    if (offsetAndTimestamp != null) {
        consumer.seek(tp, offsetAndTimestamp.offset());
    }
});
```

**Real-world production scenarios:**

1. **Bug deployed that corrupted data** → seek to timestamp before deployment, replay to re-derive correct state.

2. **Consumer group created but topic already has messages** → `auto.offset.reset=latest` means new consumer starts from end. Change to `earliest` or use `seekToBeginning` to process all historical data.

3. **Disaster recovery** → production topic replicated to DR. Seek to the last offset your DR consumer processed.

4. **Skip a poison pill message** (message that keeps crashing consumer):
```java
// After identifying the bad offset
consumer.seek(new TopicPartition(topic, partition), badOffset + 1);
```

5. **Hourly reprocessing job** → seek to 1 hour ago via `offsetsForTimes` every run.

---

### Q12. What is a poison pill message? How do you handle it in production without stopping your entire consumer?

**Answer:**
A poison pill is a message that consistently causes the consumer to throw an exception — could be malformed data, unexpected schema, or a bug in processing logic.

**Default behavior:** Consumer retries indefinitely (or up to `max.poll.interval.ms`) → consumer is considered dead → rebalance → another consumer gets the same partition → same crash → group thrashing.

**Spring Kafka solution — `DefaultErrorHandler` with Dead Letter Topic (DLT):**

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> template) {
    // Retry 3 times with 1s, 2s, 4s backoff
    ExponentialBackOffWithMaxRetries backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1000L);
    backoff.setMultiplier(2.0);

    // After retries exhausted, send to DLT
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backoff);

    // Don't retry these — they'll never succeed
    handler.addNotRetryableExceptions(JsonParseException.class, DeserializationException.class);

    return handler;
}
```

**DLT message contains:**
- Original message payload.
- Headers with exception class, message, stack trace, original topic/partition/offset.

**DLT consumer for investigation/replay:**
```java
@KafkaListener(topics = "orders.DLT")
public void consumeDLT(ConsumerRecord<String, String> record) {
    String originalTopic = new String(record.headers().lastHeader("kafka_dlt-original-topic").value());
    String exceptionMessage = new String(record.headers().lastHeader("kafka_dlt-exception-message").value());
    // Alert, log, store for manual inspection
}
```

**Important:** Always monitor DLT size — growing DLT means a systemic bug, not a one-off bad message.

---

## SECTION 4 — Advanced Scenarios

---

### Q13. You need to consume from Kafka and write to a database atomically — either both happen or neither. How?

**Answer:**
This is the **dual write problem**. You can't span a Kafka consumer offset commit and a DB transaction in one atomic operation because they are different systems.

**Option 1 — Outbox Pattern (most reliable):**
```java
@Transactional
public void processKafkaMessage(Order order) {
    orderRepo.save(order);              // DB write
    outboxRepo.save(new OutboxEvent()); // DB write in same TX
    // Offset committed separately — if crash, message reprocessed
    // Idempotency check prevents double processing
}
```
A separate poller reads the outbox and publishes downstream. Offset commit happens after DB transaction succeeds.

**Option 2 — Store offsets in the DB (same transaction):**
```java
@Transactional
public void consume(ConsumerRecord<String, Order> record, Acknowledgment ack) {
    // Write offset TO the database in the same transaction as business data
    offsetRepo.save(new KafkaOffset(record.topic(), record.partition(), record.offset()));
    orderRepo.save(new Order(record.value()));
    // Commit DB transaction
    ack.acknowledge(); // Kafka offset commit — separate, may fail
}

// On startup, restore offsets from DB:
@EventListener(ConsumerStartedEvent.class)
public void onStart(ConsumerStartingEvent event) {
    offsets.forEach((tp, offset) -> consumer.seek(tp, offset + 1));
}
```

**Option 3 — Idempotent consumer:**
Accept at-least-once delivery and make your processing idempotent using a dedupe key:
```java
@Transactional
public void process(ConsumerRecord<String, Order> record) {
    String dedupKey = record.topic() + "-" + record.partition() + "-" + record.offset();
    if (processedEventRepo.existsByDedupKey(dedupKey)) {
        return; // already processed
    }
    orderRepo.save(order);
    processedEventRepo.save(new ProcessedEvent(dedupKey));
}
```

---

### Q14. Your Kafka topic has 50GB of data and consumers are reading slowly. How does Kafka serve this without loading everything into memory?

**Answer:**
This is about Kafka's storage and serving architecture — this question tests if you really understand Kafka vs a typical message queue.

**Key mechanisms:**

**1. Segment files — not one big file:**
Each partition is split into segment files (default `log.segment.bytes=1GB`):
```
/kafka-logs/orders-0/
  00000000000000000000.log   # oldest segment
  00000000000001048576.log
  00000000000002097152.log   # active segment
  00000000000000000000.index
  00000000000000000000.timeindex
```
Kafka only needs to open and read the relevant segment, not load all 50GB.

**2. Page cache — OS does the heavy lifting:**
Kafka doesn't maintain its own read cache. It relies on the OS page cache (Linux buffer cache). When a consumer reads, the OS caches those disk pages in RAM. Subsequent reads of the same data hit RAM, not disk.

This is why Kafka needs dedicated machines — the more RAM you give the OS for page cache, the better.

**3. Zero-copy transfer (`sendfile`):**
When consumer fetches data that's in page cache:
1. Normal path: Disk → Kernel buffer → User space → Socket buffer → Network.
2. Kafka's path: Disk → Kernel buffer → Network (via `sendfile` syscall).
No copy to user space — huge performance gain for high-throughput consumers.

**4. Sequential I/O:**
Kafka always appends to the end of segments (producers) and reads sequentially (consumers). Sequential I/O on SSDs/HDDs is orders of magnitude faster than random I/O. This is why a HDD can do 600MB/s sequential but only 100 IOPS random.

**5. Fetch size tuning for slow consumers:**
```properties
fetch.min.bytes=1048576    # wait until 1MB available before responding
fetch.max.wait.ms=500      # max wait for fetch.min.bytes to be available
max.partition.fetch.bytes=10485760  # max per partition per fetch (10MB)
```

---

### Q15. What is log compaction and when would you use it? What are its pitfalls?

**Answer:**
Log compaction retains only the **latest value per key** instead of deleting old messages based on time/size. It's how Kafka provides **changelog/state storage**.

```
Before compaction:    key=user1 → v1, key=user1 → v2, key=user1 → v3, key=user2 → v1
After compaction:     key=user1 → v3, key=user2 → v1
```

**Enable:**
```properties
cleanup.policy=compact            # vs delete (default) vs compact,delete (both)
min.cleanable.dirty.ratio=0.5    # compact when 50% of log is dirty (re-updated keys)
segment.ms=86400000              # create new segment daily (compaction only runs on closed segments)
```

**Use cases:**
1. **Event sourcing / CQRS** — topic as a changelog for a table.
2. **Kafka Streams state stores** — changelog topics for fault tolerance.
3. **Configuration service** — consumer reads the whole topic on start to rebuild current config state.
4. **Cache invalidation** — consumers maintain a local map, topic has latest values per key.

**Pitfalls:**

1. **Compaction doesn't happen immediately** — dirty (uncompacted) segments still exist. New consumers see duplicates until compaction runs.

2. **`null` value = tombstone** — publishing a `null` value for a key marks it for deletion:
   ```java
   producer.send(new ProducerRecord<>("users", userId, null)); // deletes user from compacted log
   ```
   Tombstones are retained for `delete.retention.ms` (default 24h) so consumers can see the deletion.

3. **Ordering is NOT maintained after compaction** — compacted log interleaves old and new records.

4. **Consumer reads may be inconsistent mid-compaction** — consumer may see old value then new value then old value as it spans segments.

5. **Key must not be null** — messages with null keys are never compacted and will eventually be deleted by size/time policy if combined `compact,delete`.

---

### Q16. Explain Kafka Streams vs Kafka Consumer API vs Kafka Connect. When do you use each?

**Answer:**

**Kafka Consumer API — raw building block:**
```java
KafkaConsumer<String, Order> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("orders"));
while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
    records.forEach(r -> process(r.value()));
}
```
Use when: Simple consumption, full control needed, not doing stream processing.

**Kafka Streams — stateful stream processing inside your app:**
```java
StreamsBuilder builder = new StreamsBuilder();
KStream<String, Order> orders = builder.stream("orders");

// Real-time aggregation: count orders per user in 5-minute windows
orders
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
    .count()
    .toStream()
    .to("order-counts-per-user");
```
Use when:
- Joins between topics (e.g., enrich orders with user data).
- Windowed aggregations (count, sum per time window).
- Stateful processing (the state is stored in RocksDB locally, backed up to changelog topic).
- No external cluster needed — runs inside your Spring Boot app.

**Kafka Connect — data pipeline, no code:**
```json
{
  "name": "jdbc-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://...",
    "table.whitelist": "orders",
    "mode": "timestamp",
    "timestamp.column.name": "updated_at",
    "topic.prefix": "db."
  }
}
```
Use when:
- **Source**: DB → Kafka (CDC with Debezium), S3 → Kafka, REST API → Kafka.
- **Sink**: Kafka → DB, Kafka → Elasticsearch, Kafka → S3.
- No custom processing — just moving data.
- Managed at-least-once delivery, offset tracking, fault tolerance built in.

**Decision matrix:**
| Scenario | Tool |
|---|---|
| Read from Kafka, write to DB | Consumer API or Kafka Connect Sink |
| Read from DB, write to Kafka | Kafka Connect Source (Debezium) |
| Join two Kafka topics | Kafka Streams |
| Aggregate in time windows | Kafka Streams |
| Simple event-driven processing | Consumer API |
| Real-time ETL pipeline | Kafka Connect + Kafka Streams |

---

## SECTION 5 — Spring Kafka Specific

---

### Q17. You have a `@KafkaListener` method. The deserialization fails for one bad message. What happens by default and what should you configure?

**Answer:**
**Default behavior:** `ErrorHandlingDeserializer` is NOT configured by default. A `DeserializationException` propagates up, consumer thread throws, Spring retries the same message indefinitely (or until `max.poll.interval.ms`), triggering a rebalance loop.

**Proper setup:**

```java
// 1. Wrap your deserializer with ErrorHandlingDeserializer
@Bean
public ConsumerFactory<String, Order> consumerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
    props.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class.getName());
    props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.myapp.*");
    return new DefaultKafkaConsumerFactory<>(props);
}

// 2. In the listener, handle null value (deserialization failure)
@KafkaListener(topics = "orders")
public void consume(ConsumerRecord<String, Order> record) {
    if (record.value() == null) {
        // Check headers for DeserializationException
        Header exHeader = record.headers().lastHeader(ErrorHandlingDeserializer.VALUE_DESERIALIZER_EXCEPTION_HEADER);
        // Log, send to DLT, skip
        return;
    }
    process(record.value());
}
```

**3. Configure DefaultErrorHandler to NOT retry deserialiation errors:**
```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> template) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(template);
    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer,
        new ExponentialBackOffWithMaxRetries(3));
    // Deserialization errors should go straight to DLT, no retry
    handler.addNotRetryableExceptions(DeserializationException.class);
    return handler;
}
```

---

### Q18. How do you implement retry with backoff in Spring Kafka? What's the difference between blocking retry and non-blocking retry?

**Answer:**

**Blocking retry (DefaultErrorHandler) — retries within the same poll loop:**
```java
@Bean
public DefaultErrorHandler errorHandler() {
    ExponentialBackOffWithMaxRetries backoff = new ExponentialBackOffWithMaxRetries(3);
    backoff.setInitialInterval(1000L);   // 1s
    backoff.setMultiplier(2.0);           // 1s, 2s, 4s
    backoff.setMaxInterval(10000L);       // cap at 10s
    return new DefaultErrorHandler(new DeadLetterPublishingRecoverer(kafkaTemplate), backoff);
}
```
**Problem:** During retries, the consumer thread is blocked — no other messages from that partition are processed. If retrying for 7 seconds (1+2+4), offset is not committed, heartbeats still sent though.

**Non-blocking retry (RetryTopic) — retries via separate retry topics:**
```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10000),
    dltTopicSuffix = ".DLT",
    retryTopicSuffix = "-retry"
)
@KafkaListener(topics = "orders")
public void consume(Order order) {
    orderService.process(order);
}
```

This creates topics: `orders-retry-1000`, `orders-retry-2000`, `orders-retry-4000`, `orders.DLT`.

Failed message is published to `orders-retry-1000`. After 1 second, retry consumer reads it. If still failing → `orders-retry-2000`. After 4 total attempts → `orders.DLT`.

**Key difference:**
| | Blocking Retry | Non-blocking Retry (`@RetryableTopic`) |
|---|---|---|
| Processing during retry | Blocked | Other messages processed normally |
| Retry delay | Thread sleep (blocks partition) | Message sits in retry topic |
| Lag impact | High (partition stalled) | Low |
| Ordering | Maintained | BROKEN — retry messages processed later |
| Use case | Low volume, ordering critical | High volume, ordering not critical |

---

### Q19. Your Kafka topic has schema evolution — a new field was added. Old consumers are running. How do you handle this without downtime?

**Answer:**
This is about **backward and forward compatibility**.

**Schema compatibility rules:**
- **Backward compatible** — new schema can read old data (new consumer reads old messages).
- **Forward compatible** — old schema can read new data (old consumer reads new messages).
- **Full compatible** — both backward and forward.

**Without Schema Registry (JSON):**
```java
// Producer adds new optional field
public class OrderEvent {
    private String orderId;
    private String userId;
    private String newOptionalField; // NEW — old consumers just ignore it with @JsonIgnoreProperties
}

// Consumer side — ignore unknown fields
@JsonIgnoreProperties(ignoreUnknown = true)
public class OrderEvent {
    private String orderId;
    private String userId;
    // newOptionalField not here — Jackson ignores it
}
```

**With Schema Registry (Avro — production best practice):**
```json
// v1 schema
{"type": "record", "name": "Order", "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "userId", "type": "string"}
]}

// v2 schema — backward compatible: new field has default
{"type": "record", "name": "Order", "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "userId", "type": "string"},
    {"name": "promoCode", "type": ["null", "string"], "default": null}
]}
```

Schema Registry enforces compatibility rules — it **rejects** schema changes that break compatibility. Messages include a schema ID prefix, consumers fetch the schema by ID and deserialize correctly regardless of version.

**Migration strategy for breaking changes:**
1. Create a new topic `orders-v2`.
2. Deploy new producers writing to `orders-v2`.
3. Dual-consume: consumers read both `orders` and `orders-v2` during transition.
4. Once all producers moved, stop consuming `orders`.

---

### Q20. A Kafka partition leader goes down. What happens to producers and consumers in that window?

**Answer:**
This is a critical operations question. The timeline:

**T=0: Leader broker crashes**

**Producer side:**
- Producer gets `NotLeaderOrFollowerException` or `LeaderNotAvailableException`.
- With `retries > 0` and `retry.backoff.ms=100`, producer automatically retries.
- During retry window, messages buffer in `RecordAccumulator`.
- If `delivery.timeout.ms` is exceeded and leader not recovered → `TimeoutException`.

**Consumer side:**
- Consumer's `FETCH` request to dead leader fails.
- Consumer gets metadata refresh, discovers new leader.
- Consumer sends `FETCH` to new leader.
- **Gap**: Depends on how far followers were replicated.

**Leader election:**
- Zookeeper (pre-2.8) or KRaft (2.8+) detects leader death via session timeout (default 6 seconds for Zookeeper).
- Controller broker (elected in KRaft) selects new leader from ISR.
- New leader election in KRaft: ~20-30ms (much faster than Zookeeper's ~30s!).

**Data loss scenarios:**
- If `acks=1` and leader had messages followers hadn't replicated → those messages are lost.
- If `acks=all` with `min.insync.replicas=2` → no data loss because followers had the data before ack was sent.

**Unclean leader election:**
```properties
unclean.leader.election.enable=false  # default since Kafka 0.11
```
If `true`, an out-of-sync replica can become leader — data loss but higher availability. Always keep `false` for financial/critical data.

---

### Q21. What is `__consumer_offsets` topic and why should you never manually write to it?

**Answer:**
`__consumer_offsets` is an internal Kafka topic (50 partitions by default) that stores committed consumer group offsets. It's a compacted topic keyed by `(group_id, topic, partition)`.

**What's stored:**
```
Key:   [GroupId="order-processor", Topic="orders", Partition=2]
Value: [Offset=100459, Metadata="", CommitTimestamp=1234567890]
```

**Why you should NEVER write to it manually:**

1. **Schema is internal and can change** — Kafka team has changed the binary format between versions. Direct writes in old format corrupt the topic.

2. **Race conditions** — the consumer group coordinator manages this atomically with group state. Manual writes bypass coordination, causing split-brain between what the broker thinks is committed and actual consumer position.

3. **Compaction timing** — you write offset 100, compaction hasn't run, consumer reads the old offset from before your write. Unpredictable behavior.

4. **Broker caches offset state in memory** — writing to the topic directly doesn't update the in-memory cache. The broker still returns the old offset until it re-reads the partition.

**Correct ways to reset offsets:**
```bash
# Stop consumer group first, then:
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --group my-group --reset-offsets \
  --topic orders:0 \
  --to-offset 50000 \
  --execute
```
Or programmatically via `AdminClient`:
```java
adminClient.alterConsumerGroupOffsets(groupId,
    Map.of(new TopicPartition("orders", 0), new OffsetAndMetadata(50000L)));
```

---

### Q22. You have a Kafka consumer that reads 1000 records, sends them to an external API in bulk, but the API fails for some records and succeeds for others. How do you handle partial failures?

**Answer:**
This is one of the hardest real-world Kafka problems. You cannot partially commit offsets — you either commit all 1000 or none.

**Approach 1 — Track failed records, store for retry:**
```java
@KafkaListener(topics = "events")
public void consume(List<ConsumerRecord<String, Event>> records, Acknowledgment ack) {
    List<ConsumerRecord<String, Event>> failed = new ArrayList<>();

    for (ConsumerRecord<String, Event> record : records) {
        try {
            externalApi.send(record.value());
        } catch (PartialFailureException e) {
            failed.add(record);
        }
    }

    // Store failed records in a retry table / separate Kafka topic
    if (!failed.isEmpty()) {
        failed.forEach(r -> retryKafkaTemplate.send("events-retry", r.key(), r.value()));
    }

    // Commit ALL offsets — retries handled separately
    ack.acknowledge();
}
```

**Approach 2 — Bulk API call with response mapping:**
```java
BulkResponse response = externalApi.bulkSend(records);
for (int i = 0; i < records.size(); i++) {
    if (response.getItem(i).isFailed()) {
        // Send this specific record to DLT with failure reason
        dltTemplate.send(new ProducerRecord<>("events.DLT",
            records.get(i).key(),
            records.get(i).value()));
    }
}
ack.acknowledge(); // commit all — failures went to DLT
```

**Approach 3 — Seek back to first failed record (at-least-once for all-or-nothing):**
```java
// If ANY record fails, don't commit — reprocess entire batch
// Risk: infinite loop if one record is always bad
// Must combine with a dead letter mechanism after N retries
```

**Best practice:** Approach 1 or 2. Make the external API call idempotent (send an idempotency key) so reprocessing from retry topic doesn't cause duplicates on the API side.

---

### Q23. How does Kafka handle backpressure? What happens when your consumer is slower than the producer?

**Answer:**
Unlike reactive streams, Kafka does NOT have built-in backpressure signals. The producer doesn't slow down because the consumer is slow. Instead:

**What actually happens:**
1. Messages accumulate in the partition log on the broker.
2. Consumer lag grows — `log-end-offset - committed-offset` increases.
3. Broker stores messages until `retention.ms` (default 7 days) or `retention.bytes` is hit.
4. After retention period, old messages are deleted — **consumer may miss them if too slow**.

**Kafka's "backpressure" mechanisms:**

1. **Producer-side: `buffer.memory` and `max.block.ms`**
   - When broker is slow and the buffer fills, `send()` blocks for `max.block.ms`, then throws.
   - This is implicit backpressure — application code must handle the exception.

2. **Consumer-side: `fetch.min.bytes` and `fetch.max.wait.ms`**
   - Consumer controls how much to fetch per request.
   - Reduce `max.poll.records` to process smaller batches.

3. **Quota-based throttling** (Kafka feature):
```bash
kafka-configs.sh --alter --add-config 'producer_byte_rate=1048576'
  --entity-type clients --entity-name my-producer
```
Broker throttles producers exceeding the quota — network response is delayed.

**Real-world solution for slow consumer:**
1. Scale consumers horizontally (add instances, up to partition count).
2. Increase `max.poll.records` + batch process (DB bulk insert instead of one-by-one).
3. Process asynchronously with bounded queue:
```java
ExecutorService pool = new ThreadPoolExecutor(10, 20, 60L, SECONDS,
    new LinkedBlockingQueue<>(1000),            // bounded queue = backpressure
    new ThreadPoolExecutor.CallerRunsPolicy()); // slow down Kafka poll if queue full
```
4. If consumer is I/O bound on external service, increase concurrency with `@KafkaListener` `concurrency`.

---

### Q24. What is the difference between `commitSync` and `commitAsync` in Kafka? When does each fail silently?

**Answer:**

**`commitSync()`:**
- Blocks until the broker acknowledges the commit.
- Retries automatically on retriable failures.
- Throws exception on non-retriable failures.
- Slower — adds latency to every poll cycle.
```java
try {
    consumer.commitSync();
} catch (CommitFailedException e) {
    // Group rebalanced, this consumer no longer owns these partitions
    log.error("Commit failed", e);
}
```

**`commitAsync()`:**
- Non-blocking — returns immediately.
- Does NOT retry on failure (because a retry might commit an older offset after a newer one was already committed).
- Calls the callback with error info.
- **Silent failure risk:** If you don't provide a callback, failed commits are silently ignored.
```java
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Async commit failed for offsets {}", offsets, exception);
        // Don't retry here — may cause out-of-order commits
    }
});
```

**Best pattern — async normally, sync on shutdown:**
```java
try {
    while (running) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        process(records);
        consumer.commitAsync(); // fast path
    }
} finally {
    try {
        consumer.commitSync(); // ensure last offsets committed on shutdown
    } finally {
        consumer.close();
    }
}
```

**When `commitAsync` fails silently:** When you pass `null` as the callback — Kafka logs at WARN level internally but your application has no visibility. Always pass a callback.

**`CommitFailedException`** — thrown when the group has rebalanced and this consumer no longer owns the partitions it's trying to commit. Your processing was wasted — those messages will be reprocessed by the new owner.

---

### Q25. Kafka broker disk is filling up. You can't add new brokers right now. What do you do immediately and what is your longer-term plan?

**Answer:**
**Immediate actions:**

1. **Reduce retention on high-volume topics:**
```bash
kafka-configs.sh --bootstrap-server broker:9092 --alter \
  --entity-type topics --entity-name high-volume-topic \
  --add-config retention.ms=86400000  # reduce to 1 day from 7
```

2. **Enable log compaction on topics where only latest value matters:**
```bash
kafka-configs.sh --alter --entity-type topics --entity-name user-profiles \
  --add-config cleanup.policy=compact
```

3. **Add retention bytes limit:**
```bash
kafka-configs.sh --alter --entity-type topics --entity-name events \
  --add-config retention.bytes=10737418240  # 10GB per partition
```

4. **Enable compression on producers** — reduces disk usage by 50-80% for JSON:
```bash
kafka-configs.sh --alter --entity-type topics --entity-name events \
  --add-config compression.type=lz4  # or snappy
```

5. **Identify top space consumers:**
```bash
du -sh /kafka/data/*/  | sort -rh | head -20
kafka-log-dirs.sh --bootstrap-server broker:9092 \
  --topic-list high-volume-topic --describe
```

6. **Trigger log cleanup manually:**
```bash
kafka-configs.sh --alter --entity-type brokers --entity-name 1 \
  --add-config log.retention.check.interval.ms=1000
```

**Longer-term plan:**
1. **Tiered storage** (Kafka 3.6+) — move cold segments to S3/GCS automatically.
2. **Capacity planning** — set `retention.bytes` per topic based on throughput × desired retention.
3. **Separate clusters** by criticality — don't let analytics topics eat disk needed for transactional topics.
4. **Monitoring** — alert at 70% disk usage, page at 85%.
5. **Partition rebalancing** — if imbalanced, some brokers may be full while others are empty:
```bash
kafka-reassign-partitions.sh  # reassign hot partitions to less-used brokers
```
6. **Producer-side**: Ensure compression is enabled in producer config, not just topic config.

---
