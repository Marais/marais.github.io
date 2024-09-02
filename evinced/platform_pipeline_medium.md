# Architecting for Scale: A Case Study in Using ClickHouse for High-Volume Data Pipeline Processing and Asynchronous Updates
At Evinced, a company focused on accessibility compliance for enterprise clients, we manage tools that generate a significant amount of traffic to our SaaS platform. This data is leveraged for smart categorization and accessibility issue detection using various techniques, including machine learning models.

We developed a pipeline designed to handle traffic from multiple sources, such as a scraping and scanning service that produced massive bursts of traffic. Additionally, the pipeline needed to accommodate the updating of a large number of records, enabling users to categorize items into groups based on custom functions of fields and to label specific items or records. Furthermore, query response times were required to be sub-second.

# Choosing ClickHouse
After considering several alternatives such as Apache Pinot, Rockset, Apache Druid, and Firebolt, we ultimately decided to go with ClickHouse Cloud. While each of these databases has its own advantages, this article focuses on why we chose ClickHouse and how it met our specific requirements at a minimal cost compared to other solutions.

ClickHouse.com offers a managed ClickHouse database with a unique cloud engine called SharedMergeTree, which simplifies sharding and scaling far more than managing your own ClickHouse cluster. Additionally, it supports an idle mode that significantly reduces costs while keeping the service operational. ClickHouse also provides a rich set of engines, which leads us to the next section.
## Choosing the Engine
As I mentioned earlier, ClickHouse offers a wide range of engines within the MergeTree family. Our next task was to determine which engine best suited our needs. One of our key requirements was to support updates. The standard MergeTree engine does support updates, deletes, and lightweight deletes, which offer better performance for delete operations. However, it’s important to understand that updates and deletes do not automatically trigger changes downstream in materialized views. ClickHouse does offer projections that support updates and deletes with the lightweight_mutation_projection_mode setting.

We explored this option, but our complex queries required the use of materialized views. Although updates were infrequent, they were still significantly large, which would degrade performance. ClickHouse also features an engine called CollapsingMergeTree, which can handle large volumes of update/delete traffic.

This engine allows for real-time materialized views on your traffic or fact tables. However, to maintain the accuracy of materialized views, you must provide the full state of the record you wish to update so that ClickHouse can correctly adjust the aggregated data. Despite this limitation, the significant advantages of this engine prompted me to explore whether we could overcome this constraint. In the "Update Design" section later in this article, I explain how we successfully addressed this challenge.

## Ingestion Design
As I mentioned earlier, we performed categorization based on a function of the record fields. Since the criteria for categorizing an item can be very complex and computationally expensive to solve during a query, we decided to handle this during the ingestion phase. The following diagram illustrates the ingestion pipeline.

![My SVG Image](/evinced/platform_ingestion.svg)

## Raw Query Performance

First, we demonstrated that ClickHouse can handle large queries on our data. I imported a massive dataset via Spark and began testing real-world queries. The results were impressive: on the traffic table, I achieved 393.65 million rows per second and 7.39 GB per second. Most queries returned in under a second, though some complex queries still took 40 seconds. To optimize these, I utilized materialized views, ensuring they stayed up to date thanks to the CollapsingMergeTree. After implementing the materialized views, we reduced the 40-second queries to just a few milliseconds.

This was exciting and promising news. However, one major challenge remained: you need the full state of a record if you want to update it. If you don't have the state, you must also read from ClickHouse, which will result in numerous point queries. In the next section I address how this limitation was solved.

## Update Design

We decided to experiment with the idea of using the same pipeline as the ingestion for updates as well. The solution we developed was to make the updates asynchronous using an orchestrator that schedules which updates can run in parallel. Given that our product requires infrequent updates, handling these occasional requests with a scheduler was acceptable. The orchestrator would schedule a worker process that streams data from ClickHouse, applies the updates, and then streams the data back into the ingestion pipeline. This pipeline ultimately inserts two records for each updated entry: one with the original state to update the chain of materialized views reflecting the record's removal, and one with the new state.

The following diagram depicts the flow:
![My SVG Image](/evinced/platform_update.svg)

The key aspect of this design is its ability to perform streaming updates instead of batch updates. This approach allows throughput throttling and significantly reduces memory requirements. The streamer service was developed using the Go native ClickHouse driver, which supports streaming with large SELECT statements. 

Because the order of messages in our update pipeline could not be guaranteed, we chose the VersionedCollapsingMergeTree over the CollapsingMergeTree. This choice ensures that the correct record is matched regardless of the order in which the messages arrive.

