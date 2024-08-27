# Data Pipeline Design
This was a particularly interesting project. The requirements involved handling traffic from various sources, including a scraping service that generated massive bursts of traffic. Additionally, there was a need to update a large number of records, allowing users to categorize items into groups based on an arbitrary function of fields, as well as label certain items or records. Moreover, query response times needed to be sub-second. The expected traffic volume ranged from 1 to 5 billion records per day. The company’s cloud provider is GCP, and the solution had to be easily deployable and hosted on this GCP.

## Database Selection
In order solve this, I had to make an extensive research on Database technology. 
The following databases were shortlist from a huge list:
- Apache Pinot with their star tree index
- Clickhouse cloud
- Rockset
- Apache Druid
- Firebolt

After further refining my options based on various criteria, I decided to conduct two proof-of-concepts (POCs) to compare Apache Pinot and ClickHouse:

- Apache Pinot: What stands out about this technology is its unique index called the StarTree index, which is essentially a much leaner OLAP cube inspired by Iceberg design concepts. However, it doesn't support updates, as the index can only be based on immutable facts. Although Pinot has strong update capabilities, this requires the index to be kept entirely in memory. While there's an option to store part of the index on disk, this approach seemed inelegant in design. Moreover, we couldn't utilize the StarTree index, which diminished its appeal for this project.
- ClickHouse Cloud: Clickhouse.com offers a managed ClickHouse database with an exclusive cloud engine called ShardMergeTree, which handles sharding and scaling. Additionally, it supports an idle mode where costs nearly drop to zero while keeping the service running. ClickHouse also features an engine called CollapsingMergeTree, which appears ideal for the company's update needs. This engine allows real-time materialized views on your traffic/fact tables. However, to maintain the accuracy of materialized views, you must provide the full state of the record you want to update so ClickHouse can correctly adjust the aggregated data. Given this significant constraint, I decided to explore whether I could overcome this limitation ehich I described in the Update Design section.

## Ingestion Design
Every item can be categorized. Because the criteria for categorizing an item can be very complex and expensive to solve on a query, I decided to solve this in the ingestion phase.

![My SVG Image](/evinced/platform_ingestion.svg)

## Query Performance

First, I demonstrated that ClickHouse can handle large queries on our data. I imported a massive dataset via Spark and began testing real-world queries. The results were impressive: on the traffic table, I achieved 393.65 million rows per second and 7.39 GB per second. Most queries returned in under a second, though some complex queries still took 40 seconds. To optimize these, I utilized materialized views, ensuring they stayed up to date thanks to the CollapsingMergeTree. After implementing the materialized views, I reduced the 40-second queries to just a few milliseconds.

This was exciting and promising news. However, one major challenge remained: you need the full state of a record if you want to update it. If you don't have the state, you must also read from ClickHouse, which could result in numerous point queries—a significant drawback for ClickHouse. In the next section I address how this limitation was solved.

## Update Design

The solution I developed was to make the updates asynchronous using an orchestrator that schedules which updates can run in parallel. Given that our product requires infrequent updates, handling these occasional requests with a scheduler was acceptable. The orchestrator would schedule a worker process that streams data from ClickHouse, applies the updates, and then streams the data back into the ingestion pipeline. This pipeline ultimately inserts two records for each updated entry: one with the original state to update the chain of materialized views reflecting the record's removal, and one with the new state.

The following diagram depicts the flow:
![My SVG Image](/evinced/platform_update.svg)

The key aspect of this design is that it performs streaming updates rather than batch updates, significantly reducing memory requirements. The streamer service was developed using the Go native ClickHouse driver, which supports streaming with large SELECT statements. Additionally, failure recovery measures were implemented to ensure the system could recover from any failed updates. In testing, I successfully updated a billion records within minutes, thanks to the service being in the same region as the ClickHouse host. This was remarkable because it ensured that the aggregation tables were always up to date.

## Replay Ability
As with any pipeline, it's important to have the ability to replay specific data chunks and reprocess them if needed. To address this, the raw input was saved in Parquet format, partitioned by tenant and date.

![My SVG Image](/evinced/platform_replay.svg)

## Item Deduplication
Item deduplication was added. Details can be seen on this page: [Deduplication Of Items](./platform_deduplication.md)

## Query API Design
To meet the requirement for a flexible API, I designed a GraphQL engine that translates queries into ClickHouse queries. All complex query logic is encapsulated within ClickHouse views, and the GraphQL layer simply reflects the view fields, providing filtering, sorting, and aggregation capabilities on those fields. This design allows us to easily create a new view whenever a new API is needed!

## Conclusion
The solution chosen for this project was to use ClickHouse's CollapsingMergeTree in combination with the update streaming solution. Thanks to ClickHouse's idling capabilities and scalability, we were able to start with a very small cluster, significantly reducing startup costs compared to competitors.

Back To [Home](../index.md)
