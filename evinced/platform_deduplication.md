# Deduplication Of Items

## Requirements
There was a need to deduplicate traffic in the pipeline to ensure that messages sent more than once are discarded.

## Performance requirements
- Need to write fast, we receive a lot of new unique message
- Need to read fast per message, each message needs to be check if it was received at an earlier stage
- Need to delete keys based on a time window or predefined size

## Overview
Each item must have fields that define its uniqueness, which should generate a hash. This hash will represent the item's "uniqueness," and the item itself is immutable.

The choice was between an embedded or distributed approach. A distributed solution is useful when centralized access from different services is required. However, since the data in this case is solely for deduplicating inbound traffic, an embedded approach seemed more suitable.

## Embedded benefits

- Very fast, close proximity
- Very cheap, no need for another managed cluster
- No network traffic

## Embedded cons

- Distributed state on different DBs
- Rebalancing can be required at times

## RocksDB solution
Here is a diagram of an Embedded solution with RocksDB.

![My SVG Image](/evinced/platform_dedup_flow.svg)

Note that the hash algorithm can be manual or part of functionality provided by platforms like Kafka.

## RocksDB functionality and configuration
![My SVG Image](/evinced/platform_rocksdb_flow.svg)
## The deduplication worker
### Message Deduplication flow
RocksDB was configured with a Bloom filter on the keys. Since we had a read-heavy requirement, the filter reduced disk reads during lookups by quickly determining if a key exists. Additionally, we implemented a multiget, which further improved performance.

The writes were done in batches to enhance performance. In the case of RocksDB, it utilizes an LSM tree along with a write-ahead log, which makes the writing process very fast.

The following diagram depicts what is going on in a single deduplication worker process:
![My SVG Image](/evinced/platform_dedup_worker_flow.svg)
 
## Key design
The key was prefixed with a tenant ID, which provided better data locality and made the Bloom filters more efficient. Additionally, there is an option to add a prefix Bloom filter for iterators, further enhancing performance.

### Removing old keys
Kyes was configured with a TTL.

## Rebalancing
If the database on each partition grows too large, we will need to rebalance. However, this isn't something that has happened yet or is expected to happen in the near future because I accounted for the expected traffic by selecting a sufficient number of partitions.

If rebalancing becomes necessary, we can create a new stateful set with additional workers. We would then stream all the data from a specific point in time to these new workers to build their state.

The following daigram depicts this process:
![My SVG Image](/evinced/platform_dedup_rebalance_flow.svg)

## Honorable mentions
- Redis: Can also work, but is more expensive. Needs to keep the keys in memory. Recovery more complex with snapshot
- Casandra, AeroSpike or any other distributed KVDB: Was an overkill for our needs and more expensive
- SpeedDb: RocksDB compatible, but requires more memory. Is meant for very low latency requirements

Back To [Evinced Pipeline](./platform_pipeline_medium.md)  
Back To [Home](../index.md)
