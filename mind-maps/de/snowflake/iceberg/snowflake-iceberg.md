# Iceberg in Snowflake {#snowflake-iceberg}

> type: category

How does Snowflake work with Iceberg tables, and where does Snowflake Open Catalog fit? A catalog stores one value for each Iceberg table: the location of its current metadata file, called the pointer. Snowflake can hold that value itself, as in a Snowflake-managed table. Another catalog, such as Snowflake Open Catalog, can hold it instead. Snowflake then connects to the table through a catalog integration. Snowflake Open Catalog is a separate catalog service that many engines can share.

## Main concepts {#main-concepts}

> type: category

### External volume {#external-volume}

An external volume is a Snowflake object that points to a location in your cloud storage, such as an S3 bucket. Snowflake uses it to reach the data files and the Iceberg metadata files of a table that lives in your storage.

The volume names an IAM (AWS Identity and Access Management) role in your cloud account. Snowflake assumes that role through its own IAM user, with an external ID that the role's trust policy must list. `DESC EXTERNAL VOLUME` shows both values. `ALLOW_WRITES`, `TRUE` by default, controls whether Snowflake may write through the volume. The [CREATE EXTERNAL VOLUME reference](https://docs.snowflake.com/en/sql-reference/sql/create-external-volume) lists its settings.

### Snowflake-managed Iceberg table {#snowflake-managed-table}

