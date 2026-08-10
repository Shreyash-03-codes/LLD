# Design a Pub-Sub System Like Kafka: Internal Data Structures of the Broker

## Learning Objectives

- Learn the storage model that makes a broker fast, the append-only log, and why the file system ends up being the data structure.
- Model topics, partitions, and offsets, and understand why the partition is the unit of parallelism for both producers and consumers.
- Design the consumer group and its offset tracking, and know where the at-least-once and at-most-once trade-off actually lives.

## Introduction

This case study is the deepest dive in the chapter, and it deserves to be. The chat system used a broker without opening it. This one opens it. A pub-sub broker like Kafka is a distributed, persistent, append-only log, and every interesting property falls out of that phrase. It is fast because it writes sequentially. It is durable because it never deletes until told to. It lets consumers replay because it keeps a position, an offset, per consumer instead of deleting what was read. And it scales because the log is split into partitions, and a partition is the unit that can be moved, replicated, and processed independently. Interviewers ask this because a candidate who can explain the internals of a log-based broker has demonstrated they understand the systems underneath half the backend products they would work on.

## Requirements Gathering

Functional requirements:

- A producer publishes a record to a topic; the record has a key, a value, and a timestamp.
- A topic is divided into partitions; records with the same key land in the same partition.
- A consumer subscribes to a topic, reads records in order within a partition, and tracks its position.
- A consumer group shares a topic's partitions, each partition consumed by one member of the group.
- Records are retained for a retention period and can be replayed by resetting a consumer's offset.

Non-functional requirements:

- Writes and reads must be bounded-latency and high-throughput, which the sequential log delivers.
- The broker must survive the loss of a broker node without losing committed records.

Assumptions to state out loud: a topic is created with a fixed partition count (no dynamic re-partitioning), record ordering is guaranteed per partition and not across partitions, the cluster is small enough that a single controller does the leadership bookkeeping, and exactly-once semantics are out of scope, replaced by the honest at-least-once with idempotent consumers. Cut exactly-once, it is a protocol, not a class, and it would eat the interview.

## Identifying Core Entities

The entity list is the internal anatomy of one broker plus the cluster-level pieces.

| Entity | One-line responsibility |
| --- | --- |
| `Topic` | A named log split into partitions. |
| `Partition` | An ordered, append-only sequence of records, the unit of parallelism. |
| `Record` | The payload: key, value, timestamp, and a per-partition offset. |
| `LogSegment` | A file-backed chunk of a partition's records. |
| `Offset` | The monotonic position of a record within a partition. |
| `PartitionLeader` | The broker replica that owns writes and reads for its partition. |
| `ConsumerGroup` | The set of consumers that divide a topic's partitions. |
| `OffsetTracker` | Records each consumer group's position per partition. |
| `BrokerController` | The cluster coordinator that assigns partition leadership. |

The two entities doing the deep work are `LogSegment`, which is the storage model, and `OffsetTracker`, which is the delivery model.

## Class Design

Start with the record and the offset. The offset is the record's address within a partition, and it is the single number the whole delivery model hangs on.

```java
public class Record {
    private final long offset;
    private final byte[] key;
    private final byte[] value;
    private final long timestampMillis;

    public Record(long offset, byte[] key, byte[] value, long timestampMillis) {
        this.offset = offset;
        this.key = key;
        this.value = value;
        this.timestampMillis = timestampMillis;
    }

    public long getOffset() { return offset; }
    public byte[] getKey() { return key; }
    public byte[] getValue() { return value; }
}
```

`LogSegment` is the storage unit. A partition is not one file; it is a series of segments, each a fixed-size or fixed-time chunk, written once, appended to, and never modified. The append is sequential, which is why the broker writes fast: a sequential write to disk is orders of magnitude cheaper than random writes, and the segment design exists to keep writes sequential and to make old segments the unit of deletion and compaction.

```java
public class LogSegment {
    private final FileSegment file;
    private long baseOffset;
    private long nextOffset;

    public LogSegment(FileSegment file, long baseOffset) {
        this.file = file;
        this.baseOffset = baseOffset;
        this.nextOffset = baseOffset;
    }

    public long append(byte[] key, byte[] value, long timestamp) {
        long offset = nextOffset;
        // sequential write: serialize record, write to end of the segment file
        file.append(recordBytes(offset, key, value, timestamp));
        nextOffset++;
        return offset;
    }

    public Record read(long offset) {
        // sequential scan from baseOffset or an index; returns the record
        return null; // index lookup elided
    }

    public boolean covers(long offset) {
        return offset >= baseOffset && offset < nextOffset;
    }
}
```

