# Evinced Deduplication Of Items

## Requirements
There was need to deduplicate traffic in the pipeline. It should discard messages being send more than once.

## Performance requirements:
- Need to write fast, we receive a lot of new unique message
- Need to read fast per message, each message needs to be check if it was received at an earlier stage
- Need to delete keys based on a time window or predefined size

## Overview
Each item needs to have fields that defines its uniqueness. These fields should produce a hash. An item is immutable. From now on, we call this hash the “uniqueness” of the item.

The choice was between embedded or distributed. Distributed has it use case if you need a centralised access from different services. Here we want the data for the sole purpose of deduplicating inbound traffic, therefore embedded seemed more suited.

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
RocksDb was configured with a bloom filter on the keys. Because we had a read heavy need, the filter reduced disk reads when we do lookups if the key exists. On top of this we did a multiget which futher improves performance.

The writes was done in batches to improve performance. In the case of RocksDB, it utilizes a LSM tree with write ahead log. This make writing very fast.

The following diagram depicts what is going on in a single deduplication worker process:
![My SVG Image](/evinced/platform_dedup_worker_flow.svg)
 
## Key design
The key was prefixed with a tenant Id. This gave more locality to the data and help the bloom filters be more efficient. On top of this, there is the option to add a prefix bloom filter for iterators.

### Removing old keys
Kyes was configured with a TTL

## Rebalancing
If we get to a state that the DB on each partition gets to big, we will need to rebalance. This is not something that happened yet or will in near future because I planned the expected traffic by picking enough partitions. 

But if we need to, we can create a new stateful set with more workers. Then stream all the data from a specific time to these workers to build its state.

The following daigram depicts this process:
![My SVG Image](/evinced/platform_dedup_rebalance_flow.svg)

## Honorable mentions
- Redis: Can also work, but is more expensive. Needs to keep the keys in memory. Recovery more complex with snapshot
- Casandra, AeroSpike or any other distributed KVDB: Was an overkill for our needs and more expensive
- SpeedDb: RocksDB compatible, but requires more memory. Is meant for very low latency requirements

Back To [Evinced Pipeline](./platform_pipeline.md)  
Back To [Home](../index.md)
