# 15 — Kafka Fundamentals

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

---

## 🟢 Start Here — Kafka in Plain English

### What is Kafka?

Kafka is a **messaging system** — it lets different applications send messages to each other. Think of it like a **post office for your microservices**.

**Without Kafka:** Service A calls Service B directly. If B is down, A fails too.

**With Kafka:** Service A drops a message in the mailbox (Kafka). Service B picks it up whenever it's ready. If B is down, the message waits safely until B comes back.

### The key concepts — using a newspaper analogy

| Kafka concept | Newspaper analogy |
|--------------|-------------------|
| **Topic** | A newspaper section (Sports, Business, Tech) |
| **Producer** | A journalist who writes articles and submits them to a section |
| **Consumer** | A reader who subscribes to sections they care about |
| **Partition** | Multiple printing presses for the same section (parallel processing) |
| **Consumer Group** | A team of readers who split the work — each reads different pages |
| **Offset** | Your bookmark — "I've read up to page 42" |

### How it works (simplified)

```
Producer (Order Service)  →  Topic: "orders"  →  Consumer (Payment Service)
                                                  Consumer (Shipping Service)
                                                  Consumer (Analytics Service)

Order Service publishes: "New order: #123, iPhone, $999"
- Payment Service reads it → charges the customer
- Shipping Service reads it → arranges delivery
- Analytics Service reads it → updates dashboards

Each consumer reads INDEPENDENTLY — they don't interfere with each other.
```

### Why partitions matter

A topic can be split into **partitions** — think of it like multiple checkout lanes at a grocery store. More lanes = more customers served in parallel.

```
Topic "orders" with 3 partitions:
  Partition 0: orders for customers A-H
  Partition 1: orders for customers I-P
  Partition 2: orders for customers Q-Z

3 consumers can read simultaneously — one per partition!
```

**Important rule:** Messages within ONE partition are always in order. Across partitions, order is NOT guaranteed. Use a **partition key** (like customer ID) to keep related messages together.

### What happens when things go wrong?

- **Consumer crashes?** No problem — Kafka remembers where it left off (offset). When it restarts, it picks up where it stopped.
- **Message can't be processed?** Send it to a **Dead Letter Topic** (DLT) — a "problem messages" bin that humans can review later.
- **Kafka server crashes?** Messages are **replicated** across multiple servers — no data loss.

> Key takeaway: Kafka = a post office for services. Topics = categories. Partitions = parallel lanes. Consumer groups = teams sharing the work. Messages are durable — they survive crashes.

---

## 📚 Study Material

### 1. What Kafka IS (vs Traditional Message Queues)

```
Traditional MQ (RabbitMQ, ActiveMQ):         Kafka:
- Message consumed → DELETED                - Message consumed → STAYS (retention period)
- Point-to-point or pub-sub                 - Append-only immutable LOG
- Push-based (broker pushes to consumer)     - Pull-based (consumer pulls at its own pace)
- No replay                                  - Replay by resetting offset
- Single consumer per message               - Multiple consumer groups read independently
```

**Kafka is a distributed, append-only commit log.** Think of it as an infinitely growing file where producers append to the end, and consumers independently read at their own position (offset).

**When to use Kafka:**
- Decouple services (event-driven architecture)
- High-throughput data pipelines (millions of messages/sec)
- Event sourcing / audit trail
- Real-time stream processing
- Change Data Capture (CDC)

### 2. Architecture — How the Pieces Fit

```
                          ┌──── Kafka Cluster ─────┐
                          │                         │
Producer ─────→  Topic: "payments"                  │
                 ┌────────────────────┐             │
                 │ Partition 0 [Leader: Broker 1]   │──→ Consumer Group: payment-svc
                 │ █ █ █ █ █ █ █ █ █               │    ├─ Consumer A → reads P0
                 │ offset 0 1 2 3 4 5 6 7 8        │    ├─ Consumer B → reads P1
                 ├────────────────────┤             │    └─ Consumer C → reads P2
                 │ Partition 1 [Leader: Broker 2]   │
                 │ █ █ █ █ █ █                     │──→ Consumer Group: audit-svc
                 │ offset 0 1 2 3 4 5              │    ├─ Consumer X → reads P0, P1
                 ├────────────────────┤             │    └─ Consumer Y → reads P2
                 │ Partition 2 [Leader: Broker 3]   │
                 │ █ █ █ █ █ █ █ █                 │
                 │ offset 0 1 2 3 4 5 6 7          │
                 └────────────────────┘             │
                          └─────────────────────────┘

Each partition is replicated across brokers for fault tolerance.
```

