---
layout: default
title: Let's Get Checked - Data Engineer
comments: False
---

# Let's Get Checked - Data Engineer / Challenge

> Problem statement ABC company wants to provide reports to the end customer
> from one time daily to real time.
> 
> 1.The data required for the report is stored in MSSQL, large data files and
> Google Analytic.
> The schema for the above is very different, but few data points are common
> and can be linked.
> They use AWS as a cloud provider and want to make sure the data is secure
> while at rest or in transit.
> 
> 2.Event driven architecture is also something on their roadmap.
> 
> 3.The data from the above sources are also required for use of Regulatory reporting,
> Data Science and BI reporting.
> 
> Draw an approach / solution / architecture design that helps ABC company to build
> reports in real time. Outline the tools that you would like to use:
> OpenSource, SAAS tools, anything that you think will help the company to
> address the immediate need but also prepare for scaling in future. Explain
> the advantage of using the proposed architecture

***


## Assumptions / missing information / miscellaneous notes

 - For the purpose of this exercise I will assume the absence of PII (personable identifiable information) data and that GDPR (and similar legislation) is a non-goal - in practice this will must certainly have to be addressed somehow but it's context dependent).
 - I'll just talk about the production environment, the others will have similar structures, with possible variations in sizing.
 - There is no mention of data volumes, ingress rates and retention periods. I'm happy to discuss how changes in these factors can potentially change the solution.
 - No information is given about the data model, in particular no information about "temporal dimensions". These can affect stuff like partitions schemes and aggregation granularity.
 - There is no information on the immutability of data in MSSQL, is it append only or are any deletes/updates? Do we have to keep a "consolidated view" of the data? Do we need to "see data as it was" in any arbitrary point in the past?
- "few data points are common and can be linked" ... I will assume this relation is 1:1.
- at scale the way we store data it very closely related with how we expect to query it, the solution described bellow can satisfy what I understand of the requirements but can be suboptimal for other scne

## Bird's-eye view

The solution proposed below is not exhaustively described, it merely aspire to be a starting point for a discussion where we can go into more detail.

The primary functionality of the solution depends on the following components:

 - [Apache Kafka](https://kafka.apache.org/) for event streaming
 - [AWS S3](https://aws.amazon.com/s3/) for storage
 - [Debezium](https://debezium.io/) for streaming database changes to Kafka
 - Custom developed proxy service to GA that uses the [Google Analytics Measurement Protocol](https://developers.google.com/analytics/devguides/collection/protocol/v1)
 - [AWS Athena](https://aws.amazon.com/athena/) (managed PrestoDB) for ad-hoc SQL data exploration.
 - [Apache Druid](https://druid.apache.org/) for real time aggregation and analytics
 - [Metabase](https://www.metabase.com/) for data visualization, reporting and exploration.
 - [Spark]() / [Flink]() / [](https://ksqldb.io/) for streaming ETL


In order to guarantee the minimum requirements with regard to information security, the following measures must be implemented:

 - Configure endpoints to only allow authenticated access
 - Enable audit-logging for security related events (login failures, and declined calls)
 - "Encryption at rest" using file system encryption capabilities of the host server
 - Ensure privacy/integrity of data using end-to-end SSL


 I propose to create an Event driven architecture from the start.
 There are 3 sources of data and all of them can be "instrumented" to publish "on change" events
 
  - the large data files (LDF) can be stored in S3 and updates can be triggered via [S3 Event Notifications](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html)
  - using Debezium [changes to MSSQL](https://debezium.io/documentation/reference/1.4/connectors/sqlserver.html) can be published to Kafka .
  - the apps/websites that send events to GA should point to a proxy service that uses the GA Measurement Protocol to send events server-side, this way the raw data can be captured, published to kafka and then forwarded for GA proper.

With these primitives in place we can join streams/static data using Spark/Flink and re-publish the "enriched" data back into Kafka and use Druid support for kafka ingestion and realtime aggregation to create a scalable OLAP cube (of sorts).

In parallel we can also Kafka connectors that drain the above mentioned Kafka topics and stores the raw data in S3 in a query friendly format (e.g.: Parquet). Athena tables can be mapped to these files in order to provide ad-hoc SQL exploration of the raw data. The S3 stored raw data can serve as input for Data Science / Machine Learning pipelines and also satisfy any regulatory request.


With regards to reporting there are many solutions available. I suggest the use of Metabase but it's a very "soft" suggestion. Metabase is an OSS product that can easily connect to both Athena and Druid and allows the creation of dashboards and reports that I imagine to be adequate to this use-case... but it's one of many solutions available, others maybe better suited


As for the advantages of this solution, I'm not sure what to say... there is no frame of reference... advantages compared to what?
I think it fulfills all stated goals.

--

Luis Neves
