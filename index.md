---
layout: default
title: Marais Krüger
---
# About Me
I’m a seasoned Big Data Engineer with a decade of experience in data driven design, specializing in designing and optimizing high-performance data pipelines. My work spans database technologies, distributed systems, and cloud architecture, delivering B2B SaaS solutions used by industry giants like Salesforce, Comcast, Netflix and Amazon.

My passion lies in distributed systems—an area I explore deeply through continuous learning. I actively pursue advanced knowledge through online courses, including MIT's Distributed Systems, staying at the forefront of technological advancements.

Beyond my professional endeavors, I’m a recognized contributor in the Apache Pulsar community and enjoy sharing my expertise. In my spare time, you’ll find me on the tennis court, mountain biking, or socializing with friends.

Let’s connect and explore how we can innovate and transform together:
- [linkedin](https://www.linkedin.com/in/marais-kruger-a5b94214/)
- marais.kruger@gmail.com

# Main Projects
I have highlighted only a few of the projects I've encountered in my career, but I believe these are particularly significant in showcasing my experience as a Data Engineer:
## Evinced: Digital accessibility software
I am the architect and lead data engineer that designed the following projects:
- [Platform Data Pipeline](./evinced/platform_pipeline_medium.md): This data pipeline is part of a scraping system designed to collect accessibility data for compliance checks. It is built to handle significant traffic surges and can scale dynamically according to demand. The pipeline performs complex data transformations, resulting in aggregated data that can be queried via GraphQL. It processes millions of messages per minute and is capable of scaling to handle even greater traffic in the future. Key features include [data deduplication](./evinced/platform_deduplication.md) and complex data grouping, all implemented with a focus on minimizing cloud costs.
- I designed a new pipeline using Airflow for the company's research team, specifically tailored for machine learning tasks. This pipeline was challenging as it required spawning separate workers to scrape thousands of URLs simultaneously. The data was materialized into BigQuery, meeting all the research team’s query requirements. Special care was taken to enforce filters on the partition fields to reduce query costs.
- Researched and designed the pipeline for generative AI needs.

## Cynet Security: All-in-One Cybersecurity Platform 
I was the Team Lead and Principal Data Engineer where I led the design for the follwing projects:
- A XDR (detection and response) data pipeline. The pipeline was created from scratch. It includes transforming data so that it can be queried very fast. The pipeline includes a sophisticated event system that raises alerts based and a highly generic configuration of sequence of events. The pipeline stores data in various databases in order to fulfill the different query types required from the product and research team. The research includes transforming data for machine learning needs.

## Pyramid Analytics: BI Product listed in the Gardner Magic Quadrant
I was team lead and Pricible Software Engineer of the Query Logic team. I led the design of the following projects:
- Designed a proprietary query language that rivals platforms like PowerBI and Tableau. The system starts from a visual representation and then translates into a proprietary query language. This language then translates to various SQL dialects that include most databases out there. The language includes a multi dimensional ability (similar to MDX) that answers hard to solve BI queries.
- On top of this I created a NLP layer that translates natural language into our query language.