`Partition` owns its list of segments and answers the two operations the whole broker needs: append a record and read a record at an offset. The partition is where the ordering guarantee physically lives, because it is the serial append point.

```java
public class Partition {
    private final String topic;
    private final int partitionId;
    private final List<LogSegment> segments = new ArrayList<>();
    private final ReentrantLock appendLock = new ReentrantLock();

    public Partition(String topic, int partitionId, long baseOffset) {
        this.topic = topic;
        this.partitionId = partitionId;
        segments.add(new LogSegment(new FileSegment(topic, partitionId, baseOffset), baseOffset));
    }

    public long append(byte[] key, byte[] value, long timestamp) {
        appendLock.lock();
        try {
            LogSegment active = segments.get(segments.size() - 1);
            return active.append(key, value, timestamp);
        } finally {
            appendLock.unlock();
        }
    }

    public Optional<Record> read(long offset) {
        for (LogSegment segment : segments) {
            if (segment.covers(offset)) {
                return Optional.ofNullable(segment.read(offset));
            }
        }
        return Optional.empty();
    }

    public String getTopic() { return topic; }
    public int getPartitionId() { return partitionId; }
}
```

The lock on append is the one concurrency detail worth writing. A partition's append is serialized, because the offset sequence is the ordering guarantee, and two concurrent appends must not interleave records. The lock is how the partition stays a single logical log.

`OffsetTracker` is the delivery model. Each consumer group stores, per partition, the offset of the next record it wants. This is the design decision that makes the broker different from a queue: the broker does not delete a record when it is read, it keeps it and records the consumer's position, which is what makes replay possible.

```java
public class OffsetTracker {
    private final Map<String, Map<String, Long>> groupOffsets = new ConcurrentHashMap<>();
    // key: consumerGroup -> partitionKey(topic:partition) -> next offset to read

    public long nextOffset(String group, String topic, int partitionId) {
        return groupOffsets
                .computeIfAbsent(group, k -> new ConcurrentHashMap<>())
                .getOrDefault(topic + ":" + partitionId, 0L);
    }

    public void commit(String group, String topic, int partitionId, long offset) {
        groupOffsets.get(group).put(topic + ":" + partitionId, offset + 1);
    }
}
```

The commit is the at-least-once story in one method. A consumer reads a batch, processes it, then commits. If it crashes between reading and committing, the offset is stale, and on restart it reads the same records again, which is at-least-once: duplicates possible, nothing lost. Commit before processing and a crash loses the batch, which is at-most-once. The candidate who can place that trade-off on the commit method has the whole delivery model.

`BrokerController` assigns each partition to a leader replica and handles failover. In the interview version it is a table of partition to leader broker plus a small heartbeat protocol; in production it is ZooKeeper or KRaft. The candidate who names the controller's job, keep exactly one leader per partition so appends serialize, has answered the replication question's core.

