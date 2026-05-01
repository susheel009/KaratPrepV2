# 15 — Kafka Fundamentals

[← Back to Index](./00_INDEX.md) | **Priority: 🟡 High**

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