**Key components:**

| Component | What it is |
|-----------|-----------|
| **Broker** | Kafka server process. Cluster = multiple brokers. |
| **Topic** | Named category (like a table). E.g., `payments`, `trades`, `audit-log` |
| **Partition** | Ordered, immutable sequence within a topic. Unit of parallelism. |
| **Offset** | Position of a message within a partition (monotonically increasing) |
| **Producer** | Publishes messages to topics |
| **Consumer** | Reads messages from topics |
| **Consumer Group** | Set of consumers sharing the work of consuming a topic |
| **KRaft** | Replaces ZooKeeper (Kafka 4.0+) for metadata management |

### 3. Partitions — Ordering & Parallelism

```java
// Ordering is guaranteed ONLY WITHIN a partition, NOT across partitions
// Partition key determines which partition a message goes to

// Example: all events for customer "C001" go to the same partition
producer.send(new ProducerRecord<>("payments", "C001", paymentEvent));
//                                   topic      key    value
// Kafka: hash("C001") % numPartitions → always the same partition

// WHY this matters:
// Events for C001: [created → authorized → captured → settled]
// These MUST be processed in order → same partition key guarantees it

// Messages with DIFFERENT keys may go to different partitions
// → no ordering guarantee between customers (and that's fine)

// NO KEY → round-robin (or sticky partitioning in newer versions)
producer.send(new ProducerRecord<>("logs", null, logEntry));
// Use for: event logs where ordering doesn't matter, maximise throughput
```

### 4. Consumer Groups — Scaling Consumption

```
Consumer Group "payment-svc" (3 consumers, 6 partitions):
  Consumer A → P0, P1    (2 partitions each)
  Consumer B → P2, P3
  Consumer C → P4, P5

Consumer Group "audit-svc" (2 consumers, same 6 partitions):
  Consumer X → P0, P1, P2  (independent — reads the SAME data)
  Consumer Y → P3, P4, P5

Rules:
✅ Each partition is assigned to EXACTLY ONE consumer within a group
✅ One consumer can handle MULTIPLE partitions
❌ More consumers than partitions → some consumers sit idle
✅ Multiple groups consume the same topic independently (fan-out)
```

**Rebalancing:** When a consumer joins, leaves, or crashes → Kafka reassigns partitions. During rebalance, consumption pauses.
```
Minimise rebalance disruption:
- CooperativeStickyAssignor — incremental rebalance (only reassigns affected partitions)
- Static group membership — group.instance.id (avoids rebalance on rolling restarts)
- Heartbeat tuning — session.timeout.ms, heartbeat.interval.ms
```

### 5. Offset Management — Your Commit Strategy Determines Semantics

```
Offset = your position in a partition. "I've processed up to message 42."

                  Message 38  39  40  41  42  43  44  45
                  [████████████████████████│─────────────]
                                           ↑
                                    committed offset
                                    (consumer will resume here after restart)
```

```java
// AUTO-COMMIT (default, risky)
// props.put("enable.auto.commit", "true");   // commits every 5 seconds
// Problem: offset committed BEFORE processing → if processing fails, message lost
//          offset committed AFTER processing → if crash before commit, message reprocessed

// MANUAL COMMIT (recommended)
// props.put("enable.auto.commit", "false");
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record);                    // process first
    }
    consumer.commitSync();                  // then commit → at-least-once
}
// If crash after process but before commit → message reprocessed (at-least-once)
// Make your processing IDEMPOTENT to handle this safely
```

### 6. Delivery Semantics

| Semantic | How | Trade-off |
|----------|-----|-----------|
| **At-most-once** | Commit offset BEFORE processing | May lose messages (fast) |
| **At-least-once** | Process THEN commit | May reprocess (safe with idempotent consumer) |
| **Exactly-once** | Transactional API | Highest guarantee, highest overhead |

```java
// EXACTLY-ONCE: Three layers working together
// 1. Idempotent Producer (enable.idempotence=true)
//    Broker uses ProducerID + sequence number → deduplicates retries
//    Prevents: duplicate messages from producer retries

// 2. Transactional API
//    Atomic writes across multiple partitions + offset commit
props.put("transactional.id", "payment-processor-1");
producer.initTransactions();
producer.beginTransaction();
producer.send(new ProducerRecord<>("output-topic", key, value));
producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);
producer.commitTransaction();   // all-or-nothing

// 3. Consumer isolation.level=read_committed
//    Only reads messages from committed transactions
```

### 7. Replication & Durability (ISR)

