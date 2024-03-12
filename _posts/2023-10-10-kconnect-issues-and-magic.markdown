---
layout: post
read_time: true
show_date: true
title:  'Kafka Connect: A love/hate relationship'
date:   2023-10-10 12:00:00 -0600
description: A write up on what makes Kafka Connect such a pain to work with - yet so useful in the building of real time data pipelines. We'll discuss the Postgres connector as the main example, and provide some gotchas that are general to the tool.
img: posts/connected.svg
tags: [kafka, streaming, event sourcing, connect, databases]
author: Abraham Leal
github:  abraham-leal
mathjax: yes
---
Apache Kafka is the de-facto streaming platform for businesses today, and with it gaining popularity, so has the associated subproject "Kafka Connect". Kafka Connect allows to use pre-made "Connectors" to send and receive data to/from Kafka. They require no code, it is all configuration driven. This provides a lot of advantages and a lot of headaches. In this blog we'll talk about both, how to get around them, and hopefully there will be enough information to help you make good decisions around its usage.

We will use the community's [Debezium Postgres Connector](https://debezium.io/documentation/reference/stable/connectors/postgresql.html) to talk about sourcing data from Postgres to Kafka, and Confluent's [JDBC Sink Connector](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc) to talk about sending data from Kafka to another database. 

## A primer on how Kafka Connect works

