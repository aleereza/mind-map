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

The catalog also controls access, with identities of its own. A principal is the catalog's identity for a program, such as the sink or Snowflake. The catalog creates and manages it. When you create a service principal, the catalog gives it a client ID and a secret. The program signs in with them through OAuth (open authorization) client credentials, a standard sign-in method for programs.

Permissions reach a principal through a chain. Privileges go to a catalog role, a catalog role goes to a principal role, and a principal role goes to principals. The sink's chain ends in privileges to create tables and write data, often granted together as CATALOG_MANAGE_CONTENT. Snowflake's chain ends in TABLE_READ_DATA only. The [Apache Polaris access control guide](https://polaris.apache.org/in-dev/unreleased/managing-security/access-control/) describes the chain.

If two writers commit to one table at the same time, the catalog accepts one commit and rejects the other, which then retries.

Snowflake Open Catalog is a managed version of Apache Polaris, an open-source REST catalog.

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

Credential vending means that the catalog hands out short-lived storage credentials together with a table. The credentials are scoped to that table's location in S3. So a trusted reader needs no storage role of its own. Table access is then managed in one place: the catalog's roles. A writer can use vended credentials too, or keep its own storage keys.

In Snowflake, the object that connects to an external catalog is a catalog integration. It turns this on with ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS. The default is EXTERNAL_VOLUME_CREDENTIALS, which needs an external volume instead. An external volume is a Snowflake object. It names the S3 location and an IAM (AWS Identity and Access Management) role that Snowflake assumes to read it. The [catalog integration reference](https://docs.snowflake.com/en/sql-reference/sql/create-catalog-integration-rest) lists both modes.

### Catalog-linked database {#catalog-linked-database}

A catalog-linked database is a Snowflake database that mirrors one or more namespaces of a remote Iceberg catalog. Each namespace appears as a schema, and each table appears as an Iceberg table.

Snowflake polls the catalog, every 30 seconds by default. It finds new tables on its own, and it picks up each table's new commits. No table needs its own setup in Snowflake. The [Snowflake guide](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database) describes the options.

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

The read side takes two inputs: each table's current metadata from the catalog, and the files in S3 that the metadata lists. Both ways to bind Snowflake to the catalog take these same inputs: a catalog-linked database, and one Iceberg table per source table. They differ only in how Snowflake gets into S3, as Under the hood shows.

Snowflake reaches the catalog through a catalog integration. This Snowflake object stores the catalog's web address, the catalog's name, and the client ID and secret that Snowflake signs in with. Snowflake signs in to the catalog. It sends the ID and secret to the catalog, gets back a short-lived token, and shows that token on every catalog request. The integration also names the principal role that the token should carry.

The ID and secret belong to a principal: the catalog's own identity for a program, as the [REST catalog](#rest-catalog) concept explains. Snowflake has its own principal, which can only read. The sink has a different principal, which can write.

S3 is a separate sign-in. With vended credentials, the catalog hands Snowflake short-lived S3 credentials with each table. With an external volume, Snowflake assumes the AWS IAM role that the volume names.

#### Outputs {#snowflake-outputs}

The output is a set of Iceberg tables that a Snowflake role can query once it has SELECT. dbt reads them as sources. Snowflake stores none of the rows. A query reads from S3 only the files that can match it. A repeat query can skip S3 in two cases. Either Snowflake has a saved result, or the servers that ran the query still hold the files on local disk. The [Iceberg in Snowflake map](../../snowflake/iceberg/snowflake-iceberg.md) explains both and shows a query step by step.

Catalog names are case-sensitive and usually lowercase. Snowflake turns unquoted names into uppercase, so queries and dbt sources must quote the schema and table names, for example "sensors"."readings".

#### Under the hood {#snowflake-under-the-hood}

Snowflake can bind to the catalog in two ways. This table shows each one as it is usually set up:

| | Catalog-linked database | One Iceberg table per source table |
| --- | --- | --- |
| Snowflake objects | A catalog integration and one database | A catalog integration, an external volume, and one CREATE ICEBERG TABLE per table |
| A new table appears | On the next poll after the sink creates it | Only after the sink creates it and someone registers it |
| New commits appear | On the next poll | Only with AUTO_REFRESH on, or after a manual refresh |
| S3 access | Vended credentials | The external volume's IAM role |

In an ordinary database, CREATE ICEBERG TABLE does not create a table in the catalog. It registers a table that the catalog already has, under a Snowflake name. Until the sink has created that table, the statement fails with "table not found in catalog". So the per-table way needs two steps. First, build the catalog integration, the external volume, and the database. Then register each table after its first record has arrived. A source that sends no records never gets a table, so its registration keeps failing.

For a table of its own, AUTO_REFRESH is off by default, as the [CREATE ICEBERG TABLE reference](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-rest) shows. Then Snowflake never polls the catalog for that table. The table keeps an old snapshot while the sink keeps committing, and nothing reports an error.

For example, suppose the sink creates a new table at 10:00 and commits every 5 minutes. With a catalog-linked database, the table appears in Snowflake by about 10:00:30. Each later commit follows within a minute or so. With one table per source table, the table appears only after someone registers it in Snowflake. If AUTO_REFRESH is off, it then keeps the snapshot it found, while the sink commits 288 times a day.

The per-table way is older and more manual, and Snowflake still supports it. The catalog-linked database removes the per-table setup and the two-step order. It has [limits](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database#considerations-for-using-a-catalog-linked-database-for-iceberg-tables): it works only with an Iceberg REST catalog, and it supports no replication, cloning, or UNDROP. Renaming a table in the catalog drops and recreates it in Snowflake. As of November 2026, one Snowflake reference page still labels the feature as preview.

Choose a catalog-linked database when you want every table in a namespace, kept current with no table-by-table work. Choose the per-table way when you want only some of the tables, or your own names in an existing database. It also fits when you need a feature that the linked database lacks.

Either way, Snowflake does not copy the catalog's permissions. Grant SELECT on future Iceberg tables in each schema, so that new tables are readable when they appear. In a catalog-linked database, a grant on a schema fails until the first sync has created that schema.

## End-to-end concerns {#end-to-end-concerns}

> type: category

These problems belong to the whole path, not to one part.

### Freshness {#freshness}

### Duplicates and updates {#duplicates-and-updates}

### Schema changes {#schema-changes}

### Silent failures {#silent-failures}

### Table maintenance {#table-maintenance}

### Access and identities {#access-and-identities}