```
Partition 0:
  Broker 1 [LEADER]   ← Producers write here
  Broker 2 [FOLLOWER] ← Replicates from leader (ISR)
  Broker 3 [FOLLOWER] ← Replicates from leader (ISR)

ISR = In-Sync Replicas = replicas caught up with leader
```

```java
// Producer acks — durability vs latency trade-off
props.put("acks", "0");    // fire-and-forget — fastest, may lose data
props.put("acks", "1");    // leader acknowledged — leader crash = data loss
props.put("acks", "all");  // ALL ISR acknowledged — strongest durability

// Broker-side: min.insync.replicas=2
// With acks=all + min.insync.replicas=2:
// → Write succeeds only if at least 2 replicas (including leader) acknowledge
// → Even if one broker dies, data is safe on another
// → If fewer than 2 ISR available → producer gets NotEnoughReplicasException
```

### 8. Log Compaction vs Retention

```
RETENTION (cleanup.policy=delete):
  Messages deleted after time (retention.ms) or size (retention.bytes)
  Default: 7 days

COMPACTION (cleanup.policy=compact):
  Keeps ONLY the latest value per key
  Old values for same key are deleted
  Tombstone (null value) → key is eventually removed

  Before compaction:        After compaction:
  K1:v1, K2:v1, K1:v2     K2:v1, K1:v2
  K1:v3, K2:v2             K1:v3, K2:v2

Use compaction for:
- KTables (Kafka Streams state)
- CDC snapshots (latest state of each DB row)
- Configuration topics
```

### 9. Spring Kafka — Producer & Consumer

```java
// PRODUCER
@Service
public class PaymentProducer {
    private final KafkaTemplate<String, PaymentEvent> kafka;
    
    public CompletableFuture<SendResult<String, PaymentEvent>> send(PaymentEvent event) {
        return kafka.send("payments", event.getCustomerId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) log.error("Send failed", ex);
                else log.info("Sent to partition {} offset {}",
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            });
    }
}

// CONSUMER
@Service
public class PaymentConsumer {
    
    @KafkaListener(topics = "payments", groupId = "payment-processor")
    public void handle(
            @Payload PaymentEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {
        log.info("Processing {} from partition {} offset {}", event, partition, offset);
        paymentService.process(event);
    }
}

// ERROR HANDLING — retry + Dead Letter Topic
@Bean
public ConcurrentKafkaListenerContainerFactory<String, PaymentEvent> kafkaListenerFactory(
        ConsumerFactory<String, PaymentEvent> cf, KafkaTemplate<String, PaymentEvent> template) {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, PaymentEvent>();
    factory.setConsumerFactory(cf);
    factory.setCommonErrorHandler(new DefaultErrorHandler(
        new DeadLetterPublishingRecoverer(template),   // send to DLT after retries exhausted
        new FixedBackOff(1000, 3)                      // retry 3 times, 1s apart
    ));
    return factory;
}
```

### 10. Dead Letter Topic (DLT)

```
Normal flow:                    Error flow:
payments ──→ Consumer           payments ──→ Consumer ──→ FAIL
                                                │ retry 1 ──→ FAIL
                                                │ retry 2 ──→ FAIL
                                                │ retry 3 ──→ FAIL
                                                └──→ payments.DLT (dead letter topic)
                                                      │
                                                      └──→ Investigate, fix, replay
```

**Use DLT for:**
- Poison pill messages (bad format, validation failure)
- Messages that consistently fail processing
- Allows the main consumer to continue instead of blocking on one bad message

---

## Rapid-Fire Q&A

### Q1: What is Kafka and when do you use it?
**A:** Distributed event streaming platform. Use for: decoupling services (event-driven architecture), high-throughput data pipelines, real-time processing, audit logs, change data capture. Not a traditional message queue — it's an immutable, append-only log.

### Q2: Kafka architecture — key components?
**A:** **Broker**: Kafka server. **Topic**: named category of messages. **Partition**: ordered, immutable sequence within a topic — unit of parallelism. **Producer**: writes to topics. **Consumer**: reads from topics. **Consumer Group**: set of consumers sharing work. **ZooKeeper/KRaft**: metadata management (KRaft replaces ZooKeeper in Kafka 4.0+).

### Q3: What are partitions and why do they matter?
**A:** Each topic is split into partitions. Partitions enable: (1) parallelism — multiple consumers read different partitions simultaneously. (2) ordering — guaranteed ONLY within a partition, not across. (3) scalability — partitions distributed across brokers.

### Q4: How does partition key work?
**A:** Producer specifies a key. Kafka hashes the key → determines target partition. All messages with the same key go to the same partition → guaranteed ordering for that key. Without a key: round-robin (or sticky partitioning in newer versions). E.g., key = `customerId` ensures all events for a customer are ordered.