The advent of Cloud has abstracted a lot on how things run, and [Confluent Cloud](https://www.confluent.io/confluent-cloud/) does a good job at hiding the complexity. However, I believe understanding how something works really allows you to make the best decisions around whether it is a good fit for you. So here is a quick primer:

1. Kafka Connect is a distributed system, separate from Kafka, but powered by it
2. *Kafka Connect* is a platform, *Connectors* run on it
3. Kafka Connect has *source connectors* which bring data from systems into Kafka, and *sink connectors* which send data from Kafka into end systems 
4. Kafka Connect uses an internal topic in Kafka to coordinate source connectors, and the consumer group protocol to coordinate sink connectors
5. Connectors aren't made by one organization, they are made by the community, and the community is strong - providing 100s of connectors for organizations to use. However, this also means that quality and feature sets vary greatly

![Kafka Connect Overview](assets/img/posts/KConnectOverview.png)

That should give us a good enough base to start talking about how things work. Lets get started.

## Source Connectors

### On sourcing from databases

Postgres is a very popular database out there, which makes the Postgres connector a popular connector as well. Thankfully, the community has rallied around [Debezium](https://debezium.io) as the project to congregate and work on very important connectors, and Postgres is one of them.

Sourcing from databases is hard, if you take some time to look into how [Debezium has standardized](https://github.com/debezium/debezium) in reading from WALs, you'll realize is no joke to ensure no data loss, no data duplication, as well as high uptime. Once you've made this realization we are set: We need the community's standard connector to send data from Postgres to Kafka instead of doing our own thing and most likely not do as good of a job (I've seen plenty of companies that do this, for some reason). Now that we've made the commitment to use the connector, we have to understand it. How does it all work?

To read data from a database, it would be extremely inefficient and wasteful to issue query reads and then send that data back to Kafka (although there is a [connector](https://www.confluent.io/hub/confluentinc/kafka-connect-jdbc) that does that if you wish) so instead, what most connectors will do is listen to the Database's Write Ahead Log (WAL). Some DB's provide direct and easy access to their WAL, which is nice! would be so [jerkish not to!](https://docs.oracle.com/en/database/oracle/oracle-database/18/upgrd/deprecated-features-oracle-database-12c-r2.html#GUID-87A754A3-AC6B-4F84-8C36-AB90AC5032D4) Anyways, Postgres does, and it does so through "logical decoding" (their fancy name for it). Here lies our first blessing / problem.

#### Reading from WALs

There are 3 common problems that plague users of Kafka Connect whenever using a Change Data Capture (CDC) connector such as the Debezium Postgres connector:

- Old version of Postgres: Postgres has been around for a while, but logical decoding hasn't, and formal first-class support is even more recent. Users of old versions of Postgres will have to utilize a community decoder plugin. These have been found to have [many issues](https://issues.redhat.com/browse/DBZ-2175?jql=project%20%3D%20DBZ%20AND%20resolution%20%3D%20Unresolved%20AND%20text%20~%20%22decoderbuf%20OR%20wal2json%22%20ORDER%20BY%20priority%20DESC%2C%20updated%20DESC), and some of them have workarounds (too many for us to get into this post), but this will surely slow down deployment speed and resilience. Worth nothing that newer versions have a standard decoder called `pgoutput` that performs very well and is backed by the project as a whole.
- Size of the WAL: Obviously databases have to be space conscious, and they don't keep the WAL forever; However, in streaming we've directly plugged into it and one day the connector might fail but the database will keep chugging along... which means that when the connector comes back, it has to catch up. What if data that it needs to catch up to has already been deleted by Postgres? Using the Postgres also means negotiating with your DB Admin how long the WAL can be kept, and that becomes the maximum amount of time you have to resolve an issue with the connector... before you have to start recovery actions.
- Most source connectors don't offer exactly once ([yet](https://issues.redhat.com/browse/DBZ-5467)): This is true for [Debezium connectors](https://issues.redhat.com/browse/DBZ-5467), which means that when it all goes wrong, you will have duplicates (but no data loss, as they do offere at least once). For _MOST_ applications, this is sufficient, but a lot of people go into this connector expecting exactly once deliveries.

The above issues make the primordial task of the connector (getting data out of the system) quite the math equation that has to be solved. What is important is that it CAN be solved, but does require careful tuning and testing. This pain get exacerbated  by the very common situation that the people developing data streaming flows aren't DB Admins, so they would like not to worry about these.

Anyways, you've tamed the sourcing of the data, then you need to ensure it is actually sent in a way that is useful, which is where pain #2 pops up.

#### Delivery and interpretation of database records in Kafka

Data Types. They make a developers life harder and easier at the same time. Can't live without them. 

When sourcing data from Databases in order for it to be usable somewhere else, you can imagine maintaining data types or at least converting them properly is paramount. With such requirements, the Kafka community has been evolving in their idea on how to handle these, here are some challenges that come with the territory:

- Data interpretation and evolution is hard: Everything that goes into Kafka is serialized as a byte array, which means that when read it requires a way to deserialize it. By default, Kafka provides a JSON serializer, but as you probably know, sending large amounts of data with naive JSON serialization is actually really inefficient. Confluent has had the [Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html) for a while to address this issue as well as Data Governance and Data Quality problems. However, there are various solutions in the community that aim to help solve these problems. (I go quite into depth on this issue in a [podcast](https://www.youtube.com/watch?v=ZIJB8cs-cJU) with the great [Kris Jenkins](https://www.youtube.com/@DeveloperVoices)). This is actually a multi-tiered issue:

1. When consumed from the Database, data comes with the DBs data type
2. When converted into a Kafka Connect record, it gets casted into a set of types pre-defined and available in [documentation](https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-data-types)
3. When sent into Kafka, the data is type-less
4. When consumed, you have to cast again, either to connect's internal data types (which shouldn't be a problem since it was sourced with connect) or to a data type that is useful for the application. ksqlDB [famously does not](Ihttps://docs.ksqldb.io/en/latest/reference/sql/data-types/) have good compatibility with all the possible data types a DB may have.

Long story short, be careful around what your data gets converted to, and ensure you are comfortable with the casting. There are various connector configurations to change the behavior here, specially around time (always the easiest problem to solve in production systems :) /s) I highly recommend utilizing Schema Registry here.

- In-flight transforms: One of the more useful features in Kafka Connect is the ability to modify every record that passes through it slightly to fit your needs. These transformations are pluggable and the community has a sprawl of them. [Debezium has some that are very helpful](https://debezium.io/documentation/reference/stable/transformations/index.html). One of the problems that get developer teams is that because of how useful it would be to just modify data on the fly before even storing it, they try to fit all data transformations in these and you can get pretty creative given correctly chaining transforms and predicates. However, the reality is that these are meant to be lightweight because they can severely hamper performance of the connector. If you really need involved transformations I would try Kafka Streams, Apache Flink, or ksqlDB.

- Scalability: More often than not, the connector will be faster than whatever system you are sourcing from, which is mostly great! However, something to note is that most source connectors will be limited to 1 task (tasks are the unit of parallelism in Kafka Connect) due to ensuring total order of updates from the database. This may hamper you, specifically if you have expensive transforms in it. A good rule of thumb is to separate table updates into various connectors in order to ensure being able to keep up for high traffic databases.


#### Maintenance

Long running source connectors can come into issues as your organization evolves. These are largely rare as long as there isn't anything funny going on at the source datastore (like constant failover tests and whatnot). Commonly, the source connector maintenance story largely consists of:
- Rehydrating every now and then
- Re-running the connector when it fails due to network or connectivity issues
- Scaling the connector due to more data being pumped through

##### Rehydrating

This has become such a common occurrence that most Debezium connectors now implement snapshots while running as a common operation that can be triggered by command messages to the connector. Rehydration is necessary in cases where the target datastore of the data loses it, and you must reload all the data from the source db to bring the complete state to target (this only happens if you don't leverage compacted topics, but thats another story).

If you are using the aforementioned Debezium connectors, you are likely in a good spot to do this easily. Other connectors don't allow this very easily unless you stand up a completely new connector, due to how hard it is to modify source state progress in Kafka for source connectors.

##### Re-running the connector

This is the most commonly scripted mainteinance operation out there. Connectors fail all the time due to network issues. The reality is that unreliabile networks is the only reliable truth in the networking realm. Projects out there like Confluent for Kubernetes and Strimzi provide auto-restart capabilities in their K8s operators. Do yourself a favor and implement this from day one.

##### Scaling the connector

Scaling the connector in tasks up and down is likely the piece of maintenance you'll face the most. Most of the community would like connectors to have policy-based scaling, Kind of what KEDA did for Kafka Clients. The project doesn't address this, and neither do commercial offerings out there. I'd recommend making the application team own scaling actions of the connector, or implement your own scaler that listens for connector metrics and scales up/down though them.


#### Last note on source connectors

I know it seems like I am only talking doom and gloom, but connectors are one of the best things to happen to real-time data processing. I'm trying to highlight that they are unfortunately not that easily to productionalize, but once you got them down, they work really, really well. 

Connectors: High upfront work, medium maintenance (relatively), but doing it yourself would be worse.

## Sink Connectors

### Message Interpretation

The #1 problem that surfaces with sink connectors is data interpretation. It is common for developers to try to deserialize the message in a format it didn't come in, or to try and sink a message with a schema that isn't supported by the target system. For both these issues, I highly suggest integrating with Confluent's Schema Registry from the get-go to avoid most of the problems. 

Data SerDes is always a big topic when I speak with a customer starting their data streaming journey, my recommendation is always (in this order):

1. If your company already has a preference between Avro/Protobuf/Json, stick with that
2. If there is no preferece, use Protobuf
3. If you can't use Protobuf, use Avro
4. If you can't use Protobuf or Avro, might as well give up, because JSON Schema is trash

Now that you have a well defined schema, I'd look into what the conector expected the schema of the message to be. This is basically different for every single sink connector, so research it well, and if you need to modify data, leverage SMTs or a stream processor like Kafka Streams/ksqlDB/Flink.

### Target system success/failure tracking

A lot of organizations would like to keep track of whether a record that was consumed by a connector was successfully delivered to the target system, or if it wasn't, why not? the Kafka project sought to address this by [KIP-298](https://cwiki.apache.org/confluence/display/KAFKA/KIP-298%3A+Error+Handling+in+Connect) then expanding in [KIP-610](https://cwiki.apache.org/confluence/display/KAFKA/KIP-610%3A+Error+Reporting+in+Sink+Connectors) but keep in mind, while KIP-298 work is available in all connectors because its error reporting by the franework, KIP-610 is an optional implementation for the connector and not all connectors offer it.

A big builder of connectors in the Community is Confluent, and they sought to address this faster with their Connect Error Reporter which was implemented before KIP-610 ever made it to the project. This reporter is largely better than the framework implementation, but it is not avaialble in all connectors, and certainly not in all Confluent connectors. YMMV, but keeping these options in mind when using a sink connector is important. To summarize:

- All connectors have KIP-298
- SOME connectors have KIP-610
- SOME Confluent-built connectors have the Confluent Error Reporter

### Optimization for large datasets

A very common scenario is that organizations will the connector consume the data from Kafka very fast, but the insert/upsert to the targer is extremely slow, causing an extremely slow pipeline, or even worse, a timeout of message delivery that causes the connector to be considered dead by the poll loop constraints. 

There are a few ways to mitigate these issues, at a high level:

- For database target systems, ensure you implement indices to make upserts faster
- For http endpoints, ensure the target system can access parallel requests and that you are batching these in an efficient manner (Hello Salesforce Sinks)
- For vendor end systems that don't behave like a datastore, it ultimately ends upbeing a game of slowing down the connector to match what the end system can tolerate, and batch as efficiently as possible to ensure high throughput of data transfers. You can slow down the connector by tunning fetch settings of the underlying consumers of the connector, and batch better by tunning the underlying delivery system the connector uses if the connector makes that available.

### Last note on sink connectors

Sink connectors are extremely useful, and to be honest, often more rescillient in terms of maintenance than source connectors, so they are quite nice. Their state tracking is mostly through the kafka consumer group offset management system, which makes it a breeze to modify the starting point.

A common request i see is how to do data completenes reconcilliation between the data extracted at the source and delivered at the target. This is a big topic in distributed systems with hard answers, but commonly a flavor of Distributed Tracing is used to ensure you have a reliable reconcilliation story.

## Summary

We have discussed the Kafka Connect framework, source connectors, and sink connectors. Overall, Connectors are extremely useful, but they don't come without having to consider how they affect source and target systems, and are limited to how they expect data to be handled. However, they enforce best practices implementation on integrating systems from and to Kafka, so they are overall an excellent tool for organizations with Apache Kafka in their backend.
