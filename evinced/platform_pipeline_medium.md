# Architecting for Scale
## A Case Study in Using ClickHouse for High-Volume Data Pipeline Processing and Asynchronous Updates
This article was also published on [medium.com](https://medium.com/@marais.kruger/architecting-for-scale-e998fc0adef0) 

At Evinced, a company focused on accessibility compliance for enterprise clients, we manage tools that generate a significant amount of traffic to our SaaS platform. This data is leveraged for smart categorization and accessibility issue detection using various techniques, including machine learning models.

We developed a pipeline designed to handle traffic from multiple sources, such as a scraping and scanning service that produced massive bursts of traffic. 

The following requirements had to be addressed:
- Handle huge ingress of data
- Need the ability to update of a large number of records
- Query response times needs to be sub-second
- Categorize items into groups based on custom functions

# Choosing ClickHouse
After considering several alternatives such as Apache Pinot, Rockset, Apache Druid, and Firebolt, we ultimately decided to go with ClickHouse Cloud. While each of these databases has its own advantages, this article focuses on why we chose ClickHouse and how it met our specific requirements at a minimal cost compared to other solutions.

ClickHouse.com offers a managed ClickHouse database with a unique cloud engine called SharedMergeTree, which simplifies sharding and scaling far more than managing your own ClickHouse cluster. Additionally, it supports an idle mode that significantly reduces costs while keeping the service operational. ClickHouse also provides a rich set of engines, which leads us to the next section.
## Choosing the Engine
As I mentioned earlier, ClickHouse offers a wide range of engines within the MergeTree family. Our next task was to determine which engine best suited our needs. One of our key requirements was to support updates. 

The standard MergeTree engine does indeed support updates, deletes, and lightweight deletes, which offer better performance for delete operations. However, it’s important to understand that updates and deletes do not automatically trigger changes downstream in materialized views. ClickHouse does offer projections that support updates and deletes with the lightweight_mutation_projection_mode setting. We explored this option, but our complex queries required the use of materialized views. Although updates were infrequent, they were still significantly large, which would degrade performance. 

ClickHouse also features an engine called CollapsingMergeTree, which can handle large volumes of update/delete traffic. This engine allows for real-time materialized views on your traffic or fact tables. However, to maintain the accuracy of materialized views, you must provide the full state of the record you wish to update so that ClickHouse can correctly adjust the aggregated data. Despite this limitation, the significant advantages of this engine prompted me to explore whether we could overcome this constraint. In the "Update Design" section later in this article, I explain how we successfully addressed this challenge.

# Ingestion Design
As I mentioned earlier, we performed categorization based on a function of the record fields. Since the criteria for categorizing an item can be very complex and computationally expensive to solve during a query, we decided to handle this during the ingestion phase. The following diagram illustrates the ingestion pipeline.

![My SVG Image](/evinced/platform_ingestion.svg)

# Raw Query Performance

First, we demonstrated that ClickHouse can handle large queries on our data. I imported a massive dataset via Spark and began testing real-world queries. The results were impressive: on the traffic table, I achieved 393.65 million rows per second and 7.39 GB per second. Most queries returned in under a second, though some complex queries still took 40 seconds. To optimize these, I utilized materialized views, ensuring they stayed up to date thanks to the CollapsingMergeTree. After implementing the materialized views, we reduced the 40-second queries to just a few milliseconds.

This was exciting and promising news. However, one major challenge remained: you need the full state of a record if you want to update it. If you don't have the state, you must also read from ClickHouse, which will result in numerous point queries. In the next section I address how this limitation was solved.

# Update Design

We decided to experiment with the idea of using the same pipeline as the ingestion for updates as well. The solution we developed was to make the updates asynchronous using an orchestrator that schedules which updates can run in parallel. Given that our product requires infrequent updates, handling these occasional requests with a scheduler was acceptable. The orchestrator would schedule a worker process that streams data from ClickHouse, applies the updates, and then streams the data back into the ingestion pipeline. This pipeline ultimately inserts two records for each updated entry: one with the original state to update the chain of materialized views reflecting the record's removal, and one with the new state.

The following diagram depicts the flow:
![My SVG Image](/evinced/platform_update_with_pairs.svg)

The key aspect of this design is its ability to perform streaming updates instead of batch updates. This approach allows throughput throttling and significantly reduces memory requirements. The streamer service was developed using the Go native ClickHouse driver, which supports streaming with large SELECT statements. 

Because the order of messages in our update pipeline could not be guaranteed, we chose the VersionedCollapsingMergeTree over the CollapsingMergeTree. This choice ensures that the correct record is matched regardless of the order in which the messages arrive.

Additionally, failure recovery measures were implemented to ensure the system could recover from any failed updates. In testing, I successfully updated a billion records within minutes, thanks to the service being in the same region as the ClickHouse host. This was remarkable performance.

# Duplicate Message Resilience
One of the requirements was for the pipeline to be deterministic. To achieve this, the update pipeline was designed to be idempotent—well, sort of...

## PubSub
We use Google Pub/Sub as the message queue between our microservices. To achieve resilience against duplication in the pipeline, we configured Pub/Sub with a long acknowledgment timeout. We also ensured that each microservice doesn't hold onto a message for too long. However, this does not guarantee exactly-once delivery; for example, a service could crash or be terminated unexpectedly. To mitigate expecetd service termination, we designed our microservices with graceful shutdowns to avoid leaving tasks partially completed.

With these preventive measures in place, we were confident that the probability of duplicate messages was low, and we considered this an acceptable risk.

## ClickHouse
The remaining task was to make writes to ClickHouse idempotent. To achieve this, writes to ClickHouse were done synchronously. ClickHouse, being an OLAP database, does not support transactional commits, but it does offer duplication block detection.

ClickHouse uses a block ID, which is a hash of the data in a block or batch. This block ID acts as a unique key for the insert operation. If the same block ID is found in the deduplication log, the block is considered a duplicate and is not inserted into the table. This approach made the traffic table write idempotent.

The next step was to ensure that the materialized views behaved similarly. ClickHouse provides several settings to fine-tune block deduplication, and one of them is deduplicate_blocks_in_dependent_materialized_views. By enabling this setting, we ensured that the aggregation tables would not be affected by duplicate blocks.

To leverage this feature of ClickHouse, we designed the sink so that if a batch write fails, the exact batch is retried until it succeeds before moving on to the next batch. This guarantees that the same block ID will be generated, and any duplicate block will be discarded by ClickHouse. Additionally, we ensured that all the aggregate functions we used were deterministic.

## Update
To avoid having unbalanced record pairs in the VersionedCollapsingMergeTree table, the streamer produces a record pair for each update: one for the removal of the current record and one for the new record. The sink then batches these messages together, ensuring that a write to the database contains both records for remove and add. This approach eliminates the risk of having unbalanced records in the VersionedCollapsingMergeTree, meaning you cannot have a removal record without a corresponding insert. While this doesn't make the update pipeline completely resilient against deduplicate message delivery, it still has the risk associated with incorrect aggregation updates via the materialized views. This risk was deemed acceptable for us since the same risk was present in the regular ingestion pipeline.

Additionally, the SELECT statement was enforced to include an upper ingestion timestamp in the WHERE clause. This kept the number of records streamed deterministic, allowing us to predict how many records would be updated. If the update failed, it could be retried.

# Replay Ability
As with any pipeline, it's important to have the ability to replay specific data chunks and reprocess them if needed. To address this, the raw input was saved in Parquet format, partitioned by tenant and date.

![My SVG Image](/evinced/platform_replay.svg)

# Query API Design
To meet the requirement for a flexible API, we designed a GraphQL engine that translates GQL into ClickHouse queries. All complex query logic is encapsulated within ClickHouse views, while the GraphQL layer simply reflects the view fields, offering filtering, sorting, and aggregation capabilities on those fields. This design enables us to easily create a new view that is automatically added to the API. Metadata can be easily retrieved using a GraphQL introspection query.

## Optimizing Tenant-Specific Queries on the Traffic Table
One important aspect to understand about ClickHouse's CollapsingMergeTree table engine is that it applies collapsing lazily in the background. This means that if you need to retrieve the updated records immediately, you must instruct ClickHouse to apply the collapse by using the FINAL keyword in your query. However, in our tests, we observed a significant performance degradation when including the FINAL keyword in queries.

Given that updates per tenant are infrequent, we leveraged this fact to our advantage. Whenever a tenant triggers an update, we mark the tenant as "dirty" in the API service. We then set up a cron job to force optimization during periods of low query and ingestion activity:
``` SQL
OPTIMIZE TABLE my_table FINAL;
```
With this in place, when a tenant makes a query and is not marked as "dirty," we can omit the FINAL keyword from the query. This approach helps maintain performance while ensuring that only necessary queries undergo the collapsing process.

# Conclusion
The solution chosen for this project was to use ClickHouse's CollapsingMergeTree in combination with the update streaming solution. Thanks to ClickHouse's idling capabilities and scalability, we were able to start with a very small cluster, significantly reducing startup costs compared to competitors.

Back To [Home](../index.md)

