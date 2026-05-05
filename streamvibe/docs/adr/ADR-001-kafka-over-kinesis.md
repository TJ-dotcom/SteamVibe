# ADR-001: Apache Kafka (Amazon MSK) over Amazon Kinesis

**Date:** January 2026  
**Status:** Accepted  
**Context:** Ingestion layer message bus selection

---

## Context

StreamVibe needs to ingest two simultaneous high-throughput streams:

1. Twitch IRC chat — text events at thousands of messages per minute during peak moments
2. Audio frames from FFmpeg — binary chunks that need routing to the Whisper ASR pipeline

Both streams need to fan out to the same downstream Spark job but at different processing rates. The system must survive viral spikes (sudden 10x message bursts) without dropping data.

We evaluated Amazon Kinesis Data Streams and Apache Kafka via Amazon MSK.

## Decision

**Apache Kafka via Amazon MSK.**

## Rationale

**Consumer group model.** Kafka lets the chat consumer and audio consumer operate independently with their own offsets. If the audio pipeline falls behind, it doesn't block chat processing. Kinesis shards don't provide the same per-consumer isolation without significant extra engineering.

**Spark connector maturity.** The `spark-sql-kafka` connector is battle-tested in production at large scale. The Spark–Kinesis connector is less mature and has historically had more issues with exactly-once semantics.

**Replay on failure.** Kafka retains messages by configurable offset. In the prototype this was set to 24-hour retention. If a Spark job crashed mid-stream, we could replay from the last committed offset without data loss. This was critical during Whisper ASR failures.

**Local development parity.** The Docker Compose stack runs a real Confluent Kafka instance locally. This means local dev and AWS production behave identically. Kinesis has no true local equivalent — LocalStack's Kinesis emulation introduces subtle behavioral differences.

**ZooKeeper coordination.** MSK + ZooKeeper gives consistent broker management. The docker-compose.yml shows this was a concrete requirement: `ZOOKEEPER_CLIENT_PORT: 2181` with dedicated data volumes.

## Trade-offs Accepted

- MSK has a higher baseline cost (~$58/mo) vs. Kinesis pay-per-shard
- ZooKeeper adds operational overhead, mitigated by the managed MSK service
- Kafka is more complex to configure correctly (partition count, replication factor, compression)

## Configuration Used

```yaml
KAFKA_NUM_PARTITIONS: 6
KAFKA_COMPRESSION_TYPE: 'lz4'
KAFKA_LOG_RETENTION_HOURS: 24
Topics: streamvibe-chat, streamvibe-audio, streamvibe-events
```

## Consequences

Kafka becomes the single source of truth for all in-flight data. All upstream producers (Twitch IRC via `twitch_irc_muscle.py`, FFmpeg audio chunks) write to Kafka. All downstream consumers (Spark EMR jobs) read from Kafka. ZooKeeper manages broker coordination.
