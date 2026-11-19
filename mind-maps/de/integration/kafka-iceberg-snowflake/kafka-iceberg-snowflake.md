# Kafka → Iceberg → Snowflake {#kafka-iceberg-snowflake}

> type: category

How does a database change or an application event become a table that Snowflake can query? A connector writes the records as Iceberg tables in S3. A catalog tells Snowflake which files make up each table, and Snowflake reads those files without loading a copy.

## Why this pattern {#why-this-pattern}

> type: category

The pattern makes two choices. It puts Kafka in front of the tables. It also lets Snowflake read Iceberg tables in place, instead of loading a copy. Each choice solves a real problem.

### Kafka in front {#kafka-in-front}

Sources write only to Kafka, and Kafka keeps each record for days. If a later step stops, the records wait in Kafka and nothing is lost. A buffer with short retention cannot do this. If it keeps events for only one day, every longer outage loses data.

For example, suppose the topics keep records for 7 days, unless a partition reaches 1 GB first. Then the sink, the connector that writes Kafka records into Iceberg tables, stops for 3 days. When it restarts, it reads the 3 missed days and loses nothing. A buffer that keeps events for 1 day would have deleted the first 2 days before anyone read them.

Kafka also keeps the sources simple. A source needs only Kafka credentials to send its data. On the write side, only the sink connectors hold S3 and catalog credentials, and only they write data files and request commits.

### Read in place {#read-in-place}