Additionally, failure recovery measures were implemented to ensure the system could recover from any failed updates. In testing, I successfully updated a billion records within minutes, thanks to the service being in the same region as the ClickHouse host. This was remarkable performance.

## Duplicate Message Resilience
One of the requirements was for the pipeline to be deterministic. To achieve this, the update pipeline was designed to be idempotent—well, sort of...

### PubSub
We use Google Pub/Sub as the message queue between our microservices. To achieve resilience against duplication in the pipeline, we configured Pub/Sub with a long acknowledgment timeout. We also ensured that each microservice doesn't hold onto a message for too long. However, this does not guarantee exactly-once delivery; for example, a service could crash or be terminated unexpectedly. To mitigate expecetd service termination, we designed our microservices with graceful shutdowns to avoid leaving tasks partially completed.

With these preventive measures in place, we were confident that the probability of duplicate messages was low, and we considered this an acceptable risk.

### ClickHouse
What was left is to make writes to clickhouse idempotent.
To achieve this, the write to Clishouse was done syncronizly. Clickhouse doesnt support transactional commits becasue it is an OLAP db, but what it does offer is duplication block detection.
Clickhosue uses a blockID,  which is a hash of the data in that block/batch. This block_id is used as a unique key for the insert operation. If the same block_id is found in the deduplication log, the block is considered a duplicate and is not inserted into the table. 
That took care of the traffic table write to be idempotent.

What was left was to ensure that the materialzed views behave the same. Clickhouse provides a lot of settings to fine tune block deduplication and one of them is deduplicate_blocks_in_dependent_materialized_views. By simply enabling this settings, we could ensure that the aggreagtion tables will not be affected by duplicate blocks. 

To leverage this feaure of clickhouse, I designed the sink so that if the batch write fails, the exact batch will be retried until it succeeds before it moves on to the next batch. This ensured that the same block_id will be generated and a duplicate block will be discarded by clickhouse. Also we ensured that all our aggreagte functions we used were deterministics.

### Update
To achieve idenpotency on the update pipline, the message produced by the streamer is a record pair, one for the removal of the current record and one for the new record. The sink then batches messages together, garenteing that a write to the database will contain both records in the pair. This removes to risk to have unbalanced record in the colpsingMergeTre, meaning that you cannot if a remove record without an insert. This alone of course doesn't make the update pipeline compeltely resilient against deduplicate message delivery, but this poses the same risk for incorrect aggregation updates via the materialized views with the ingestion pipeline. So again, this risk was accaptible for us.

On top of this, the select statement was enforced to have a upper ingestion timestamp in the where clause. This kept the amount of  records streamed deterministic, which meant we could predict how many records will be updated and if the update failed, it could be retried.

DIAGRAM


## Replay Ability
As with any pipeline, it's important to have the ability to replay specific data chunks and reprocess them if needed. To address this, the raw input was saved in Parquet format, partitioned by tenant and date.

![My SVG Image](/evinced/platform_replay.svg)

## Query API Design
To meet the requirement for a flexible API, a GraphQL engine was designed that translates GQL into ClickHouse queries. All complex query logic is encapsulated within ClickHouse views, and the GraphQL layer simply reflects the view fields, providing filtering, sorting, and aggregation capabilities on those fields. This design allows us to easily create a new view which will automatically be add to the API. The meta data can be trieved easily with a GraphQL introspectin query.

### Optimizing Queries per tenant on the Traffic table
One thing to realize about clickhouse table engine CollapsingMergeTree is that it lazily apply the colapsing in the background. This means that if you want the updated records you need to tell clickhouse in the query to apply the collapse immediatly by providing the FINAL keyword. With tests we saw a signaficant degradation of performance when we put the FINAL keyword in the queries.

Remember that I said that the updates per tenant are infrequent. We used this fact to our advantage. Whenever a tenant triggered an update, we marked the tanant as dirty in the API service.
We then went ahead and created a cron job that force the optimization on a time where we experience low amount of queries and ingestion.
``` SQL
OPTIMIZE TABLE my_table FINAL;
```
With this in place, when a tenant make a query, if the teant is not marked as dirty, we dont need to include the FINAL keyword in the query.

## Conclusion
The solution chosen for this project was to use ClickHouse's CollapsingMergeTree in combination with the update streaming solution. Thanks to ClickHouse's idling capabilities and scalability, we were able to start with a very small cluster, significantly reducing startup costs compared to competitors.

Back To [Home](../index.md)