Diagram: a topic as append-only partitions. Parallelism comes from partitions; ordering comes from the serialized append inside each. The consumer group tracks a per-partition offset, and the commit decides the delivery semantics.

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 550" font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif">
  <defs>
    <marker id="ah7" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#64748b"/>
    </marker>
  </defs>
  <rect width="900" height="550" fill="#ffffff"/>

  <text x="450" y="28" text-anchor="middle" font-size="19" font-weight="bold" fill="#1f2937">A broker is a log, not a queue</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none">
    <line x1="28" y1="118" x2="60" y2="118"/>
    <line x1="28" y1="118" x2="28" y2="292"/>
    <line x1="28" y1="168" x2="60" y2="168"/>
    <line x1="28" y1="228" x2="60" y2="228"/>
    <line x1="28" y1="288" x2="60" y2="288"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="60" y="78" width="160" height="40" rx="8" fill="#e0e7ff" stroke="#6366f1"/>
    <text x="140" y="103" text-anchor="middle" font-weight="bold" fill="#3730a3">Topic: orders</text>

    <rect x="60" y="150" width="560" height="36" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="76" y="173" fill="#1e3a8a" font-weight="bold">P0 · offsets: 0 1 2 3 4 5 6 7 8 9 →</text>
    <rect x="60" y="210" width="560" height="36" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="76" y="233" fill="#1e3a8a" font-weight="bold">P1 · offsets: 0 1 2 3 4 5 →</text>
    <rect x="60" y="270" width="560" height="36" rx="8" fill="#dbeafe" stroke="#3b82f6"/>
    <text x="76" y="293" fill="#1e3a8a" font-weight="bold">P2 · offsets: 0 1 2 3 4 →</text>
  </g>

  <line x1="340" y1="150" x2="340" y2="186" stroke="#64748b" stroke-width="1.6" stroke-dasharray="5 4"/>
  <text x="120" y="140" text-anchor="middle" font-size="12" fill="#6b7280">segment 0 (rolled)</text>
  <text x="430" y="140" text-anchor="middle" font-size="12" fill="#6b7280">segment 1 (active, sequential append)</text>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah7)">
    <line x1="680" y1="240" x2="624" y2="168"/>
    <line x1="680" y1="240" x2="624" y2="228"/>
    <line x1="680" y1="240" x2="624" y2="288"/>
  </g>

  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="680" y="120" width="200" height="46" rx="8" fill="#f5f3ff" stroke="#a78bfa"/>
    <text x="780" y="138" text-anchor="middle" font-weight="bold" fill="#5b21b6">ConsumerGroup 'payments'</text>
    <text x="780" y="156" text-anchor="middle" font-size="12" fill="#6d28d9">members: C1, C2, C3</text>
    <rect x="680" y="200" width="200" height="80" rx="8" fill="#fffbeb" stroke="#f59e0b"/>
    <text x="780" y="222" text-anchor="middle" font-weight="bold" fill="#92400e">OffsetTracker</text>
    <text x="780" y="240" text-anchor="middle" font-size="12" fill="#92400e">group → topic:partition</text>
    <text x="780" y="258" text-anchor="middle" font-size="12" fill="#92400e">→ next offset to read</text>
    <text x="780" y="274" text-anchor="middle" font-size="12" fill="#b45309">payments → orders:0 → 4</text>
  </g>

  <g stroke="#64748b" stroke-width="1.8" fill="none" marker-end="url(#ah7)">
    <line x1="220" y1="403" x2="246" y2="403"/>
    <line x1="410" y1="403" x2="436" y2="403"/>
  </g>
  <g font-family="-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif" font-size="13" fill="#1f2937">
    <rect x="60" y="380" width="160" height="46" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="140" y="407" text-anchor="middle" font-weight="bold" fill="#334155">read batch</text>
    <rect x="250" y="380" width="160" height="46" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="330" y="407" text-anchor="middle" font-weight="bold" fill="#334155">process</text>
    <rect x="440" y="380" width="160" height="46" rx="8" fill="#f8fafc" stroke="#cbd5e1"/>
    <text x="520" y="407" text-anchor="middle" font-weight="bold" fill="#334155">commit (advance offset)</text>
  </g>
  <g stroke="#b91c1c" stroke-width="1.8" stroke-dasharray="6 4">
    <line x1="250" y1="368" x2="410" y2="368"/>
  </g>
  <text x="330" y="362" text-anchor="middle" font-size="12.5" fill="#b91c1c" font-weight="bold">crash window</text>
  <text x="640" y="403" font-size="12.5" fill="#475569">stale offset →</text>
  <text x="640" y="419" font-size="12.5" fill="#475569">re-read same records</text>

  <rect x="60" y="480" width="400" height="48" rx="8" fill="#f0fdf4" stroke="#bbf7d0"/>
  <text x="260" y="508" text-anchor="middle" font-size="13" fill="#14532d" font-weight="bold">at-least-once — crash in window: duplicates possible, nothing lost</text>
  <rect x="480" y="480" width="400" height="48" rx="8" fill="#fef2f2" stroke="#fecaca"/>
  <text x="680" y="508" text-anchor="middle" font-size="13" fill="#7f1d1d" font-weight="bold">atmost-once commit before processing: crash loses the batch</text>