A Snowflake-managed Iceberg table uses Snowflake as its Iceberg catalog. You create it with `CREATE ICEBERG TABLE` and `CATALOG = 'SNOWFLAKE'`. The files need storage: Snowflake-managed storage, or an [external volume](#external-volume) that points to your bucket. The [CREATE ICEBERG TABLE reference](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-snowflake) lists the settings.

Snowflake generates the Iceberg metadata files itself, for version 2 or 3 of the Apache Iceberg specification, and writes them to the external volume. It does this on a periodic basis with the function `SYSTEM$GET_ICEBERG_TABLE_INFORMATION`, and you can run that function on demand. Each new metadata file holds all changes since the previous one. The runs that Snowflake starts appear under the user `SYSTEM` in the query history, and Snowflake deletes old metadata, manifest lists, and manifests after the retention period. The [Metadata and retention page](https://docs.snowflake.com/en/user-guide/tables-iceberg-metadata) describes this.

Snowflake handles all lifecycle maintenance for the table, such as compaction, and you can turn compaction off. The [Iceberg tables overview](https://docs.snowflake.com/en/user-guide/tables-iceberg) describes the two table types.

### Iceberg REST catalog {#iceberg-rest-catalog}

"Iceberg REST catalog" names a protocol, not a product. The Iceberg project defines the [REST Catalog API](https://iceberg.apache.org/rest-catalog-spec/) as an OpenAPI spec. As of November 2026, these products implement it:

| Catalog | Run by | Evidence |
| --- | --- | --- |
| Apache Polaris | Open source (Apache Software Foundation) | The [Apache Software Foundation press release](https://news.apache.org/foundation/entry/the-apache-software-foundation-graduates-two-open-source-projects-from-incubator) says it implements the Iceberg REST API. |
| Snowflake Open Catalog | Snowflake (hosted Polaris) | The [Open Catalog overview](https://docs.snowflake.com/en/user-guide/opencatalog/overview) says it is built on the Iceberg REST protocol. |
| Snowflake Horizon Catalog | Snowflake | The [Horizon docs](https://docs.snowflake.com/en/user-guide/snowflake-horizon) offer an Iceberg REST Catalog API for Spark, Flink, and Trino. |
| Project Nessie | Open source | The [Nessie site](https://projectnessie.org/) has a guide for configuring Nessie with Iceberg REST. |
| Lakekeeper | Open source (written in Rust) | The [Lakekeeper docs](https://docs.lakekeeper.io/) call it a standards-compliant Iceberg REST catalog. |
| Apache Gravitino | Open source (Apache Software Foundation) | The [Gravitino docs](https://gravitino.apache.org/docs/latest/iceberg-rest-service/) say its Iceberg REST server implements the Iceberg REST API spec. |
| Databricks Unity Catalog | Databricks | The [Databricks docs](https://docs.databricks.com/aws/en/external-access/iceberg) say it implements the spec at `/api/2.1/unity-catalog/iceberg-rest`. |
| AWS Glue Data Catalog | AWS | The [AWS docs](https://docs.aws.amazon.com/glue/latest/dg/connect-glu-iceberg-rest.html) show an Iceberg REST endpoint, such as `https://glue.us-east-1.amazonaws.com/iceberg`. |
| Amazon S3 Tables, Microsoft OneLake | AWS, Microsoft | The [Snowflake REST integration page](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-rest) lists them as commonly used REST catalogs. |
| Cloudflare Basin Catalog | Cloudflare | The [Cloudflare docs](https://developers.cloudflare.com/basin-catalog/) say it exposes a standard Iceberg REST catalog interface. The R2 Data Catalog address redirects to this page. |

### Catalog integration {#catalog-integration}

A catalog integration is a Snowflake object that holds the settings for an external Iceberg catalog: the catalog type, its address and name, and the sign-in details. For example, an integration for Snowflake Open Catalog has three key settings:

- `CATALOG_SOURCE = POLARIS` selects the catalog type.
- `REST_CONFIG = (CATALOG_URI = '<account URL>' CATALOG_NAME = '<catalog name>')` gives the catalog address and name.
- `REST_AUTHENTICATION = (TYPE = OAUTH OAUTH_CLIENT_ID = '<id>' OAUTH_CLIENT_SECRET = '<secret>' ...)` gives the sign-in details.

Three more settings decide how an Open Catalog integration behaves:

- `ACCESS_DELEGATION_MODE` chooses [credential vending](#credential-vending) or an external volume for storage access.
- `OAUTH_ALLOWED_SCOPES` names the principal role that the token carries, such as `PRINCIPAL_ROLE:readers` or `PRINCIPAL_ROLE:ALL`.
- `REFRESH_INTERVAL_SECONDS` sets how often Snowflake polls the catalog, 30 seconds by default.

The client ID and secret belong to the principal that Snowflake signs in as. For other Iceberg REST catalogs, `CATALOG_SOURCE = ICEBERG_REST` selects the type, and it also works for Open Catalog. The [Open Catalog integration reference](https://docs.snowflake.com/en/sql-reference/sql/create-catalog-integration-open-catalog) and the [Iceberg REST integration reference](https://docs.snowflake.com/en/sql-reference/sql/create-catalog-integration-rest) list all settings.

### Externally managed Iceberg table {#externally-managed-table}

An externally managed Iceberg table uses a catalog that another system owns, such as AWS Glue or an Iceberg REST catalog. You create a [catalog integration](#catalog-integration) for that catalog. Storage access comes from an [external volume](#external-volume), or from credentials that the catalog vends.

For a REST catalog, the table names the integration and its table in the catalog: `CREATE ICEBERG TABLE my_table EXTERNAL_VOLUME = '<volume>' CATALOG = '<integration>' CATALOG_TABLE_NAME = '<table>'`. The [CREATE ICEBERG TABLE reference](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-rest) lists the settings.

The catalog's owner is probably responsible for maintenance, such as compaction. The [Iceberg tables overview](https://docs.snowflake.com/en/user-guide/tables-iceberg) describes the two table types.

### Snowflake Open Catalog {#snowflake-open-catalog}

[Apache Polaris](https://polaris.apache.org/) is the software, and it implements the Iceberg REST API ([Apache Software Foundation press release](https://news.apache.org/foundation/entry/the-apache-software-foundation-graduates-two-open-source-projects-from-incubator)). Snowflake Open Catalog is Snowflake's hosted service that runs it. Its job is to give many engines read and write access to the same Iceberg tables. Snowflake, Spark, Flink, Trino, and a Kafka sink connector can all use it.

It is a Snowflake account of its own type, with its own web interface, users, and principals. An organization administrator creates it in Snowsight, under Admin > Accounts, or with CREATE ACCOUNT ... POLARIS = TRUE. Its REST endpoint has the form https://<org>-<account>.snowflakecomputing.com/polaris/api/catalog.

As of November 2026, only organizations that already have an Open Catalog account can create more. The [Open Catalog overview](https://docs.snowflake.com/en/user-guide/opencatalog/overview) describes the service.

Four names appear in the sources:

| Name | What it is | Source |
| --- | --- | --- |
| Apache Polaris | An open-source catalog for Apache Iceberg. It became an Apache Software Foundation Top-Level Project in 2026. | [Polaris blog, 19 February](https://polaris.apache.org/blog/2026/02/19/apache-polaris-graduates-to-top-level-project/), [Apache Software Foundation press release, 5 March](https://news.apache.org/foundation/entry/the-apache-software-foundation-graduates-two-open-source-projects-from-incubator) |
| Polaris Catalog | The 2024 name of Snowflake's hosted service. Snowflake announced it in June 2024 for the broader data ecosystem, which includes Snowflake customers. | [announcement, 3 June 2024](https://www.snowflake.com/en/blog/introducing-polaris-catalog/), [release note, 18 October 2024](https://docs.snowflake.com/en/release-notes/2024/other/2024-10-18-snowflake-open-catalog-ga) |
| Snowflake Open Catalog | The current name of that service. It is a managed service for Apache Polaris, hosted in Snowflake-managed infrastructure. | [overview](https://docs.snowflake.com/en/user-guide/opencatalog/overview) |
| Snowflake Horizon Catalog | The catalog that Snowflake runs for Snowflake-managed tables. External engines reach it through Polaris APIs. | [developer guide](https://www.snowflake.com/en/developers/guides/interoperable/), [Horizon docs](https://docs.snowflake.com/en/user-guide/snowflake-horizon) |

### Internal and external catalogs {#internal-and-external-catalogs}

An Open Catalog account holds one or more catalogs, and each catalog holds namespaces. Iceberg clients choose a catalog by name, in a setting called warehouse. Here, warehouse means the catalog name.

A catalog is internal or external. Open Catalog manages an internal catalog, and engines, including Snowflake, read and write its tables through Open Catalog. An external catalog mirrors tables that another catalog manages, for example Snowflake through catalog sync. Its tables are read-only in Open Catalog.

### Principal and roles {#principal-and-roles}

A principal is Open Catalog's identity for a program, such as Snowflake, a Spark job, or a Kafka sink connector. Open Catalog creates and manages it. When you create a service connection for an engine, Open Catalog creates a service principal with a client ID and a secret. The engine signs in with them through OAuth (open authorization) client credentials, a standard sign-in method for programs, and gets a short-lived token. The token expires, often after about an hour, so a long-running client such as a sink connector must refresh it.

Permissions reach a principal through a chain. Privileges go to a catalog role, a catalog role goes to a principal role, and a principal role goes to principals. A catalog role belongs to one catalog, and a principal role spans the whole account. When a client asks for a token, it names the principal role it wants, such as PRINCIPAL_ROLE:readers, or PRINCIPAL_ROLE:ALL for every role it holds.

For example, a Kafka sink connector signs in as principal kafka-sink, which holds principal role sink-writer. In catalog events, catalog role events-writer holds CATALOG_MANAGE_CONTENT and is granted to sink-writer. Snowflake signs in as principal snowflake-reader, whose principal role holds catalog role events-reader with TABLE_READ_DATA. So the sink can create tables and commit to them, and Snowflake can only read them.

TABLE_READ_DATA reads a table through short-lived read-only storage credentials. TABLE_CREATE and TABLE_WRITE_DATA create and write tables, and CATALOG_MANAGE_CONTENT bundles them with the namespace and metadata privileges. The [Apache Polaris access control guide](https://polaris.apache.org/in-dev/unreleased/managing-security/access-control/) describes the model.

### Credential vending {#credential-vending}

Credential vending means that the catalog hands out short-lived storage credentials together with a table. The credentials are scoped to that table's location in storage. So a trusted reader needs no storage role of its own, and table access is managed in one place: the catalog's roles.

The [Iceberg REST catalog spec](https://github.com/apache/iceberg/blob/main/open-api/rest-catalog-open-api.yaml) defines how any client asks for it. On a table lookup, the client sends the header `X-Iceberg-Access-Delegation: vended-credentials`. The catalog decides whether to vend, and returns the credentials in the `storage-credentials` field of its answer.

In Open Catalog, vending is a setting on each catalog, and it is off by default for external catalogs. A principal whose roles hold TABLE_READ_DATA gets read-only credentials, and the write privileges give credentials that can also write. A writer can use vended credentials, or keep storage keys of its own.

In Snowflake, the catalog integration turns vending on with ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS, for both CATALOG_SOURCE = POLARIS and ICEBERG_REST. The default, EXTERNAL_VOLUME_CREDENTIALS, uses an external volume instead.

For example, Snowflake reads tables through a catalog-linked database whose integration vends credentials. With each table lookup, the catalog returns the table's metadata location and credentials for that table's prefix, such as s3://lake/sensors/readings/. Snowflake needs no external volume for these tables.

The setting serves every Iceberg table that uses the integration: the tables of a catalog-linked database, and tables of their own. A table of its own then leaves out EXTERNAL_VOLUME, for example `CREATE ICEBERG TABLE readings CATALOG = 'open_catalog_int' CATALOG_NAMESPACE = 'sensors' CATALOG_TABLE_NAME = 'readings' AUTO_REFRESH = TRUE`. The [vended credentials guide](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-vended-credentials) describes the setup.

### Namespace and schema {#namespace-and-schema}

> edge.related: internal-and-external-catalogs, catalog-linked-database

A namespace is a logical entity inside a catalog that can hold tables or views. A schema is a logical grouping of database objects inside one Snowflake database ([Snowflake DDL reference](https://docs.snowflake.com/en/sql-reference/ddl-database)).

They do the same job, but they are not the same object. Both group tables below a higher level. The Polaris docs say other systems call a namespace a schema or a database ([Polaris entities page](https://polaris.apache.org/in-dev/unreleased/entities/)). In a catalog-linked database, each namespace appears as one schema.

### Catalog-linked database {#catalog-linked-database}

A catalog-linked database is a Snowflake database that mirrors one or more namespaces of a remote Iceberg REST catalog, such as Snowflake Open Catalog. Each table appears as an Iceberg table.

Snowflake polls the catalog, every 30 seconds by default (SYNC_INTERVAL_SECONDS). It finds new namespaces and tables on its own, and it picks up each table's new commits. No table needs its own setup in Snowflake. When its principal has the privileges, Snowflake can also create tables in the catalog and run INSERT, UPDATE, DELETE, and MERGE on them.

For example, this statement links namespace sensors, from the catalog behind integration open_catalog_int:

```sql
CREATE DATABASE lake LINKED_CATALOG = (CATALOG = 'open_catalog_int', ALLOWED_NAMESPACES = ('sensors'));
```

Within about 30 seconds, the catalog's tables readings and alarms appear as lake."sensors"."readings" and lake."sensors"."alarms". The names keep the catalog's lowercase, so queries quote them.

The Snowflake guide sets up a linked database with a catalog integration that vends credentials. When the catalog does not support vending, the database also names an external volume, for example `CREATE DATABASE lake LINKED_CATALOG = (CATALOG = 'open_catalog_int') EXTERNAL_VOLUME = 'lake_volume'`. The [CREATE DATABASE (catalog-linked) reference](https://docs.snowflake.com/en/sql-reference/sql/create-database-catalog-linked) lists the settings.

Snowflake permissions are granted separately from the catalog's roles. Grant SELECT on future Iceberg tables in each schema, so that new tables are readable when they appear. A grant on a schema fails until the first sync has created that schema.

The linked database has [limits](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database#considerations-for-using-a-catalog-linked-database-for-iceberg-tables). It works only with a catalog integration for an Iceberg REST catalog, and it supports no replication, cloning, or UNDROP. Renaming a table in the catalog drops and recreates it in Snowflake. As of November 2026, one Snowflake reference page still labels the feature as preview. The [Snowflake guide](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database) describes the options.

### Auto-refresh {#auto-refresh}

Auto-refresh keeps an externally managed Iceberg table in Snowflake up to date with its catalog. Snowflake polls the catalog for the table's current metadata, and moves the table to each new snapshot. When auto-refresh is off, the table stays on the snapshot it had until someone runs ALTER ICEBERG TABLE <name> REFRESH.

A table of its own needs AUTO_REFRESH = TRUE, because the default is FALSE. Tables in a catalog-linked database pick up new commits on their own. The catalog integration's REFRESH_INTERVAL_SECONDS sets how often Snowflake polls, 30 seconds by default.

Two checks show what happens. SYSTEM$AUTO_REFRESH_STATUS('<table>') reports whether auto-refresh runs for a table. ICEBERG_TABLE_SNAPSHOT_REFRESH_HISTORY lists each refresh, with its time, REFRESHED_ON, and the snapshot's summary. The [auto-refresh guide](https://docs.snowflake.com/en/user-guide/tables-iceberg-auto-refresh) describes the setup.

### Query caches {#query-caches}

> edge.related: auto-refresh

Snowflake can answer a query without reading the table's files from S3. It has two ways to do this: a saved result and the warehouse cache.

A saved result is the stored answer to an earlier query. Snowflake keeps it for 24 hours. It returns the saved result when the same SQL text runs again, the table data has not changed, and the role has the privileges. Each reuse restarts the 24 hours, up to 31 days after the first run. The [saved results guide](https://docs.snowflake.com/en/user-guide/querying-persisted-results) lists every condition.

A [virtual warehouse](https://docs.snowflake.com/en/user-guide/warehouses) is the cluster of servers that runs a query. The catalog setting called warehouse, under Internal and external catalogs, means something else: a catalog name. The warehouse cache is table data that a running warehouse keeps on its local disk. A later query that needs the same data reads it from there, not from storage. Suspending the warehouse drops the cache, so a short auto-suspend time empties it between queries. The [warehouse cache guide](https://docs.snowflake.com/en/user-guide/performance-query-warehouse-cache) describes it.

Both caches work for an externally managed Iceberg table. The node A query test, under Snowflake account, shows both at work.

### Snowflake Horizon Catalog {#horizon-catalog}

### Catalog sync {#catalog-sync}

## Data flow {#data-flow}

> type: category

Another engine writes a table → Snowflake Open Catalog → catalog integration → Snowflake reads the table
Snowflake writes a Snowflake-managed table → Horizon Catalog or catalog sync → another engine reads the table

### Snowflake account {#snowflake-account}

Holds the Iceberg tables that users query, and the objects that connect them to storage and to catalogs.

#### Inputs {#account-inputs}

The Snowflake account takes three kinds of input:

- table metadata and tokens from catalogs, through catalog integrations;
- data files and metadata files from cloud storage, through external volumes or vended credentials;
- SQL from users and tools, such as dbt.

#### Outputs {#account-outputs}

The account returns query results from Iceberg tables that Snowflake roles can query. For Snowflake-managed tables, it also writes new data files and metadata files to storage.

#### Under the hood {#account-under-the-hood}

For a table that an external catalog owns, Snowflake keeps a chain of objects:

1. A catalog integration that signs in to the catalog.
2. Storage access: vended credentials, or an external volume.
3. A database: a catalog-linked database, or an ordinary database with one Iceberg table per catalog table.
4. Grants that let Snowflake roles use the database, its schemas, and its Iceberg tables.

A break in any link stops the read, but each link fails differently. A wrong client secret fails at sign-in. A missing trust entry fails when Snowflake reads files. A missing grant fails only for the roles that lack it.

#### A query, in order {#account-query-order}

> edge.related: auto-refresh, query-caches

A query on an externally managed Iceberg table runs in the order below. Before any query, a refresh has set the table's snapshot. Auto-refresh does this in the background.

![Diagram of a query on an externally managed Iceberg table: a background refresh sets the snapshot; the query is planned on that snapshot; Snowflake returns a saved result if one exists; otherwise the warehouse reads the remaining files from its cache and from S3, returns the rows, and saves the result](snowflake-iceberg-d2-diagram/query-order.d2.svg)

1. The SQL arrives. Snowflake checks that the role has SELECT on the table.
2. Snowflake plans the query on the snapshot from the last refresh. It drops the data files that cannot match.
3. Snowflake looks for a saved result of the same SQL. If it finds one, it returns that result and reads no S3.
4. If not, the warehouse reads the remaining data files. It takes what it holds from the warehouse cache and reads the rest from S3, through the external volume or the catalog's vended credentials.
5. Snowflake returns the rows and saves the result.

Amber marks the step that reads S3. Teal marks the answer that skips S3.

#### A query test {#account-query-test}

A test shows both caches at work on an externally managed Iceberg table. It ran one SQL text five times and read the statistics of each run. The SQL counts rows, sums one column, and counts distinct values of another, for one fixed hour of data. The scan kept 13 of the table's 18,390 data files in runs 1 to 4, and 13 of 18,391 in run 5.

| Run | Saved results | Data from the cache | Execution time | Elapsed time |
| --- | --- | --- | --- | --- |
| 1 | off | 0 | 425 ms | 2,267 ms |
| 2 | off | 1 | 36 ms | 1,907 ms |
| 3 | on | 1 | 31 ms | 1,855 ms |
| 4 | on | none: the saved result of run 3 | 1 ms | 129 ms |
| 5, six minutes later | on | 0 | 293 ms | 1,054 ms |

Run 1 read all its data from S3. Runs 2 and 3 read the same data from the warehouse cache. Run 4 returned the saved result of run 3 and scanned no bytes.

The cache column is `percentage_scanned_from_cache`. It is a fraction: 0 means all data came from S3, and 1 means all came from the cache. A query for two hours of data, run right after a query for the first hour, showed about 0.16.

Run 5 used the same SQL after one refresh had moved the table to a new snapshot. It did not reuse the saved result, and it took nothing from the cache. The test warehouse suspends after 60 idle seconds.

To repeat the test on your own table, use a query of this shape. Run the same SELECT twice with saved results off, then turn them on and run it twice more. Read the statistics right after each run:

```sql
ALTER SESSION SET USE_CACHED_RESULT = FALSE;
SELECT SUM("value") FROM lake."sensors"."readings"
  WHERE "produced_at" >= '2026-11-16 06:00:00' AND "produced_at" < '2026-11-16 07:00:00';
SELECT operator_type, operator_statistics
  FROM TABLE(GET_QUERY_OPERATOR_STATS(LAST_QUERY_ID()));
```

In the statistics, `io.percentage_scanned_from_cache`, `pruning.partitions_scanned`, and `pruning.partitions_total` describe the scan. For an Iceberg table, a partition is one data file: `partitions_total` matched the number of rows that ICEBERG_TABLE_FILES returned. A run that returned a saved result shows the operator QUERY RESULT REUSE instead of TableScan.

### Snowflake Open Catalog account {#open-catalog-account}

A separate catalog service, with its own web interface, that tracks Iceberg tables for many engines.

#### Inputs {#open-catalog-inputs}

Open Catalog takes four kinds of input:

- sign-in requests, with the client ID and secret of a principal;
- table lookups from readers, such as Snowflake;
- commit requests from writers, such as Snowflake or a Kafka sink connector;
- setup changes from people, in the web interface or the API: catalogs, service connections, roles, and grants.

#### Outputs {#open-catalog-outputs}

For each sign-in, Open Catalog returns a short-lived token. For each table lookup, it returns the location of the table's current metadata file. When credential vending is on, it also returns short-lived storage credentials. For each commit, it writes a new metadata file to storage and moves the table's pointer to that file.

#### Under the hood {#open-catalog-under-the-hood}

Each catalog has storage settings: a default base location, such as s3://bucket/prefix, optional extra locations, and an IAM role in your cloud account. Open Catalog has one IAM user for the whole account. You add that user to the role's trust policy, with an external ID, and Open Catalog assumes the role to reach table data, metadata files, and manifest files.

With that access, Open Catalog writes the new metadata file on each commit. It probably also uses the role to create the short-lived credentials that it vends. The [catalog setup guide](https://docs.snowflake.com/en/user-guide/opencatalog/create-catalog) lists the storage settings.

### Cloud storage {#cloud-storage}

Holds the data files and metadata files of every Iceberg table.

#### Inputs {#storage-inputs}

#### Outputs {#storage-outputs}

#### Under the hood {#storage-under-the-hood}

### Other engines {#other-engines}

Engines such as Spark, Flink, and Trino that read and write the same tables through a catalog.

#### Inputs {#engines-inputs}

#### Outputs {#engines-outputs}

#### Under the hood {#engines-under-the-hood}

## Choices {#choices}

> type: category

Which decisions shape an Iceberg setup in Snowflake?

### Snowflake or an external catalog {#choice-catalog-owner}

This choice decides which catalog owns the table, and so which engines write it.

With Snowflake as the catalog, the table is Snowflake-managed. Snowflake writes and maintains it, and other engines read and write it through Horizon Catalog.

With an external catalog, such as Snowflake Open Catalog, the table is externally managed. Engines such as a Kafka sink connector write it through that catalog. Snowflake reads it through a catalog integration, either as a table of its own or in a catalog-linked database.

Choose Snowflake as the catalog when Snowflake writes the data. Choose an external catalog when other engines write it, and Snowflake mostly reads it.

### Catalog-linked database or one table per source table {#choice-binding}

> edge.related: credential-vending

Snowflake can bind to an external catalog in two ways. This table shows each one as it is usually set up:

| | Catalog-linked database | One Iceberg table per source table |
| --- | --- | --- |
| Snowflake objects | A catalog integration and one database | A catalog integration, an external volume, and one CREATE ICEBERG TABLE per table |
| A new table appears | On the next poll after a writer creates it in the catalog | Only after someone registers it with CREATE ICEBERG TABLE |
| New commits appear | On the next poll | Only with AUTO_REFRESH on, or after a manual refresh |
| S3 access | Vended credentials | The external volume's IAM role |
| Infrastructure as code, such as Terraform | The integration and the database, declared once | One resource per table, and a change and an apply for each new table |

Storage access is a setting of the catalog integration, so either binding can use vended credentials or an external volume, as credential vending explains.

In an ordinary database, CREATE ICEBERG TABLE registers a table that the catalog already has, under a Snowflake name. The statement fails with "table not found in catalog" until a writer has created the table in the catalog.

For a table of its own, [auto-refresh](#auto-refresh) is off by default. The table then keeps an old snapshot while writers keep committing, and nothing reports an error.

Infrastructure as code (IaC) declares objects in files, and a tool such as Terraform creates them when someone runs an apply. With the per-table way, each table is one resource, such as [`snowflake_iceberg_table_from_rest`](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/iceberg_table_from_rest.md) in the Snowflake Terraform provider. Each new source table needs a change to the files and an apply, and that apply fails with "table not found in catalog" until a writer has created the table. With a catalog-linked database, the files declare the integration and the database once, and new tables in a linked namespace need no change.

As of provider version 2.23.0 (November 2026), the per-table resource is a preview feature. The [`snowflake_database`](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/database.md) resource has the settings of an ordinary database, so a linked database is created with SQL, for example through the [`snowflake_execute`](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/execute.md) resource.

The per-table way is older and more manual, and Snowflake still supports it. Choose a catalog-linked database when you want every table in a namespace, kept current with no table-by-table work. Choose the per-table way when you want only some of the tables, or your own names in an existing database. It also fits when you need a feature that the linked database lacks. With IaC, it also lets a team review each new table before Snowflake exposes it.

### Horizon Catalog or catalog sync {#choice-publishing}
