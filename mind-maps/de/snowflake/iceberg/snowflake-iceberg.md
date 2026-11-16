# Iceberg in Snowflake {#snowflake-iceberg}

> type: category

How does Snowflake work with Iceberg tables, and where does Snowflake Open Catalog fit? Snowflake can be the catalog for an Iceberg table, or it can use tables that another catalog owns. Snowflake Open Catalog is a separate catalog service that many engines can share.

## Main concepts {#main-concepts}

> type: category

### External volume {#external-volume}

An external volume is a Snowflake object that points to a location in your cloud storage, such as an S3 bucket. Snowflake uses it to reach the data files and the Iceberg metadata files of a table that lives in your storage. The [CREATE EXTERNAL VOLUME reference](https://docs.snowflake.com/en/sql-reference/sql/create-external-volume) lists its settings.

### Snowflake-managed Iceberg table {#snowflake-managed-table}

A Snowflake-managed Iceberg table uses Snowflake as its Iceberg catalog. You create it with `CREATE ICEBERG TABLE` and `CATALOG = 'SNOWFLAKE'`. The files need storage: Snowflake-managed storage, or an [external volume](#external-volume) that points to your bucket. The [CREATE ICEBERG TABLE reference](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-snowflake) lists the settings.

Snowflake generates the Iceberg metadata files itself, for version 2 or 3 of the Apache Iceberg specification, and writes them to the external volume. It does this on a periodic basis with the function `SYSTEM$GET_ICEBERG_TABLE_INFORMATION`, and you can run that function on demand. Each new metadata file holds all changes since the previous one. The runs that Snowflake starts appear under the user `SYSTEM` in the query history, and Snowflake deletes old metadata, manifest lists, and manifests after the retention period. The [Metadata and retention page](https://docs.snowflake.com/en/user-guide/tables-iceberg-metadata) describes this.

Snowflake handles all lifecycle maintenance for the table, such as compaction, and you can turn compaction off. The [Iceberg tables overview](https://docs.snowflake.com/en/user-guide/tables-iceberg) describes the two table types.

### Catalog integration {#catalog-integration}

A catalog integration is a Snowflake object that holds the settings for an external Iceberg catalog: the catalog type, its address and name, and the sign-in details. For example, an integration for Snowflake Open Catalog has three key settings:

- `CATALOG_SOURCE = POLARIS` selects the catalog type.
- `REST_CONFIG = (CATALOG_URI = '<account URL>' CATALOG_NAME = '<catalog name>')` gives the catalog address and name.
- `REST_AUTHENTICATION = (TYPE = OAUTH OAUTH_CLIENT_ID = '<id>' OAUTH_CLIENT_SECRET = '<secret>' ...)` gives the sign-in details.

For other Iceberg REST catalogs, `CATALOG_SOURCE = ICEBERG_REST` selects the type. The [Open Catalog integration reference](https://docs.snowflake.com/en/sql-reference/sql/create-catalog-integration-open-catalog) and the [Iceberg REST integration reference](https://docs.snowflake.com/en/sql-reference/sql/create-catalog-integration-rest) list all settings.

### Externally managed Iceberg table {#externally-managed-table}

An externally managed Iceberg table uses a catalog that another system owns, such as AWS Glue or an Iceberg REST catalog. You create a [catalog integration](#catalog-integration) for that catalog. Storage access comes from an [external volume](#external-volume), or from credentials that the catalog vends.

For a REST catalog, the table names the integration and its table in the catalog: `CREATE ICEBERG TABLE my_table EXTERNAL_VOLUME = '<volume>' CATALOG = '<integration>' CATALOG_TABLE_NAME = '<table>'`. The [CREATE ICEBERG TABLE reference](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-rest) lists the settings.

The catalog's owner is probably responsible for maintenance, such as compaction. The [Iceberg tables overview](https://docs.snowflake.com/en/user-guide/tables-iceberg) describes the two table types.

### Snowflake Open Catalog {#snowflake-open-catalog}

Snowflake Open Catalog is Snowflake's hosted version of Apache Polaris, an open-source Iceberg REST catalog. Its job is to give many engines read and write access to the same Iceberg tables. Snowflake, Spark, Flink, Trino, and a Kafka sink connector can all use it.

It is a Snowflake account of its own type, with its own web interface, users, and principals. An organization administrator creates it in Snowsight, under Admin > Accounts, or with CREATE ACCOUNT ... POLARIS = TRUE. Its REST endpoint has the form https://<org>-<account>.snowflakecomputing.com/polaris/api/catalog.

It is probably separate because its clients are other engines, which need only a catalog API and identities of their own. As of November 2026, only organizations that already have an Open Catalog account can create more. The [Open Catalog overview](https://docs.snowflake.com/en/user-guide/opencatalog/overview) describes the service.

### Internal and external catalogs {#internal-and-external-catalogs}

An Open Catalog account holds one or more catalogs, and each catalog holds namespaces, which group tables much as schemas do in Snowflake. Iceberg clients choose a catalog by name, in a setting called warehouse. Here, warehouse means the catalog name.

A catalog is internal or external. Open Catalog manages an internal catalog, and engines, including Snowflake, read and write its tables through Open Catalog. An external catalog mirrors tables that another catalog manages, for example Snowflake through catalog sync. Its tables are read-only in Open Catalog.

### Principal and roles {#principal-and-roles}

A principal is Open Catalog's identity for a program, such as Snowflake, a Spark job, or a Kafka sink connector. Open Catalog creates and manages it. When you create a service connection for an engine, Open Catalog creates a service principal with a client ID and a secret. The engine signs in with them through OAuth (open authorization) client credentials, a standard sign-in method for programs, and gets a short-lived token.

Permissions reach a principal through a chain. Privileges go to a catalog role, a catalog role goes to a principal role, and a principal role goes to principals. A catalog role belongs to one catalog, and a principal role spans the whole account. When a client asks for a token, it names the principal role it wants, such as PRINCIPAL_ROLE:readers, or PRINCIPAL_ROLE:ALL for every role it holds.

For example, a Kafka sink connector signs in as principal kafka-sink, which holds principal role sink-writer. In catalog events, catalog role events-writer holds CATALOG_MANAGE_CONTENT and is granted to sink-writer. Snowflake signs in as principal snowflake-reader, whose principal role holds catalog role events-reader with TABLE_READ_DATA. So the sink can create tables and commit to them, and Snowflake can only read them.

TABLE_READ_DATA reads a table through short-lived read-only storage credentials. TABLE_CREATE and TABLE_WRITE_DATA create and write tables, and CATALOG_MANAGE_CONTENT bundles them with the namespace and metadata privileges. The [Apache Polaris access control guide](https://polaris.apache.org/in-dev/unreleased/managing-security/access-control/) describes the model.

### Credential vending {#credential-vending}

### Catalog-linked database {#catalog-linked-database}

A catalog-linked database is a Snowflake database that mirrors one or more namespaces of a remote Iceberg REST catalog, such as Snowflake Open Catalog. Each namespace appears as a schema, and each table appears as an Iceberg table.

Snowflake polls the catalog, every 30 seconds by default (SYNC_INTERVAL_SECONDS). It finds new namespaces and tables on its own, and it picks up each table's new commits. No table needs its own setup in Snowflake. When its principal has the privileges, Snowflake can also create tables in the catalog and run INSERT, UPDATE, DELETE, and MERGE on them.

For example, this statement links namespace sensors, from the catalog behind integration open_catalog_int:

```sql
CREATE DATABASE lake LINKED_CATALOG = (CATALOG = 'open_catalog_int', ALLOWED_NAMESPACES = ('sensors'));
```

Within about 30 seconds, the catalog's tables readings and alarms appear as lake."sensors"."readings" and lake."sensors"."alarms". The names keep the catalog's lowercase, so queries quote them.

Snowflake permissions are granted separately from the catalog's roles. Grant SELECT on future Iceberg tables in each schema, so that new tables are readable when they appear. A grant on a schema fails until the first sync has created that schema.

The linked database has [limits](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database#considerations-for-using-a-catalog-linked-database-for-iceberg-tables). It works only with a catalog integration for an Iceberg REST catalog, and it supports no replication, cloning, or UNDROP. Renaming a table in the catalog drops and recreates it in Snowflake. As of November 2026, one Snowflake reference page still labels the feature as preview. The [Snowflake guide](https://docs.snowflake.com/en/user-guide/tables-iceberg-catalog-linked-database) describes the options.

### Auto-refresh {#auto-refresh}

### Snowflake Horizon Catalog {#horizon-catalog}

### Catalog sync {#catalog-sync}

## Data flow {#data-flow}

> type: category

Another engine writes a table → Snowflake Open Catalog → catalog integration → Snowflake reads the table
Snowflake writes a Snowflake-managed table → Horizon Catalog or catalog sync → another engine reads the table

### Snowflake account {#snowflake-account}

Holds the Iceberg tables that users query, and the objects that connect them to storage and to catalogs.

#### Inputs {#account-inputs}

#### Outputs {#account-outputs}

#### Under the hood {#account-under-the-hood}

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

Each catalog has storage settings: a default base location, such as s3://bucket/prefix, optional extra locations, and an IAM (AWS Identity and Access Management) role in your cloud account. Open Catalog has one IAM user for the whole account. You add that user to the role's trust policy, with an external ID, and Open Catalog assumes the role to reach table data, metadata files, and manifest files.

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

### Catalog-linked database or one table per source table {#choice-binding}

Snowflake can bind to an external catalog in two ways. This table shows each one as it is usually set up:

| | Catalog-linked database | One Iceberg table per source table |
| --- | --- | --- |
| Snowflake objects | A catalog integration and one database | A catalog integration, an external volume, and one CREATE ICEBERG TABLE per table |
| A new table appears | On the next poll after a writer creates it in the catalog | Only after someone registers it with CREATE ICEBERG TABLE |
| New commits appear | On the next poll | Only with AUTO_REFRESH on, or after a manual refresh |
| S3 access | Vended credentials | The external volume's IAM role |

In an ordinary database, CREATE ICEBERG TABLE registers a table that the catalog already has, under a Snowflake name. The statement fails with "table not found in catalog" until a writer has created the table in the catalog.

For a table of its own, AUTO_REFRESH is off by default, as the [CREATE ICEBERG TABLE reference](https://docs.snowflake.com/en/sql-reference/sql/create-iceberg-table-rest) shows. Then Snowflake never polls the catalog for that table. The table keeps an old snapshot while writers keep committing, and nothing reports an error.

The per-table way is older and more manual, and Snowflake still supports it. Choose a catalog-linked database when you want every table in a namespace, kept current with no table-by-table work. Choose the per-table way when you want only some of the tables, or your own names in an existing database. It also fits when you need a feature that the linked database lacks.

### Horizon Catalog or catalog sync {#choice-publishing}