</svg>
```

## Design Patterns Used

The structural ideas here are not GoF patterns, and the honest answer is to say so. The append-only log is an event-sourcing pattern in miniature, state as an ordered stream of events. The segment is a classic log-structured storage pattern, the same idea under LSM trees. The consumer group is a work-stealing-ish partition assignment. The candidate who says "this is a log-structured design, and the GoF patterns that apply to application code mostly do not apply to a storage engine" has demonstrated a senior truth: storage engines are not shape-fitted with visitors and strategies, they are shaped by the physics of the disk. There is one place a strategy earns its keep, the record serialization and index format, and naming it is honest without building it.

## Handling Edge Cases / Concurrency

The concurrency story is partitioned, literally. Appends to a partition serialize under the partition's lock, appends to different partitions do not block each other, and that is how a broker gets throughput: parallelism at the partition, serialization inside it. The candidate who states that sentence has the broker's concurrency model.

The edges beyond the lock: a consumer that lags behind retention, so the records it wants are already deleted, must fail with a clear error and reset to the earliest available offset, which is why segments are the deletion unit and why the earliest offset per partition is tracked. A broker that fails must have its partitions reassigned, and the at-least-once model absorbs the duplicate records a failover produces, which is why idempotent consumers are the contract, not a feature. The duplicate delivery on consumer-group rebalance, where one consumer crashes and another takes its partition starting from the last commit, is the same at-least-once story wearing a different name, and naming it unprompted is a strong signal.

## Common Mistakes

The most common mistake is modeling the broker as a queue of messages. A `Queue<Record>` per topic, deleted on read. That model cannot replay, cannot have multiple consumer groups reading the same topic at different positions, and cannot let a new consumer group read the whole history. The difference between a queue and a log with offsets is the entire case study, and the candidate who draws a queue has drawn the wrong product.

The second mistake is the single global log. One ordered log for everything, no partitions, which means one serialization point for the whole cluster and one consumer group possible. The partition is what makes parallelism and independent consumer groups possible, and a topic with one partition is a topic that does not scale.

The third mistake is hand-waving the offset. The candidate says "the broker tracks what each consumer read" and cannot say where the tracking lives, when it commits, or what a crash between read and commit does. The `OffsetTracker` with its commit semantics is the delivery model, and the at-least-once trade-off is its direct consequence.

## Interview Perspective

A weak answer is a `Map<String, Queue<Message>>` called a broker. The interviewer asks "two consumer groups read the same topic at different speeds" and the queue model has no answer, because reading deletes. The interview is over at that question.

A strong answer says "the broker is an append-only log split into partitions, segments make deletion and recovery cheap, the offset is the consumer's position and the commit is where the delivery semantics live, and parallelism comes from the partition while ordering comes from the lock inside it." Follow-ups to expect: "what happens when a consumer is slow" (it lags, its offset trails, and it is at risk of running past retention, which is why offsets and retention are tracked together), "how do you add a consumer group later" (it starts at any offset it chooses, earliest, latest, or a specific point, which is the replay property no queue has), "what happens when a broker dies" (the controller reassigns its partitions, the new leader replays from the committed log, and at-least-once duplicates are the accepted cost). The strongest candidates volunteer the read-commit crash scenario and the idempotent-consumer contract unprompted.

## Knowledge Check

1. A consumer reads records 10 through 19, processes them, and crashes before committing. Walk through what the `OffsetTracker` holds, what the consumer reads after restart, and which delivery semantics this is.
2. Two records with the same key arrive at a topic with four partitions. Explain which partition each lands in, why the key exists, and what ordering guarantee holds for those two records.
3. A consumer group has two members and a topic has four partitions. Describe how the partitions divide, what happens when one member crashes, and why the offset commit decides whether the crash loses messages.

## Key Takeaways

- A broker is a log, not a queue. Offsets make replay and multiple consumer groups possible; deletion-on-read is the wrong product.
- The segment is the storage unit, and sequential append is why the broker is fast.
- Parallelism comes from partitions, ordering comes from the serialized append inside each one.
- The offset commit is the delivery semantics: commit after processing means at-least-once, and idempotent consumers are the contract.
- The controller keeps one leader per partition, and failover duplicates are absorbed by at-least-once.

## What's Next

The broker stored, routed, and replayed records with a log and offsets. The payment processing system flips from throughput back to correctness of a different kind: the money must move once, the retry must not double-charge, and the idempotency key that was a guard clause in the e-commerce chapter becomes the entire architecture.

---

This article explains how to design a pub-sub broker as an append-only log split into partitions and stored in file-backed segments. Its point of view is that a broker is a log, not a queue, and the offset commit owns the at-least-once trade-off.
