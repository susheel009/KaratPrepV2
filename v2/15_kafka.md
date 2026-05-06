# 15 — Kafka Fundamentals

> [← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/15_kafka_doubts.md) · [← Prev: 14 Spring AOP & Data](./14_spring_aop_data.md) · [Next: 16 Debugging →](./16_debugging.md)
>
> **Priority:** 🟡 High · **Related topics:** [12 Spring Boot](./12_spring_boot.md) · [01 Concurrency (consumers)](./01_concurrency.md) · [16 Debugging](./16_debugging.md)

## Contents

1. [The Problem](#1-the-problem-story-style-no-code) — services that don't have to be on the phone at once
2. [Walkthrough](#2-walkthrough-concept--tiny-example-pseudo-code-only) — one event, three independent readers
3. [First Code](#3-first-code-minimal-every-non-obvious-line-commented) — Spring Kafka producer + consumer
4. [Build Up — Practical Patterns](#4-build-up--practical-patterns)
   - [4.1 Kafka vs traditional MQ](#41-kafka-vs-traditional-message-queues)
   - [4.2 Architecture](#42-architecture--brokers-topics-partitions)
   - [4.3 Partitions, ordering, partition keys](#43-partitions-ordering-partition-keys)
   - [4.4 Consumer groups](#44-consumer-groups--scaling-and-rebalance)
   - [4.5 Offsets](#45-offsets-and-commit-strategy)
   - [4.6 Spring Kafka basics](#46-spring-kafka--basic-producer-and-consumer)
   - [4.7 Dead Letter Topic](#47-dead-letter-topic-dlt)
5. [Going Deep — Interview-Level Material](#5-going-deep--interview-level-material)
   - [5.1 Delivery semantics](#51-delivery-semantics-at-most-once-at-least-once-exactly-once)
   - [5.2 Replication, ISR, acks](#52-replication-isr-and-acks)
   - [5.3 Log compaction vs retention](#53-log-compaction-vs-retention)
   - [5.4 Rebalancing strategies](#54-rebalancing-strategies)
   - [5.5 Idempotent + transactional producer](#55-idempotent-and-transactional-producer)
   - [5.6 Common pitfalls](#56-common-pitfalls)
6. [Memory Aids](#6-memory-aids)
7. [Cheat Sheet — Rapid-Fire Q&A](#7-cheat-sheet--rapid-fire-qa)
8. [Self-Test](#8-self-test)
9. [Glossary (in plain English)](#9-glossary-in-plain-english)

> **15 minutes before the interview?** Skip to [§7 Cheat Sheet](#7-cheat-sheet--rapid-fire-qa) and [§6 Memory Aids](#6-memory-aids).

---

## 1. The Problem (story-style, no code)

You run an e-commerce backend. The order service handles checkouts. When an order is placed, three other services need to know:

- **Payments** charges the card.
- **Shipping** prints the label.
- **Analytics** updates the dashboard.

The naive design: when an order is placed, the order service calls payments, then shipping, then analytics. Synchronously. Three problems show up immediately:

1. **One service down → the whole flow breaks.** If analytics is degraded, checkouts fail.
2. **Adding a new consumer means modifying the producer.** A new "fraud detection" service? Edit order-service, add another HTTP call, redeploy.
3. **Spiky load saturates everyone simultaneously.** Black Friday traffic hits all four services at once.

Kafka is the **post office** that fixes all three. Order service drops a message into a mailbox (a *topic*). The post office (Kafka) **stores it durably**. Each interested service (a *consumer*) picks up messages at its own pace. Adding a fifth service tomorrow? Subscribe to the same topic; nothing changes for the producer. Slow service? Kafka holds the queue, no backpressure on order service.

That's the simple picture. Two refinements make Kafka different from a classical message queue:

- **Messages are not deleted on consumption** — they're written to an append-only log, kept for some retention window (default 7 days). Consumers track *where they are* (offset). A new consumer can replay from the beginning. A buggy consumer can rewind and replay after a fix.
- **Topics are split into partitions** for horizontal scale and parallel processing. Each partition is an independent ordered log; Kafka guarantees order *within* a partition, not across.

> Once you see Kafka as "an append-only log per topic, partitioned for parallelism, with consumers reading at their own pace by tracking offsets", every bit of jargon — broker, ISR, consumer group, rebalance, exactly-once, DLT — is a refinement of one of those ideas.

---

## 2. Walkthrough (concept + tiny example, pseudo-code only)

`order-service` produces `OrderPlaced(orderId="O1", custId="C7")` to the topic `orders`.

The topic has **3 partitions**. Two independent **consumer groups** subscribe: `payments-group` (3 consumers) and `analytics-group` (1 consumer).

| Step | What |
|---|---|
| 1 | Producer asks Kafka: "what partition for key C7?" Kafka hashes "C7" mod 3 → partition 1. |
| 2 | Producer sends to leader of partition 1 (say broker 2). Broker 2 appends; offset = 47 (whatever the next offset was). |
| 3 | Broker 2 replicates to followers (brokers 1, 3). Once replicated to enough followers, broker 2 acks. |
| 4 | `payments-group`: each partition is owned by exactly one consumer in the group. Consumer that owns partition 1 reads offset 47. **3 partitions / 3 consumers = each owns one.** |
| 5 | `analytics-group`: 1 consumer owns all 3 partitions. It reads from partition 1 along with partitions 0 and 2. |
| 6 | Both groups process the message **independently** — `payments` doesn't know `analytics` exists. |
| 7 | Each consumer **commits its offset** after processing. If a consumer crashes, it resumes from its last committed offset on restart. |

What this walkthrough makes obvious:

1. **Partition key picks the partition.** Same key → same partition → same order → same consumer.
2. **A consumer group divides the work.** N partitions, K consumers in the group → each owns N/K partitions.
3. **Multiple groups read independently.** Same data, different consumers, no interference.
4. **Offsets are per-group.** `payments-group`'s offset is independent of `analytics-group`'s offset.

Now the actual code.

---

## 3. First Code (minimal, every non-obvious line commented)

```java
// === Producer ===
@Service
public class OrderProducer {
    private final KafkaTemplate<String, OrderEvent> kafka;
    public OrderProducer(KafkaTemplate<String, OrderEvent> kafka) { this.kafka = kafka; }

    public CompletableFuture<SendResult<String, OrderEvent>> publish(OrderEvent e) {
        // key = customerId so all events for a customer go to the same partition (preserves order)
        return kafka.send("orders", e.customerId(), e)
            .whenComplete((res, ex) -> {
                if (ex != null) log.error("send failed", ex);
                else log.info("sent partition={} offset={}", res.getRecordMetadata().partition(),
                                                              res.getRecordMetadata().offset());
            });
    }
}

// === Consumer ===
@Service
public class OrderConsumer {

    @KafkaListener(topics = "orders", groupId = "payments")
    public void handle(@Payload OrderEvent e,
                       @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                       @Header(KafkaHeaders.OFFSET) long offset) {
        log.info("processing partition={} offset={} order={}", partition, offset, e.orderId());
        paymentService.charge(e);
        // On normal return → Spring auto-commits the offset (default).
        // If processing throws → Spring's error handler kicks in (retry / DLT — §4.7).
    }
}

// === Configuration (application.yml) ===
// spring:
//   kafka:
//     bootstrap-servers: localhost:9092
//     consumer:
//       group-id: payments
//       auto-offset-reset: earliest      # for new groups, read from beginning of retained log
//       enable-auto-commit: false        # commit explicitly via container — safer
//       isolation-level: read_committed  # for exactly-once
//     producer:
//       key-serializer: org.apache.kafka.common.serialization.StringSerializer
//       value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
//       acks: all                         # strongest durability
//       enable-idempotence: true          # exactly-once producer
```

What just happened: a producer publishing keyed events to a topic, a consumer reading them by group, with explicit offset commit and JSON value-serialization. About 30 lines of declared code; everything else (partition assignment, offset commit, retry, DLT) is configurable on the listener container.

---

## 4. Build Up — Practical Patterns

### 4.1 Kafka vs traditional message queues

```
Traditional MQ (RabbitMQ, ActiveMQ):                  Kafka:
- Message consumed → DELETED                          - Message consumed → STAYS in the log
- Push-based (broker pushes to consumer)              - Pull-based (consumer reads at its own pace)
- Single consumer per message (or fan-out via xchg)   - Multiple consumer groups read independently
- No replay                                            - Replay by resetting offset
- Per-message routing semantics                        - Per-partition append-only log
```

Kafka is **a distributed append-only commit log**. Consumers track position via offset; data lives for the retention period regardless of consumption.

**When to use Kafka:**
- High-throughput data pipelines (millions of messages/sec).
- Decoupling services (event-driven architecture).
- Event sourcing / audit trail (replay everything to rebuild state).
- Real-time stream processing (Kafka Streams, Flink, Spark).
- Change Data Capture (Debezium → Kafka → consumers).

**When *not* to use Kafka:** you need request/reply (use HTTP/gRPC). Low message rates and you don't need persistence (use a queue). You need per-message TTL or priority (Kafka has neither natively).

### 4.2 Architecture — brokers, topics, partitions

```
                                 Kafka Cluster (3 brokers)
                       ┌─────────────────────────────────────────────┐
Producer ─────→  Topic: "orders"                                       │
                 ┌──────────────────────────────────────────────────┐  │
                 │ Partition 0 [Leader: broker 1] [Followers: 2, 3] │  │  → Consumer Group "payments"
                 │   offsets   0  1  2  3  4  5  6  7  8           │  │      ├─ consumer A → P0
                 ├──────────────────────────────────────────────────┤  │      ├─ consumer B → P1
                 │ Partition 1 [Leader: broker 2] [Followers: 1, 3] │  │      └─ consumer C → P2
                 │   offsets   0  1  2  3  4  5                     │  │
                 ├──────────────────────────────────────────────────┤  │  → Consumer Group "analytics"
                 │ Partition 2 [Leader: broker 3] [Followers: 1, 2] │  │      └─ consumer X → P0, P1, P2
                 │   offsets   0  1  2  3  4  5  6  7              │  │
                 └──────────────────────────────────────────────────┘  │
                       └─────────────────────────────────────────────┘
```

| Component | What it is |
|---|---|
| **Broker** | A Kafka server process. A cluster has multiple brokers. |
| **Topic** | Named category (like a SQL table — but append-only). |
| **Partition** | An ordered, immutable, append-only sub-log within a topic. Unit of parallelism. |
| **Offset** | Monotonically increasing position of a message within a partition. |
| **Producer** | Publishes records to topics. |
| **Consumer** | Reads records from topics. |
| **Consumer Group** | Set of consumers sharing the work of a topic. |
| **Leader** | The broker that handles all reads/writes for a partition. |
| **Follower** | Replicas that pull from the leader. |
| **ISR (In-Sync Replicas)** | Replicas that are caught up with the leader. |
| **KRaft** | Replaces ZooKeeper for metadata management (Kafka 4.0+). |

### 4.3 Partitions, ordering, partition keys

> Ordering is guaranteed **only within a partition**, not across partitions.

```java
// Producer chooses the partition via the key
producer.send(new ProducerRecord<>("payments", "C001", paymentEvent));
//                                  topic       key     value
// Kafka: hash("C001") % numPartitions → always the same partition

// All events for C001 go to one partition → ordered there.
// Events for different customers may end up in different partitions → no ordering between customers (which is fine).

// NO KEY → producer uses sticky partitioning (recent versions) — batches into the same partition for a while, then moves on
producer.send(new ProducerRecord<>("logs", null, logEntry));
```

**Why this design?** Partitions are how Kafka scales. The producer fans out by key; consumers fan in by partition. If you need *global* ordering across all events, you need 1 partition — and you give up parallelism.

**How to choose a partition key:**
- Choose by the entity whose order matters: `customerId`, `accountId`, `tradeId`, `orderId`.
- Don't choose a key with hot-spots (e.g. `country` if 95 % is "US" — one partition gets all the traffic).
- Don't choose a high-cardinality random key if you need ordering — the events for one entity will scatter.

### 4.4 Consumer groups — scaling and rebalance

```
Consumer Group "payments" has 3 consumers; topic "orders" has 6 partitions:
  Consumer A → P0, P1
  Consumer B → P2, P3
  Consumer C → P4, P5

Rules
✅ Each partition is assigned to EXACTLY ONE consumer within a group.
✅ One consumer can handle MULTIPLE partitions.
❌ More consumers than partitions → extra consumers sit idle.
✅ Multiple groups read the same topic independently (fan-out by group).
```

**Rebalance** — when group membership changes (consumer joins, leaves, crashes) or partition count changes, Kafka reassigns partitions across the live consumers. **During rebalance, consumption pauses.**

Rebalance triggers:
- Consumer joining or leaving the group.
- Heartbeat timeout (`session.timeout.ms`) — Kafka thinks the consumer is dead.
- Adding partitions to the topic.
- Subscription changes.

Minimise rebalance disruption:

```yaml
spring:
  kafka:
    consumer:
      properties:
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        # Cooperative rebalance: only the affected partitions move; others keep consuming.
        # Pre-Kafka 2.4 behaviour was "stop the world" — every consumer paused.
        group.instance.id: payments-pod-${HOSTNAME}
        # Static membership: consumer keeps its partitions across short outages (rolling restarts).
        session.timeout.ms: 45000
        heartbeat.interval.ms: 15000
```

### 4.5 Offsets and commit strategy

The offset is **your bookmark** in the partition: "I've processed up to message 42." Kafka stores per-group offsets in an internal `__consumer_offsets` topic.

```
Partition messages:    36  37  38  39  40  41  42  43  44  45
                        █   █   █   █   █   █   ┃    ─   ─   ─
                                              ↑
                                    committed offset = 43
                                    (consumer resumes here on restart)
```

**Commit strategies, simplest first:**

```yaml
# enable.auto.commit=true (default, RISKY)
# Commits every auto.commit.interval.ms (default 5s).
# Race: offset committed BEFORE you finish processing → crash = message lost.
#       Or: offset committed AFTER processing → crash mid-batch = some messages reprocessed.

# enable.auto.commit=false + manual commit (RECOMMENDED)
spring:
  kafka:
    consumer:
      enable-auto-commit: false
    listener:
      ack-mode: RECORD          # commit after each record processed (or BATCH, MANUAL, etc.)
```

| `ack-mode` | When the offset is committed |
|---|---|
| `RECORD` | After each record is processed |
| `BATCH` (default) | After all records in a poll batch are processed |
| `TIME` | After a configured time elapses since the last commit |
| `COUNT` | After N records processed |
| `COUNT_TIME` | Whichever of COUNT or TIME comes first |
| `MANUAL` | You call `Acknowledgment.acknowledge()` |
| `MANUAL_IMMEDIATE` | Same, committed synchronously when you call it |

### 4.6 Spring Kafka — basic producer and consumer

```java
// Producer
@Service
public class OrderProducer {
    private final KafkaTemplate<String, OrderEvent> kafka;
    public OrderProducer(KafkaTemplate<String, OrderEvent> kafka) { this.kafka = kafka; }
    public CompletableFuture<SendResult<String, OrderEvent>> send(OrderEvent e) {
        return kafka.send("orders", e.customerId(), e);
    }
}

// Consumer
@Service
public class PaymentConsumer {
    @KafkaListener(topics = "orders", groupId = "payments")
    public void handle(@Payload OrderEvent e,
                       @Header(KafkaHeaders.RECEIVED_PARTITION) int p,
                       @Header(KafkaHeaders.OFFSET) long offset) {
        paymentService.charge(e);
        // On normal return, the listener container commits per ack-mode.
    }
}

// Concurrency: how many consumer threads in this group on this instance
@KafkaListener(topics = "orders", groupId = "payments", concurrency = "3")
// → 3 internal threads; each owns a subset of the assigned partitions on this instance.
```

### 4.7 Dead Letter Topic (DLT)

```
Normal flow                                Error flow (retry exhausted)
orders ──▶ Consumer                        orders ──▶ Consumer ─FAIL→ retry × 3 ─FAIL→ orders.DLT
                                                                                              │
                                                                                  investigate, fix, replay
```

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> kafkaFactory(
        ConsumerFactory<String, OrderEvent> cf,
        KafkaTemplate<String, OrderEvent> template) {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, OrderEvent>();
    factory.setConsumerFactory(cf);
    factory.setCommonErrorHandler(new DefaultErrorHandler(
        new DeadLetterPublishingRecoverer(template),       // → orders.DLT after retries
        new FixedBackOff(1000L, 3)                          // 3 retries, 1s apart
    ));
    return factory;
}
```

**Use DLT for:**
- Poison-pill messages (deserialisation failure, validation rejection).
- Messages that consistently fail processing (downstream API permanently broken for this record).
- So the main consumer keeps draining the queue instead of stalling on one bad record.

The DLT is itself a Kafka topic — operators can read it, fix the underlying issue, and replay.

---

## 5. Going Deep — Interview-Level Material

### 5.1 Delivery semantics: at-most-once, at-least-once, exactly-once

```
"Send a message" with a network in the middle → some messages may arrive 0, 1, or >1 times.
The semantics name promises about that count.
```

| Semantic | How | Trade-off |
|---|---|---|
| **At-most-once** | Commit offset BEFORE processing | Fast, but may LOSE messages on crash |
| **At-least-once** | Process THEN commit offset | May REPROCESS on crash → consumer must be idempotent |
| **Exactly-once** | Idempotent producer + transactional API + consumer `read_committed` | Highest guarantee, highest overhead |

**At-least-once** is the default and the right choice for almost every service: process the message, then commit the offset. If the consumer crashes after processing but before committing, the same message will be redelivered; **make your processing idempotent** (check whether you've already done the work — usually a uniqueness constraint or an `idempotency_key` table).

### 5.2 Replication, ISR, and acks

```
Partition 0 layout:
  Broker 1 [LEADER]   ← producers write here
  Broker 2 [FOLLOWER] ← pulls from leader
  Broker 3 [FOLLOWER] ← pulls from leader

ISR = In-Sync Replicas = replicas (including leader) that are caught up.
```

```yaml
spring:
  kafka:
    producer:
      acks: all                     # wait for ALL ISRs to acknowledge
      retries: 5
      properties:
        enable.idempotence: true    # producer-side dedup of retries
        max.in.flight.requests.per.connection: 5  # ≤5 with idempotence
# Broker side:
#   min.insync.replicas: 2         # must have at least 2 in-sync replicas
```

`acks` values:

| `acks` | Producer waits for | Latency | Durability |
|---|---|---|---|
| `0` | Nothing (fire and forget) | Lowest | Lowest — may lose data on broker crash |
| `1` | Leader to write to its log | Medium | Leader crash before replication = data loss |
| `all` (`-1`) | All ISRs to acknowledge | Highest | Strongest — survives leader crash if `min.insync.replicas ≥ 2` |

**Production recipe:** `acks=all`, `min.insync.replicas=2`, `replication.factor=3`. Survives one broker death; producer gets `NotEnoughReplicasException` if more than one ISR is unavailable (so the application learns it can't write durably and either retries or surfaces a 503).

### 5.3 Log compaction vs retention

```
Retention (cleanup.policy=delete) — DEFAULT
  Messages deleted after retention.ms (default 7 days) or retention.bytes.

Compaction (cleanup.policy=compact)
  Keeps only the LATEST value per key.
  Old values for the same key are deleted by a background process.
  Tombstone (key with null value) → eventually removes the key entirely.

  Stream of messages (key:value):              After compaction (eventual):
  K1:v1, K2:v1, K1:v2, K2:v2, K1:v3            K2:v2, K1:v3
```

**When to use each:**

| Use | Policy |
|---|---|
| Audit log, event stream, time-bounded data | `delete` (retention) |
| Latest state per entity (KTables, configuration topics, CDC snapshots) | `compact` |
| Both — keep recent history AND latest per key | `compact,delete` |

### 5.4 Rebalancing strategies

| Strategy | What it does |
|---|---|
| `RangeAssignor` (legacy default) | Assigns ranges per topic. Can imbalance with multiple topics. |
| `RoundRobinAssignor` | Distributes partitions round-robin. More balanced. |
| `StickyAssignor` | Like round-robin, but tries to keep existing assignments stable across rebalances. |
| `CooperativeStickyAssignor` | **Modern default.** Cooperative rebalance protocol — only affected partitions are reassigned; others keep consuming during rebalance. |

For long-running stateful consumers (with caches, in-flight DB transactions), `CooperativeStickyAssignor` + `group.instance.id` (static membership) drastically reduces churn during rolling restarts.

### 5.5 Idempotent and transactional producer

**Idempotent producer** — `enable.idempotence=true`. The broker assigns each producer a `ProducerId`, and the producer attaches a sequence number per partition. The broker rejects duplicate sequence numbers, so retries no longer cause duplicates *on the producer side*.

**Transactional producer** — assigns a stable `transactional.id`. Lets you write to multiple partitions and commit *atomically*: all writes succeed or none are visible.

```java
producer.initTransactions();
producer.beginTransaction();
producer.send(new ProducerRecord<>("output", key, value));
producer.send(new ProducerRecord<>("output-derived", key, derived));
// Commit consumer's offsets within the same transaction:
producer.sendOffsetsToTransaction(offsets, consumerGroupMetadata);
producer.commitTransaction();   // all-or-nothing
```

Combined with consumer `isolation.level=read_committed`, this gives **exactly-once** semantics across a "consume → process → produce → commit" pipeline (the canonical Kafka Streams pattern).

### 5.6 Common pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| Wrong partition key | Hot partition; rebalance starves; ordering breaks | Choose a key with even distribution and the right ordering boundary |
| `enable.auto.commit=true` + slow processing | Lost messages on crash | `enable.auto.commit=false`, manual ack |
| Long-running listener with default session timeout | Repeated rebalances ("zombie consumer") | Tune `max.poll.interval.ms`; or split work to a thread pool |
| Many consumers in one group, few partitions | Idle consumers | Match consumer count to partitions |
| Producer `acks=1` in production | Data loss on broker failover | `acks=all` + `min.insync.replicas=2` |
| Missing DLT | Poison pill blocks the entire partition | `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` |
| Consumer not idempotent | Duplicates leak through (at-least-once) | Idempotency key / dedup table / upserts |
| Schema drift with no compatibility check | Consumers fail on new field | Schema Registry (Avro/Protobuf/JSON Schema) with compatibility rules |

---

## 6. Memory Aids

### Decision tree: "ordering vs throughput vs durability"

```
Need global ordering across the whole topic?
└── Yes → 1 partition. You give up parallelism.

Need ordering per entity (per customer, per account)?
└── Yes → key on that entity. Multiple partitions, ordered within each.

No ordering needed?
└── No key (or random) → max throughput, sticky partitioning.

Durability requirement?
├── Critical (payments, banking) → acks=all + min.insync.replicas=2 + replication.factor=3.
├── Best-effort metrics?         → acks=1.
└── Fire-and-forget logs?        → acks=0 (rarely justified).
```

### "If they ask X, first think Y"

| If they ask… | First think… | Then say… |
|--------------|--------------|-----------|
| "Kafka vs RabbitMQ?" | Append-only log vs delete-on-consume | "Kafka keeps messages for retention; consumers track position by offset; multiple groups consume independently." |
| "What's a partition?" | Unit of parallelism + ordering boundary | "Ordered append-only log inside a topic. Ordering is per-partition only." |
| "Partition key?" | Determines partition; preserves order | "Hash(key) mod numPartitions. Same key → same partition → ordered there." |
| "Consumer group?" | Sharing work across consumers | "Each partition assigned to exactly one consumer in the group." |
| "Exactly-once?" | Three layers | "Idempotent producer + transactional API + consumer read_committed." |
| "ISR?" | Replicas caught up with leader | "acks=all + min.insync.replicas=2 → write durable on at least 2 brokers." |
| "DLT?" | Poison-pill bin | "After retries exhausted, message goes to <topic>.DLT for manual investigation." |
| "Rebalance?" | Membership change → reassign partitions | "Cooperative sticky assignor + static membership minimises disruption." |

### Three anchor pictures

1. **Append-only log.** Messages are not deleted on consumption.
2. **Per-partition order.** Same key → same partition → guaranteed order.
3. **Consumer groups divide partitions.** Multiple groups read the same topic independently.

---

## 7. Cheat Sheet — Rapid-Fire Q&A

### Q1: What is Kafka and when do you use it?
**A:** Distributed event streaming platform — append-only commit logs partitioned across brokers. Used for: decoupling services (event-driven), high-throughput data pipelines, real-time stream processing, audit trails, change data capture. Not for request/reply.

### Q2: Kafka architecture — key components?
**A:** **Broker** (Kafka server), **Topic** (named log), **Partition** (ordered subset, unit of parallelism), **Producer** (writes), **Consumer** (reads), **Consumer Group** (shares work), **Leader/Follower** (per-partition replication), **ISR** (in-sync replicas), **KRaft** (replaces ZooKeeper, Kafka 4.0+).

### Q3: What are partitions and why do they matter?
**A:** Each topic is split into N partitions. They enable: parallelism (consumers read different partitions concurrently), ordering (guaranteed only within a partition), scalability (partitions distributed across brokers).

### Q4: How does a partition key work?
**A:** Producer specifies a key. Kafka hashes the key mod numPartitions → target partition. Same key always → same partition → guaranteed ordering for that key. Without a key: sticky partitioning (recent versions) or round-robin.

### Q5: Consumer groups — how do they work?
**A:** Each partition is assigned to exactly one consumer in the group. 6 partitions, 3 consumers → each gets 2. 6 partitions, 8 consumers → 2 sit idle. Different groups read the same topic independently.

### Q6: What's a rebalance and how do you minimise it?
**A:** When group membership changes, Kafka reassigns partitions. Consumption pauses during rebalance. Minimise: `CooperativeStickyAssignor` (incremental rebalance — only affected partitions move) and `group.instance.id` (static membership — survives rolling restarts).

### Q7: At-most-once, at-least-once, exactly-once?
**A:** **At-most-once**: commit offset before processing. Fast; may lose. **At-least-once**: process then commit. Default; may reprocess (consumer must be idempotent). **Exactly-once**: idempotent producer + transactional API + consumer `read_committed`. Highest guarantee.

### Q8: How does Kafka achieve exactly-once?
**A:** Three layers: idempotent producer (`enable.idempotence=true` — broker dedups retries by ProducerId+sequence), transactional API (atomic writes across partitions + offset commit), consumer `isolation.level=read_committed` (only sees committed transactions).

### Q9: Replication, ISR, acks?
**A:** Each partition has a leader and N-1 replicas. Replicas pull from leader. **ISR**: replicas caught up with leader. **acks=all** + **min.insync.replicas=2** + **replication.factor=3** is the durable production recipe — survives one broker death, refuses writes if fewer than 2 ISRs available.

### Q10: What's offset management?
**A:** Offset = consumer's bookmark per partition. `auto.commit=true` (default) commits periodically — risky. Manual commit (`auto.commit=false` + container ack-mode) gives at-least-once with control. Stored in internal `__consumer_offsets` topic, per group.

### Q11: Log compaction vs retention?
**A:** Retention (`delete`): messages removed after time/size. Compaction (`compact`): keeps only the latest value per key (tombstones eventually delete a key). Use compaction for KTables, configuration topics, CDC snapshots.

### Q12: Spring Kafka — key components?
**A:** `KafkaTemplate` — producer (`.send(topic, key, value)`). `@KafkaListener` — consumer (`@KafkaListener(topics="orders", groupId="payments")`). `ProducerFactory` / `ConsumerFactory` — configure serialisers, security. `ConcurrentKafkaListenerContainerFactory` — controls concurrency, error handling.

### Q13: How do you handle failures in a Kafka consumer?
**A:** `DefaultErrorHandler` with `FixedBackOff(1000L, 3)` (3 retries 1s apart) or `ExponentialBackOff` for transient failures. After retries exhausted, `DeadLetterPublishingRecoverer` sends to `<topic>.DLT`. Send poison pills (deserialisation failures) directly to DLT — don't retry.

### Q14: Why is the partition key choice critical?
**A:** Determines load balance (skewed key = hot partition), ordering boundary (order is preserved per key), and consumer locality. Bad key → one consumer overloaded while others idle. Choose by the entity whose ordering matters (customerId, orderId).

### Q15: What's a Dead Letter Topic and why use it?
**A:** A separate topic where messages that consistently fail processing are sent after retries are exhausted. Lets the main consumer keep draining the queue instead of blocking on a single bad message. Operators read DLT, diagnose, fix, replay.

### Q16: Kafka vs traditional MQ?
**A:** Kafka — append-only log, pull-based, multiple consumer groups read independently, replay by offset reset, retention period. MQ (RabbitMQ, ActiveMQ) — message deleted on consume, push-based, no built-in replay. Kafka is for high-throughput streaming; MQ is for routing-heavy reliable point-to-point messaging.

---

### Key Code Patterns

**Producer with key**
```java
kafka.send("orders", customerId, event);   // customerId picks the partition → preserves order per customer
```

**Consumer with manual ack**
```java
@KafkaListener(topics = "orders", groupId = "payments")
public void handle(OrderEvent e, Acknowledgment ack) {
    paymentService.charge(e);
    ack.acknowledge();          // commit only after successful processing
}
```

**Retry + DLT**
```java
factory.setCommonErrorHandler(new DefaultErrorHandler(
    new DeadLetterPublishingRecoverer(template),
    new FixedBackOff(1000L, 3)));
```

**Durable producer recipe**
```yaml
spring.kafka.producer:
  acks: all
  retries: 5
  properties:
    enable.idempotence: true
```

---

## 8. Self-Test

**Easy**
- [ ] What's the unit of parallelism in Kafka?
- [ ] What's a consumer group?
- [ ] What's an offset?
- [ ] What does `acks=all` do?

**Medium**
- [ ] Walk through how a producer's send becomes a record in a partition.
- [ ] How would you preserve per-customer ordering for events?
- [ ] What's the at-least-once contract and why must consumers be idempotent?
- [ ] When is log compaction the right cleanup policy?

**Hard**
- [ ] Explain the three layers of exactly-once semantics.
- [ ] What's a Cooperative Sticky rebalance and why is it better than the legacy "stop the world" rebalance?
- [ ] Design a payment consumer with retries, DLT, and idempotent processing.
- [ ] What's the durable production recipe for `acks` + `min.insync.replicas` + `replication.factor`?
- [ ] Why is `ISR` not the same as `replication.factor`?

---

## 9. Glossary (in plain English)

| Term | Plain-English meaning |
|------|----------------------|
| **Broker** | A Kafka server. |
| **Topic** | A named append-only log (logically — physically split into partitions). |
| **Partition** | An ordered, immutable, append-only sub-log inside a topic. |
| **Offset** | Position of a record within a partition. |
| **Producer** | Writes records to a topic. |
| **Consumer** | Reads records from a topic. |
| **Consumer group** | Set of consumers sharing the work of consuming a topic. |
| **Leader** | The broker handling all reads/writes for a partition. |
| **Follower** | A replica that pulls from the leader. |
| **ISR (In-Sync Replicas)** | Replicas that are up to date with the leader. |
| **Acks** | How many replicas must ack a write before the producer considers it durable. |
| **Idempotent producer** | Producer with deduplication of retried sends. |
| **Transactional producer** | Producer that can write atomically across partitions. |
| **Exactly-once** | Each record processed once end-to-end (idempotent producer + tx + read_committed). |
| **Rebalance** | Reassigning partitions across consumers in a group. |
| **DLT** | Dead Letter Topic — where consistently-failing messages land. |
| **Compaction** | Cleanup policy that keeps only the latest value per key. |
| **Retention** | Cleanup policy that deletes messages after time/size threshold. |
| **KRaft** | Kafka's built-in metadata management (replacement for ZooKeeper, Kafka 4.0+). |

---

[← All topics](./00_INDEX.md) · [📝 Doubts log](./doubts/15_kafka_doubts.md) · [← Prev: 14 Spring AOP & Data](./14_spring_aop_data.md) · [Next: 16 Debugging →](./16_debugging.md)

[↑ Back to top](#15--kafka-fundamentals)