Reading in place means that Snowflake queries the Iceberg files where the sink wrote them. The usual alternative copies the records into tables that Snowflake owns. Snowpipe (Snowflake's file loader) and the Snowflake Kafka connector both work this way. A common copy setup is an exporter that writes files to S3, plus Snowpipe to load them.

#### What reading in place buys {#read-in-place-buys}

- The table data is stored once, in S3. There is no load step to pay for, and no second copy inside Snowflake.
- Any engine that reads Iceberg, such as Spark, Trino, or DuckDB, can read the same table.
- A ready-made sink connector replaces custom exporter code.
- The table format adds atomic (all-or-nothing) commits, schema evolution, and time travel, as the [Iceberg map](../../iceberg/iceberg.md) explains.

#### What reading in place costs {#read-in-place-costs}

> type: warning

- You run more parts: a sink, a catalog, and separate identities for writing and reading.
- Rows appear one commit at a time, not one record at a time.
- A stopped sink looks the same as a quiet source, so a failure stays hidden until someone checks.
- Small files and old snapshots pile up until a job merges the files and removes the old snapshots.

Each cost has its own node under [End-to-end concerns](#end-to-end-concerns). If Snowflake is the only reader and you want fewer parts to run, a copy is the better choice.

## Main concepts {#main-concepts}

> type: category

### Change data capture (CDC) {#change-data-capture}

Change data capture (CDC) reads a database's own change log. It turns each committed insert, update, and delete into an event. The source tables keep their schema: CDC adds no triggers and no columns.

The result is a stream of changes, not a copy of the table. A row that changes three times produces three events, and a delete produces an event too.

The usual alternatives see less. A scheduled full extract copies whole tables, so it is heavy and always late. A query for rows changed since the last run needs a reliable last-changed column, and it never sees a deleted row.

### Schema registry {#schema-registry}

A schema registry is a small service next to Kafka that stores the schema of each topic's records. A schema lists every field and its type. The registry keeps each schema under a subject, usually named after the topic, such as readings-value for the values of topic readings. Each version gets a numeric ID.

A message carries only that ID, not the schema. In this pattern, messages use Avro, a compact binary format. Each Avro message starts with a 5-byte header: a zero byte, then the schema ID in 4 bytes. The reader fetches the schema for that ID once, caches it, and decodes the rest. So messages stay small, and the sink knows each field's type. It needs those types to write Parquet, a column-based file format with a type for every column.

For example, a message starts with the bytes 0, 0, 0, 0, 7. The sink asks the registry for schema 7 and learns that id is a long, truck is an int, and end is a timestamp that can be empty. It then writes those values into columns of those types.

The registry also guards changes. Before it accepts a new version, it checks the version against the older ones with a compatibility rule. Under FULL_TRANSITIVE, a new version must work for readers and writers of every earlier version. Adding an optional field passes. Changing a field's type, such as from number to text, fails, and the producer gets an error instead of sending the change. So the registry decides which schema changes can ever reach the table.

Karapace is an open-source schema registry that speaks the same API as Confluent's.

### Kafka Connect {#kafka-connect}

Kafka Connect is a service that moves data into and out of Kafka without custom code. It runs as its own set of processes, called workers, next to the Kafka brokers. Its only job is to run connectors.

A connector is a ready-made plugin plus a configuration. A source connector reads an outside system and writes to Kafka. A sink connector reads Kafka topics and writes to an outside system. "Sink" only means the place where data flows to.

Kafka Connect splits each connector into tasks that work in parallel, and it spreads the tasks across its workers. If a worker stops, the other workers take over its tasks. For example, a sink set to tasks.max = 3 that reads a topic with 3 partitions runs 3 tasks, and each task reads one partition. A fourth task would have no partition to read.

In this pattern, both ends use Kafka Connect. The CDC source connector runs in a Kafka Connect service near the database. The Iceberg sink connector runs in a Kafka Connect service in the cloud. When this map says "the sink", it means the Iceberg sink connector and its tasks.

### Single message transform (SMT) {#single-message-transform}

A single message transform (SMT) is a small step that Kafka Connect runs on each record. It runs as the record passes through a connector. A source connector runs its transforms before the record reaches Kafka. A sink connector runs them after it reads the record from Kafka, and before it writes.

A connector's configuration lists its transforms in order. Each transform changes one record at a time: it can add, remove, or rename fields, change the record's topic, or drop the record. It cannot join or count records, because it sees only one record. A later transform can use a field that an earlier transform added, so the order matters.

For example, the sink unwraps the record from the [CDC source connector](#cdc-outputs) with Debezium's ExtractNewRecordState transform. The envelope for the down event becomes one plain row: id 7001, truck 104, start 08:00, end 09:30. The transform also adds the operation and source details as columns, such as __op = u and __source_ts_ms. Two more transforms can then add a namespace, which is a named group of tables in the catalog, and join it with the source table name. The result names the target table, so one sink can route records to many tables.

### REST catalog {#rest-catalog}

A REST catalog is an Iceberg catalog that clients reach through the open Iceberg REST API over HTTPS. Like every catalog, it maps each table name to the table's current metadata file. The [Iceberg map](../../iceberg/iceberg.md) explains that role. Because the API is a standard, any engine with an Iceberg client can use it. Here, the sink writes through it, and Snowflake reads through it.

The catalog also controls access. Each program signs in as a principal, the catalog's own identity for that program, with a client ID and a secret. Roles decide what each principal may do: Snowflake's principal can only read, and the sink's principal can create and write tables. The [Iceberg in Snowflake map](../../snowflake/iceberg/snowflake-iceberg.md) explains principals, roles, and Snowflake Open Catalog in detail.

If two writers commit to one table at the same time, the catalog accepts one commit and rejects the other, which then retries.

### Iceberg commit {#iceberg-commit}

An Iceberg commit publishes new data files as the table's next snapshot. A snapshot is the complete list of files that form the table at one moment.

A commit has two halves. First, the writer writes the data files, plus manifest files that list them. Then it asks the catalog to commit. A REST catalog such as Snowflake Open Catalog writes the new table metadata file itself. Then it moves the table's pointer to that file in one atomic step.

A reader sees all of a commit or none of it. Before the commit, the new files sit in S3, but no reader can see them. So in this pattern, the commit is the moment rows join the table. The [Iceberg map](../../iceberg/iceberg.md) describes the files, and the sink's [commit cycle](#sink-under-the-hood) shows when commits happen.

### Valid-through timestamp (VTTS) {#valid-through-timestamp}

The valid-through timestamp (VTTS) is a watermark: a time up to which the data is complete. The sink writes it into each commit's snapshot summary, a small set of notes stored with every snapshot, under the key kafka.connect.valid-through-ts.

For each topic partition, the sink takes the latest record timestamp that the commit includes. The VTTS is the earliest of those values, because the slowest partition decides what is complete. So every record with a Kafka timestamp at or below the VTTS is already committed.

For example, a commit covers three partitions whose latest records are from 10:04:58, 10:04:59, and 10:03:10. The VTTS is 10:03:10. A model filters on a column that holds each record's Kafka timestamp, and it stops at 10:03:10. A later commit cannot add a row at or below that time, except after a replay of old records. A row timestamp alone gives no such promise, because rows become visible one commit at a time.

Three details matter in practice. The VTTS is missing, not zero, after a partial commit, one that timed out before every task answered. It is also missing when a partition sent nothing in that round. One stuck partition holds the whole table's VTTS back, which looks the same as no new data. And the VTTS only moves forward when the broker stamps each record with its own clock (LogAppendTime). With the producer's clock, it can move back. In Snowflake, ICEBERG_TABLE_SNAPSHOT_REFRESH_HISTORY returns each snapshot's summary, so a query can read the VTTS.

### Credential vending {#credential-vending}

Credential vending means that the catalog hands out short-lived storage credentials, scoped to one table, together with that table. A reader then needs no storage role of its own.

In Snowflake, the object that connects to an external catalog is a catalog integration, and it turns vending on. The other option is an external volume: a Snowflake object that names the S3 location and an IAM (AWS Identity and Access Management) role to assume. The [Iceberg in Snowflake map](../../snowflake/iceberg/snowflake-iceberg.md) explains all three objects.

### Catalog-linked database {#catalog-linked-database}

A catalog-linked database is a Snowflake database that mirrors namespaces of a catalog, and it picks up new tables and new commits on its own. The [Iceberg in Snowflake map](../../snowflake/iceberg/snowflake-iceberg.md) explains it, and compares it with one Iceberg table per source table.

## Data flow {#data-flow}

> type: category

Database row change → CDC source connector → Kafka topic
Application event → event producer → Kafka topic
Kafka topic → Iceberg sink connector → S3 bucket and Snowflake Open Catalog → Snowflake read side

### CDC source connector {#cdc-source-connector}

> type: concept
> edge.flow: kafka-topic

Reads the database's recorded changes and writes one Kafka record for each changed row.

Use a CDC source when the data lives as rows in an operational database, not as events. Equipment lists, status changes, and maintenance records are typical examples. Some machines may have no device that sends events. Then their rows are the only record of which machines exist and how they change. CDC reads these tables without changing them, and it runs on Kafka Connect, the same framework as the sinks.

#### Inputs {#cdc-inputs}

For a SQL Server source, SQL Server's own CDC feature does the first half of the work. An administrator enables CDC on the database and on each table. A SQL Server Agent job then reads the transaction log every 5 seconds. It copies each committed change into a change table. An update writes two change rows: the row before the change and the row after it.

The connector polls these change tables about every 500 ms. One connector can capture many tables.

#### Outputs {#cdc-outputs}

The connector writes one Kafka topic per captured table. Each record's key is the row's primary key. Its value is an envelope with three main parts: the operation, the row before and after the change, and source details. The source details include the table name and the commit time. Key and value are both Avro, with their schemas in the schema registry.

The operation codes are c (insert), u (update), d (delete), and r (a row read during a snapshot). After a delete, the connector also sends a tombstone: a record with the same key and an empty value.

For example, a down event for truck 104 starts at 08:00 with no end time. At 09:30, someone records the end time, and the connector sends this record:

| Part | Value |
| --- | --- |
| key | id 7001 |
| op | u |
| before | id 7001, truck 104, start 08:00, end empty |
| after | id 7001, truck 104, start 08:00, end 09:30 |
| source | table down_event, committed at 09:30:02 |

#### Under the hood {#cdc-under-the-hood}

The connector runs in a Kafka Connect service on a server near the database, inside the source network. It connects out to the cloud Kafka over TLS, an encrypted connection.

On its first start, the connector reads every row of every captured table once, as a snapshot. Then it follows the change tables. In a test, a table of 2 million rows took about 1 minute 50 seconds to snapshot.

The connector stores its position in a Kafka topic, so after a restart it continues where it stopped. This works only while SQL Server still holds the missed changes. By default, SQL Server deletes change rows after 3 days. If the connector stays down longer, the changes in between are lost. Then the tables must be read again in full. [Kafka in front](#kafka-in-front) cannot help here, because those changes never reached Kafka.

CDC does not follow column changes. A column added later is not captured until an administrator recreates that table's capture instance, which is its CDC setup.

### Event producer {#event-producer}

> type: concept
> edge.flow: kafka-topic

An application that writes each event it creates to a Kafka topic.

#### Inputs {#producer-inputs}

#### Outputs {#producer-outputs}

#### Under the hood {#producer-under-the-hood}

### Kafka topic {#kafka-topic}

> type: concept
> edge.flow: iceberg-sink-connector

Holds the records in order until the sink connector reads them.

#### Inputs {#topic-inputs}

#### Outputs {#topic-outputs}

#### Under the hood {#topic-under-the-hood}

### Iceberg sink connector {#iceberg-sink-connector}

> type: concept
> edge.flow: s3-bucket, snowflake-open-catalog

Reads the topics, writes Parquet files to S3, and commits them to the catalog.

#### Inputs {#sink-inputs}

#### Outputs {#sink-outputs}

The sink produces Parquet data files and manifest files in S3, and one commit in the catalog per commit interval. The first record for a table that does not exist yet also creates that table in the catalog.

#### Under the hood {#sink-under-the-hood}

The sink runs as several tasks that read the topics in parallel. Each task keeps writing records into Parquet files, but the sink commits them only on a timer. One task also acts as the coordinator.

Every commit interval, the coordinator sends a start-commit message on the control topic. This is a small Kafka topic that only this connector uses. Each task answers with the files it wrote since the last commit. When every task has answered, the coordinator commits all of those files in one snapshot.

The coordinator waits only until the commit timeout. Then it commits whatever has arrived, and it logs the commit as complete even when files are missing.

For example, suppose the coordinator's own task answers in under a second, and the other tasks need 21 to 48 seconds. With a 20-second timeout, every commit holds only the coordinator's own files. A 90-second timeout includes every task, and it still fits well inside a 5-minute commit interval.

The control topic must have exactly one partition. If two connectors share one control topic, their coordinators mix up each other's messages, and neither finishes a commit.

With auto-create on (iceberg.tables.auto-create-enabled), the sink creates a missing table when the first record for it arrives. It must wait for that record, because the table's columns come from the record's schema. So the first record sets the columns, and it should be a real record, not a test. With schema evolution on (iceberg.tables.evolve-schema-enabled), later records can add columns. A renamed field arrives as a new column, and the old column stays.

Make sure the namespace exists before the first record arrives. If the sink cannot create it, the first record makes the sink's tasks fail. The [Kafka Connect sink guide](https://iceberg.apache.org/docs/latest/kafka-connect/) lists both settings.

### S3 bucket {#s3-bucket}

> type: concept
> edge.flow: snowflake-read-side

Holds the data files and the Iceberg metadata files.

#### Inputs {#bucket-inputs}

#### Outputs {#bucket-outputs}

#### Under the hood {#bucket-under-the-hood}

### Snowflake Open Catalog {#snowflake-open-catalog}

> type: concept
> edge.flow: snowflake-read-side

Holds the pointer to each table's current metadata, and decides who may read or write.

#### Inputs {#catalog-inputs}

#### Outputs {#catalog-outputs}

#### Under the hood {#catalog-under-the-hood}

### Snowflake read side {#snowflake-read-side}

> type: concept

Finds the tables through the catalog and reads their files in place.

#### Inputs {#snowflake-inputs}

The read side takes two inputs: each table's current metadata from the catalog, and the files in S3 that the metadata lists. Both ways to bind Snowflake to the catalog take these same inputs: a catalog-linked database, and one Iceberg table per source table.

Snowflake reaches the catalog through a catalog integration. This Snowflake object stores the catalog's web address, the catalog's name, and the client ID and secret that Snowflake signs in with. Snowflake signs in to the catalog. It sends the ID and secret to the catalog, gets back a short-lived token, and shows that token on every catalog request. The integration also names the principal role that the token should carry.

The ID and secret belong to a principal: the catalog's own identity for a program, as the [REST catalog](#rest-catalog) concept explains. Snowflake has its own principal, which can only read. The sink has a different principal, which can write.

S3 is a separate sign-in. With vended credentials, the catalog hands Snowflake short-lived S3 credentials with each table. With an external volume, Snowflake assumes the AWS IAM role that the volume names.

#### Outputs {#snowflake-outputs}

The output is a set of Iceberg tables that a Snowflake role can query once it has SELECT. dbt reads them as sources. Snowflake stores none of the rows. A query reads from S3 only the files that can match it. A repeat query can skip S3 in two cases. Either Snowflake has a saved result, or the servers that ran the query still hold the files on local disk. The [Iceberg in Snowflake map](../../snowflake/iceberg/snowflake-iceberg.md) explains both and shows a query step by step.

Catalog names are case-sensitive and usually lowercase. Snowflake turns unquoted names into uppercase, so queries and dbt sources must quote the schema and table names, for example "sensors"."readings".

#### Under the hood {#snowflake-under-the-hood}

Snowflake binds to the catalog in one of two ways: a catalog-linked database, or one Iceberg table per source table. The [Iceberg in Snowflake map](../../snowflake/iceberg/snowflake-iceberg.md) compares them and explains how to choose.

In this pipeline, the choice decides how new tables and commits reach Snowflake. The sink creates a table in the catalog when the first record for it arrives. A catalog-linked database finds that table on its next poll. With one table per source table, someone registers the table in Snowflake after that first record. So setup runs in two steps: first the catalog integration, the external volume, and the database, then each table. A source that sends no records never gets a table, so its registration keeps failing.

For example, suppose the sink creates a new table at 10:00 and commits every 5 minutes. With a catalog-linked database, the table appears in Snowflake by about 10:00:30. Each later commit follows within a minute or so. With one table per source table, the table appears only after someone registers it in Snowflake. If AUTO_REFRESH is off, it then keeps the snapshot it found, while the sink commits 288 times a day.

## End-to-end concerns {#end-to-end-concerns}

> type: category

These problems belong to the whole path.

### Freshness {#freshness}

Freshness is how old the newest row is when a query reads it. The delay adds up across four steps:

1. The record waits in Kafka until the sink reads it, usually for seconds.
2. The record waits in a data file until the next commit, for up to one commit interval.
3. The commit itself takes up to the commit timeout.
4. Snowflake picks up the new snapshot at its next poll, every 30 seconds by default.

A dbt model that runs on a schedule adds its own wait on top.

For example, with 5-minute commits, a 90-second timeout, and a 30-second poll, a record can take about 7 minutes to reach Snowflake. An hourly dbt run can add up to another hour.

To measure the delay, compare each snapshot's REFRESHED_ON time in Snowflake with its [valid-through timestamp](#valid-through-timestamp). ICEBERG_TABLE_SNAPSHOT_REFRESH_HISTORY returns both. Alert on the age of the VTTS, because row counts look normal when data is only late.

### Duplicates and updates {#duplicates-and-updates}

The sink appends every record as a new row, so a table can hold several rows for one source row. Three things cause this.

First, CDC sends every change of a row. An insert and a later update arrive as two rows, with the operation in __op and the change's position in the log in __source_commit_lsn. For example, down event 7001 gets one row from 08:00 with no end time, and one row from 09:30 with the end time. A model that wants the current state keeps the latest row for each primary key, ordered by __source_commit_lsn. The two rows differ, so removing identical rows keeps both.

Second, deletes. By default, the unwrap transform turns a delete into a tombstone, which the sink cannot write as a row. With the transform's delete handling set to rewrite, a delete arrives as a row flagged __deleted, and the model can drop that key.

Third, delivery is at least once. A replay after a failure sends records again. In Iceberg sink version 1.10, a change of coordinator can also add the same data file to two snapshots ([Iceberg issue 16282](https://github.com/apache/iceberg/issues/16282)). Models that merge on a key, such as a dbt incremental model with a unique_key, absorb these copies.

### Schema changes {#schema-changes}

A schema change travels through every layer, and each layer has its own rule:

1. The [schema registry](#schema-registry) checks the new version against its compatibility rule. Under FULL_TRANSITIVE, an optional new field passes, and a type change fails at the producer.
2. A CDC source follows a new column only after an administrator recreates the table's capture instance.
3. The sink, with schema evolution on, adds each new field as a new column.
4. Snowflake shows the new column after its next refresh of the table.
5. dbt models use the column once a model selects it.

For example, a producer adds an optional field battery_level. The registry accepts it, the next records carry it, and the sink adds a column at its next commit. Snowflake shows the column within a minute, and the column reaches a mart once a model selects it.

### Silent failures {#silent-failures}

Several failures stop data while every status looks healthy. A query against a stalled table returns the last good snapshot, so readers get old data and no error. These are the usual causes:

- Rejected records. With errors.tolerance = all, a record that fails conversion goes to a dead-letter queue (DLQ), a Kafka topic that holds bad records for review. The pipeline keeps running, so the loss shows only in the DLQ.
- Failed commits. In Iceberg sink version 1.10, a commit that keeps failing is logged only as a warning, and the tasks stay RUNNING.
- Partial commits and a shared control topic. The sink's [commit cycle](#sink-under-the-hood) explains both.
- A Snowflake table without [auto-refresh](../../snowflake/iceberg/snowflake-iceberg.md). Snowflake keeps an old snapshot while the sink keeps committing.

For example, if two connectors share a control topic, the log shows a complete commit every 5 minutes, and each commit holds zero rows.

Detection works on outcomes. Alert on the age of each table's VTTS or latest snapshot, on Snowflake's REFRESHED_ON time, and on any message in the DLQ. Connector status shows only whether the tasks run.

### Table maintenance {#table-maintenance}

Frequent commits leave many small files and many snapshots. With 5-minute commits, each table gains 288 snapshots a day, and each commit adds at least one file per task. Reads slow down as the file count and the metadata grow.

Four jobs keep a table healthy. Compaction merges small files, snapshot expiry removes old snapshots, and orphan-file cleanup deletes files that no snapshot uses. A manifest rewrite tidies the metadata. The [Iceberg map](../../iceberg/iceberg.md) explains each job.

Snowflake maintains Snowflake-managed tables itself. For a table that an external catalog owns, the catalog's owner probably schedules the jobs, for example with Spark's Iceberg maintenance procedures. A maintenance commit can race a sink commit. The catalog then accepts one commit, and the other retries.

### Access and identities {#access-and-identities}

Each hop has its own identity, set up in its own place, so each break looks different:

| Hop | Identity | Where it is set up |
| --- | --- | --- |
| The CDC connector reads the database | A database login that can read the change tables | The database |
| Producers and connectors use Kafka | Client certificates, called mutual TLS, plus schema registry credentials | The Kafka service |
| The sink commits to the catalog | A catalog principal with a client ID and a secret | The catalog |
| The sink writes data files | Storage keys of its own, or vended credentials | The cloud account |
| The catalog writes metadata files and vends credentials | An IAM role that the catalog assumes | The cloud account |
| Snowflake reads the catalog | A read-only catalog principal | The catalog, and a Snowflake catalog integration |
| Snowflake reads the files | Vended credentials, or an external volume's role | The catalog, or the cloud account |
| dbt reads the tables | A Snowflake role with grants on the database | Snowflake |

So the write path uses two storage identities: the sink's, for data files, and the catalog's, for metadata files. A failure in the write path stops new data. A failure in the read path makes existing tables unreadable, while ingestion looks healthy. The [Iceberg in Snowflake map](../../snowflake/iceberg/snowflake-iceberg.md) explains principals, roles, and storage access.