### Q5: Consumer groups — how do they work?
**A:** Each partition is assigned to exactly one consumer within a group. If 6 partitions and 3 consumers → each gets 2 partitions. If 6 partitions and 8 consumers → 2 consumers idle. More consumers than partitions = waste. Multiple groups can independently consume the same topic (fan-out).

### Q6: What's a rebalance?
**A:** When group membership changes (consumer joins/leaves/crashes) or partition count changes, Kafka reassigns partitions. During rebalance, consumption pauses. Minimise with: `CooperativeStickyAssignor` (incremental rebalance) and static membership (`group.instance.id`).

### Q7: How does Kafka achieve exactly-once semantics?
**A:** Three layers: (1) **Idempotent producer** (`enable.idempotence=true`): broker uses Producer ID + sequence number to deduplicate. (2) **Transactional API**: atomic writes across multiple partitions + offset commit. (3) **Consumer `isolation.level=read_committed`**: only reads committed messages.

### Q8: At-least-once vs at-most-once vs exactly-once?
**A:** **At-most-once**: commit offset before processing — if processing fails, message lost. **At-least-once**: process then commit — if commit fails, message reprocessed (most common, with idempotent consumer). **Exactly-once**: transactional producer + consumer — highest guarantee, highest overhead.

### Q9: What's a Dead Letter Topic (DLT)?
**A:** When a consumer can't process a message (poison pill, validation failure), it writes it to a DLT instead of retrying infinitely. Allows the main consumer to continue. DLT messages are investigated and replayed later. Spring Kafka supports this via `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`.

### Q10: What's offset management?
**A:** Offset = position in a partition. Consumers commit offsets to tell Kafka "I've processed up to here." `auto.commit=true` (default): commits periodically (risky — may commit before processing). `auto.commit=false` + manual commit: process then commit (at-least-once). Stored in internal `__consumer_offsets` topic.

### Q11: Replication and ISR?
**A:** Each partition has a leader and N-1 replicas. Producers write to leader. Replicas pull from leader. **ISR (In-Sync Replicas)**: replicas caught up with leader. `min.insync.replicas=2` + `acks=all` ensures a write is durable on at least 2 brokers before acknowledging.

### Q12: Log compaction vs retention?
**A:** **Retention** (`cleanup.policy=delete`): messages deleted after time or size threshold. **Compaction** (`cleanup.policy=compact`): keeps only the latest value per key — tombstone (null value) deletes. Use compaction for: KTables, CDC snapshots, config topics.

### Q13: Spring Kafka — key components?
**A:** `KafkaTemplate` — producer (`.send(topic, key, value)`). `@KafkaListener` — consumer (`@KafkaListener(topics = "orders", groupId = "order-service")`). `ProducerFactory` / `ConsumerFactory` — configure serialisers, group ID, acks. `ConcurrentKafkaListenerContainerFactory` — controls concurrency.

### Q14: How do you handle failures in a Kafka consumer?
**A:** `DefaultErrorHandler` with `FixedBackOff(1000, 3)` — retry 3 times, 1s apart. After retries exhausted → `DeadLetterPublishingRecoverer` sends to DLT. For transient errors (DB down): use `ExponentialBackOff`. For permanent errors (bad data): send to DLT immediately.

---

## Key Concepts Diagram

```
Producer → [Topic: "payments"]
            ├── Partition 0 → Consumer A (Group: payment-svc)
            ├── Partition 1 → Consumer B (Group: payment-svc)
            └── Partition 2 → Consumer C (Group: payment-svc)
            
            ├── Partition 0 → Consumer X (Group: audit-svc)  ← independent group
            ├── Partition 1 → Consumer X (Group: audit-svc)  
            └── Partition 2 → Consumer Y (Group: audit-svc)
```

---

## Can you answer these cold?

- [ ] Kafka vs traditional MQ — append-only log, consumer groups, replay
- [ ] Partitions — parallelism, ordering guarantee (per-partition only)
- [ ] Partition key — how it determines ordering
- [ ] Consumer group rebalancing — when it happens, how to minimise
- [ ] Exactly-once — idempotent producer + transactional API + read_committed
- [ ] At-least-once vs at-most-once — offset commit timing
- [ ] Dead Letter Topic — when and how
- [ ] ISR + `acks=all` + `min.insync.replicas` — durability guarantee
- [ ] Spring Kafka — `KafkaTemplate`, `@KafkaListener`, error handling

[← Back to Index](./00_INDEX.md)
