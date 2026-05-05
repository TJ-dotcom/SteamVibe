# ADR-002: Amazon Keyspace (Cassandra) over DynamoDB

**Date:** January 2026  
**Status:** Accepted  
**Context:** Operational storage for processed chat messages, transcriptions, and timestamps

---

## Context

The system needs to persist three types of processed data:

1. Chat messages with sentiment scores and NTP timestamps
2. Audio transcription segments with word-level timing
3. Viral moment events with correlated chat + audio metrics

Access patterns are predominantly write-heavy with time-range reads (e.g., "give me all messages in the last 30 seconds for channel X"). The data model is naturally wide-row: each stream session produces thousands of rows keyed by `(channel_name, timestamp)`.

We evaluated Amazon DynamoDB and Amazon Keyspace (Cassandra-compatible).

## Decision

**Amazon Keyspace (Apache Cassandra via AWS managed service).**

## Rationale

**Write throughput.** Cassandra's LSM-tree storage engine is optimized for write-heavy workloads. Chat ingestion during viral moments can spike to thousands of writes per second per channel. DynamoDB's WCU model would require significant pre-provisioning or on-demand pricing spikes to handle these bursts.

**Data model fit.** The access pattern — time-series data partitioned by channel with timestamp clustering — maps naturally to Cassandra's partition key + clustering column model. The schema used in the original `twitch-chat-insight` component already demonstrated this:

```sql
CREATE TABLE twitch_chat_messages (
    channel_name TEXT,
    message TEXT,
    username TEXT,
    sentiment TEXT,
    timestamp TIMESTAMP,
    PRIMARY KEY (channel_name, username)
);
```

**Timestamp-based queries.** Cassandra's clustering columns make range scans on timestamps efficient. Querying "the last 30 seconds of chat for viral window calculation" is a native access pattern. DynamoDB requires a GSI with additional cost and complexity for the same query.

**Prototype continuity.** The `twitch-chat-insight` component (The Muscle) was already built against Cassandra. Switching to DynamoDB would have required rewriting the Spark Cassandra connector integration.

**AWS Keyspace managed.** Amazon Keyspace removes the operational burden of managing Cassandra nodes — no JVM tuning, no repair cycles.

## Trade-offs Accepted

- Amazon Keyspace has quirks vs. native Cassandra (limited CQL feature support, no lightweight transactions)
- Less flexible for ad-hoc analytical queries compared to DynamoDB + PartiQL
- Consistency is eventually consistent by default — acceptable for this use case since we don't need cross-channel consistency

## Consequences

Cassandra (Amazon Keyspace) stores all processed operational data. Amazon S3 stores raw audio and transcription archives for long-term storage. Amazon InfluxDB stores time-series metrics and performance stats. The three-tier storage model separates hot (Cassandra), warm (S3), and metrics (InfluxDB) concerns cleanly.
