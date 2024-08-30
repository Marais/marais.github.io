# Architecting for Scale: A Case Study in Using ClickHouse for High-Volume Data Pipeline Processing and Asynchronous Updates
At Evinced, a company that focus on accessiblity complaince on enterprise clients, we have tools that produce a huge amount of traffic send to our SaaS platform. 
This data is used to do smart categorization through various techniques includding machine learning models.
We developed a pipeline where the requirements involved handling traffic from various sources, including a scraping and scanning service that generated massive bursts of traffic. 
Additionally, there was a need to update a large number of records, allowing users to categorize items into groups based on an arbitrary function of fields, as well as label certain items or records.
Moreover, query response times needed to be sub-second. 
The expected traffic volume ranged from 1 to 5 billion records per day.

# Choosing ClickHouse
We decided to solve the problem with clickhouse cloud after considering many othger solutions.
Clickhouse.com offers a managed ClickHouse database with an exclusive cloud engine called ShardMergeTree, which handles sharding and scaling. Additionally, 
it supports an idle mode where costs nearly drop to zero while keeping the service running. 
ClickHouse also features an engine called CollapsingMergeTree, which appears ideal for the company's update needs. 
This engine allows real-time materialized views on your traffic/fact tables. However, to maintain the accuracy of materialized views, 
you must provide the full state of the record you want to update so ClickHouse can correctly adjust the aggregated data. 
Given this significant constraint, we decided to explore whether we could overcome this limitation which is described in the Update Design section.

## Ingestion Design
Every item can be categorized. Because the criteria for categorizing an item can be very complex and expensive to solve on a query, we decided to solve this in the ingestion phase.

![My SVG Image](/evinced/platform_ingestion.svg)

## Query Performance

First, we demonstrated that ClickHouse can handle large queries on our data. I imported a massive dataset via Spark and began testing real-world queries. The results were impressive: on the traffic table, I achieved 393.65 million rows per second and 7.39 GB per second. Most queries returned in under a second, though some complex queries still took 40 seconds. To optimize these, I utilized materialized views, ensuring they stayed up to date thanks to the CollapsingMergeTree. After implementing the materialized views, we reduced the 40-second queries to just a few milliseconds.

This was exciting and promising news. However, one major challenge remained: you need the full state of a record if you want to update it. If you don't have the state, you must also read from ClickHouse, which could result in numerous point queries—a significant drawback for ClickHouse. In the next section I address how this limitation was solved.

## Update Design

The solution we developed was to make the updates asynchronous using an orchestrator that schedules which updates can run in parallel. Given that our product requires infrequent updates, handling these occasional requests with a scheduler was acceptable. The orchestrator would schedule a worker process that streams data from ClickHouse, applies the updates, and then streams the data back into the ingestion pipeline. This pipeline ultimately inserts two records for each updated entry: one with the original state to update the chain of materialized views reflecting the record's removal, and one with the new state.

The following diagram depicts the flow:
![My SVG Image](/evinced/platform_update.svg)

The key aspect of this design is that it performs streaming updates rather than batch updates, significantly reducing memory requirements. The streamer service was developed using the Go native ClickHouse driver, which supports streaming with large SELECT statements. Additionally, failure recovery measures were implemented to ensure the system could recover from any failed updates. In testing, I successfully updated a billion records within minutes, thanks to the service being in the same region as the ClickHouse host. This was remarkable because it ensured that the aggregation tables were always up to date.

## Deplicate Message Resililence
One unnegotianable requirement was that the pipeline needed to be deterministic.
To meet this requirement the update pipeline was designed to be idempotent. Well kind of...

### PubSub
We use google pubsub for the message queue between micro services. To achieve deplication resilence in the pipeline, we configured the Pubsub to have a big Ack timeout and ensure that each micro service doesn't hold a message for to much time.
This is not a gaurentee fore eaxtly once dilivry, e.g. the service could crash to name one problem. We designed our microservices with graceful shut downs.

With these prevetive measure put in place we were happy that the probibility of deplicate messages was low and this was an acceptible risk for us.

### ClickHouse
What was left is to make writes to clickhouse idempotent.
To achieve this, the write to Clishouse is done syncronizly. Clickhouse doesnt support transactional commits, but what it does offer is duplication block detection.
If the batch write fails, the exact batch will be retried until it succeeds before it moves on to the next batch. Clickhosue uses a blockID,  which is a hash of the data in that block/batch. This block_id is used as a unique key for the insert operation. If the same block_id is found in the deduplication log, the block is considered a duplicate and is not inserted into the table. That took care of the traffic table write to be idempotent.
What was left was to ensure that the materialzed views behave the same. Clickhouse provides a lot of settings to fine tune block deduplication and one of them is deduplicate_blocks_in_dependent_materialized_views. By simply enabling this settings, we could ensure that the aggreagtion tables will not be affected by duplicate blocks. Also we ensured that all our aggreagte functions we used were deterministics.


### Update
To achieve idenpotency on the update pipline, the message produced by the streamer is a record pair, one for the removal of the current record and one for the new record. The sink then batches messages together, garenteing that a write to the database will contain both records in the pair.


DIAGRAM


## Replay Ability
As with any pipeline, it's important to have the ability to replay specific data chunks and reprocess them if needed. To address this, the raw input was saved in Parquet format, partitioned by tenant and date.

![My SVG Image](/evinced/platform_replay.svg)

## Query API Design
To meet the requirement for a flexible API, I designed a GraphQL engine that translates queries into ClickHouse queries. All complex query logic is encapsulated within ClickHouse views, and the GraphQL layer simply reflects the view fields, providing filtering, sorting, and aggregation capabilities on those fields. This design allows us to easily create a new view whenever a new API is needed!

## Optimizing Queries on the Trsffic Table
tenants were mared as dirty

## Conclusion
The solution chosen for this project was to use ClickHouse's CollapsingMergeTree in combination with the update streaming solution. Thanks to ClickHouse's idling capabilities and scalability, we were able to start with a very small cluster, significantly reducing startup costs compared to competitors.

Back To [Home](../index.md)
