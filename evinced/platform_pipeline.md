# Evinced Data Pipeline Design
This was quite an interesting project. 
Requirements was to receive traffic from various products, including a scraping and service sending huge burst of traffic. Further more there was a need to update huge amount of records where users can categorize items into groups driven from an arbitrary function of fields and also label certain items/records. Further more query Tims must be sub second. The expected traffic is between 1 to 5 billion records a day. The company cloud provider is GCP and preferably had to find a solution that can easily be provided/hosted.

## Database Selection
In order solve this, I had to make an extensive research on Database technology. 
The following databases were shortlist from a huge list:
- Apache Pinot with their star tree index
- Clickhouse cloud
- Rockset
- Apache Druid
- Firebolt

After filtering out more by various criteria, I decided to do 2 POCs on Pinot and Clickhouse and compare:
- Apache Pinot: What is fascinating about this technology is the special index they have called a starter index. This is basically a much leaner OLAP cube inspired by Iceberg design concepts.
The problem with this is that it doesn’t support updates, the index can only be based on immutable facts. Pinot has a strong update capability, but this meant that it requires memory to keep the index completely in memory. They have the option to keep a part of the index on disk, but it dit not feel elegant in the design. Further more, we could not use the StarTree index which lost the appeal for this project.
- Clickhouse Cloud: Clickhouse.com is a managed Clikhouse database. It has an Engine that is only available in the cloud called ShardMergeTree. This service takes care of sharding and scaling. They also support an idle mode where your cost goes close to zero to keep the service up. Further more, clickhouse has an engine called the CollapsingMergeTree which seems ideal for our update needs. The benefit of this engine is that you can have real time materialized views on your traffic/fact table. The problem was that if you want to keep your materialized views accurate, you need to provide the full state of the record you want to update in order for clickhouse to remove stats from your aggregation tables. With this huge constraint in mind, I decided to see if I can somehow overcome this limitation.

## Ingestion Design
Every item can be categorized. Because the criteria for categorizing an item can be very complex and expensive to solve on a query, I decided to solve this in the ingestion.

![My SVG Image](/evinced/platform_ingestion.svg)

## Query Performance

First I proofed that clickhouse can handle big queries on our data. I imported a huge set via Spark and started testing real world queries. I got very impressive query results on the traffic table with 393.65 million rows/s., 7.39 GB/s. Most queries were sub second, however there were still complex queries that took 40s! For these queries I used materialized views, remember that I can count on it being up to date because of the CollapsingMergeTree. After creating materialized views I got the 40 seconds down to a few milli seconds.

This was exciting news and things were looking promising. However there were still one big problem to solve which is that you need the full state of the record if you want to update it. This means if you don’t have the state you will need to also read from clickhouse. This could result in a lot of point queries which is a big no for clickhouse.

## Update Design

The solution I came up with was to make the updates async with an orchestrator that schedules which updates can run in parallel. Because our product needs was infrequent updates, it was acceptable to handle such infrequent request with a scheduler.

The following diagram depicts the flow:
![My SVG Image](/evinced/platform_update.svg)

The important part of this design is that it is doing streaming updates and not batch updates, which made the memory requirements minimal. The streamer service was written with the golang native Clickhosue driver which supports streaming with hug select statements. There were also failure measures put in place so that it can recover from a failed update. I conducted tests and were able to update a billion records in a few minutes since the service was in the same region as the clickhouse host. This was amazing news because this meant that the aggregation tables was always up to date.

## Replay Ability
As with any pipeline, you want the ability to replay certain data chunks and reprocess it.
To solve this, the raw input was saved in parquet, partitioned by tenant and date.

![My SVG Image](/evinced/platform_replay.svg)

## Item Deduplication
Item deduplication was added. Details can be seen on this page: [Deduplication Of Items](./platform_deduplication.md)

## Query API Design
To answer the requirement of having a flexible API, I designed a graphQl engine that translate to a clickhouse query. All complex query needs is hidden behind a view and the graphQL simply reflects the view fields with filtering/sorting/aggreagtion abilities on the fields. This means we simply need to create a new view if another API is needed!

## Conclusion
The use of Clickhouse collapsingMergeTree together with the update streaming solution was the solution picked for this project. Because of the idle ability and scaling ability of clickhouse, we could start with a very small cluster and reduce the startup cost significantly compared to the competitors.

Back to [Home](../index.md)
