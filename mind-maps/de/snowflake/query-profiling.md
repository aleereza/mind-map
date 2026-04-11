## Snowflake Query Profiling {#sf-query-profiling}

> [!meta]
> type: category

High-level map for reading Snowflake query performance, capturing the metrics that matter, and linking slow queries back to a dbt project.
checkout this youtube video: [Snowflake Query Profiling Guide](https://www.youtube.com/watch?v=dQw4w9WgXcQ)

### Snowflake Docs {#snowflake-docs}

> [!meta]
> type: link
> target: https://docs.snowflake.com/

### Where To Look {#sf-where-to-look}

> [!meta]
> type: category

Start with the place that identifies the query, then move into the profile and history views that explain why it was slow or expensive.

#### Snowsight Query History {#sf-snowsight-query-history}

> [!meta]
> type: concept

Snowsight Query History is the fastest entry point for recent troubleshooting because it shows query text, status, runtime, warehouse, user, and a direct link into the profile.

#### Query Profile Graph {#sf-query-profile-graph}

> [!meta]
> type: concept

```edges
prereqs: sf-snowsight-query-history
related: sf-most-expensive-nodes, sf-operator-hotspots
```

The Query Profile graph shows the execution plan as operators and steps so you can see where time, scan volume, spill, and skew concentrate.

#### Query History Views {#sf-query-history-views}

> [!meta]
> type: concept

```edges
related: sf-query-identity, sf-query-tags, sf-capture-key-facts
```

Snowflake history table functions cover recent activity and the Account Usage `QUERY_HISTORY` view extends analysis to longer windows, so you can analyze query_id, warehouse, timing, scan volume, and credits in SQL instead of only in the UI.

#### Pattern-Level Views {#sf-pattern-level-views}

> [!meta]
> type: detail

```edges
prereqs: sf-query-history-views
related: sf-query-identity
```

Grouped history by query hash or parameterized query hash helps separate a one-off incident from a repeated pattern that deserves a dbt or warehouse change.

### Core Metrics {#sf-core-metrics}

> [!meta]
> type: category

These are the first facts to capture before debating fixes. They tell you whether the query is expensive because it reads too much data, waits too long, or does too much work after scanning.

#### Query Identity {#sf-query-identity}

> [!meta]
> type: concept

The minimum identity set is query_id, query text, user, role, warehouse, start time, and query hash so you can correlate the same statement across profile views, history tables, grouped history, and dbt runs.

#### GB Scanned {#sf-gb-scanned}

> [!meta]
> type: concept

```edges
related: sf-partition-pruning, sf-cache-usage, sf-warehouse-context
```

GB scanned is the headline measure of how much data Snowflake had to read, usually derived from bytes scanned and best interpreted together with pruning and cache signals.

#### Total Time {#sf-total-time}

> [!meta]
> type: concept

```edges
related: sf-queue-compile-time, sf-most-expensive-nodes, sf-cost-vs-runtime
```

Total elapsed time tells you what the user felt, but it only becomes actionable after you split it into queueing, compilation, execution, and post-processing time.

#### Queue and Compile Time {#sf-queue-compile-time}

> [!meta]
> type: detail

```edges
related: sf-warehouse-context
contrasts: sf-gb-scanned
```

High `queued_provisioning_time`, `queued_repair_time`, or `compilation_time` points to warehouse pressure, concurrency, or plan complexity rather than to raw table scan volume.

#### Rows and Result Size {#sf-rows-result-size}

> [!meta]
> type: concept

```edges
related: sf-data-skew-explosions, sf-operator-hotspots
```

Rows produced, rows returned, and result size help detect fanout, poor join selectivity, and expensive downstream sorts or aggregations.

### Efficiency Signals {#sf-efficiency-signals}

> [!meta]
> type: category

These metrics explain whether the query shape was efficient for the data that Snowflake had available.

#### Partition Pruning {#sf-partition-pruning}

> [!meta]
> type: principle

```edges
prereqs: sf-gb-scanned
related: sf-decide-the-fix
```

The ratio between `partitions_scanned` and `partitions_total` shows whether predicates narrowed the read set or forced a broad table scan.

#### Cache Usage {#sf-cache-usage}

> [!meta]
> type: principle

```edges
related: sf-gb-scanned, sf-cost-vs-runtime
```

Bytes served from cache can make repeat executions look cheap and fast, so compare cold-run behavior before claiming a model is well optimized.

#### Spill and Network {#sf-spill-network}

> [!meta]
> type: concept

```edges
related: sf-operator-hotspots, sf-warehouse-context
```

`bytes_spilled_to_local_storage`, `bytes_spilled_to_remote_storage`, and `bytes_sent_over_the_network` usually spike when sorts, joins, or aggregations exceed memory or shuffle too much intermediate data.

#### Warehouse Context {#sf-warehouse-context}

> [!meta]
> type: concept

```edges
related: sf-cost-vs-runtime, sf-queue-compile-time
```

Warehouse size, multi-cluster behavior, and concurrent workload determine whether the same SQL spends time computing or simply waiting for usable resources.

### Reading A Slow Query {#sf-reading-a-slow-query}

> [!meta]
> type: category

Read the profile from the bottleneck outward. Find the expensive node first, then inspect the SQL shape that fed it.

#### Most Expensive Nodes {#sf-most-expensive-nodes}

> [!meta]
> type: concept

```edges
prereqs: sf-query-profile-graph
related: sf-total-time
```

Start with the operators that dominate time or scan volume because they usually identify the narrowest place to investigate first.

#### Operator Hotspots {#sf-operator-hotspots}

> [!meta]
> type: concept

```edges
related: sf-spill-network, sf-rows-result-size
```

Table scans, joins, sorts, aggregates, and window functions are the usual hotspots, and each points to a different class of model or SQL change.

#### Data Skew and Explosions {#sf-data-skew-explosions}

> [!meta]
> type: concept

```edges
related: sf-rows-result-size, sf-spill-network
```

Skew shows up when a small part of the data drives most of the work, while row explosions usually come from many-to-many joins or incomplete join predicates.

#### Cost vs Runtime {#sf-cost-vs-runtime}

> [!meta]
> type: principle

```edges
prereqs: sf-gb-scanned, sf-total-time
related: sf-warehouse-context
```

Fast queries can still be expensive if they scan too much data, and slow queries can still be cheap if they mostly wait in queue, so track cost and time separately.

### Tie To dbt {#sf-tie-to-dbt}

> [!meta]
> type: category

The goal is to move from a Snowflake query back to the dbt model, macro, and lineage that produced it.

#### dbt Query Comments {#sf-dbt-query-comments}

> [!meta]
> type: concept

```edges
related: sf-query-identity, sf-map-back-to-dbt
```

dbt query comments let you stamp SQL with metadata such as `node_id`, target, and invocation context, and on Snowflake the default JSON comment is appended to the end of the query so it survives execution.

#### Manifest and Lineage {#sf-dbt-manifest-lineage}

> [!meta]
> type: concept

```edges
related: sf-dbt-compiled-sql, sf-map-back-to-dbt
```

`manifest.json` tells you which file owns the model, what it depends on, and which downstream models are affected if you change the SQL shape, using keys such as `nodes`, `parent_map`, and `child_map`.

#### Run Results and Artifacts {#sf-dbt-run-results}

> [!meta]
> type: detail

```edges
related: sf-dbt-manifest-lineage, sf-query-identity
```

`run_results.json` gives execution status, `unique_id`, `execution_time`, timing steps, `adapter_response`, `compiled_code`, and `relation_name` so you can compare Snowflake runtime signals with the model-level timing seen by dbt.

#### Compiled SQL and Model Files {#sf-dbt-compiled-sql}

> [!meta]
> type: concept

```edges
prereqs: sf-dbt-query-comments
related: sf-operator-hotspots, sf-decide-the-fix
```

Once you identify the model, compare the compiled SQL with the original model file to find filters, joins, incremental logic, or macros that drove the bad profile.

#### Query Tags {#sf-query-tags}

> [!meta]
> type: detail

```edges
related: sf-query-history-views, sf-map-back-to-dbt
```

If your execution flow sets Snowflake `QUERY_TAG`, it becomes a durable join key for grouping related dbt statements by environment, job, or model family.

### Practical Workflow {#sf-practical-workflow}

> [!meta]
> type: category

This turns profile reading into a repeatable debugging loop instead of an ad hoc hunt.

#### Find The Query {#sf-find-the-query}

> [!meta]
> type: concept

```edges
prereqs: sf-snowsight-query-history
related: sf-query-identity
```

Start with the slow or expensive statement in history and lock down the exact query_id, warehouse, user, and execution window before jumping to conclusions.

#### Capture Key Facts {#sf-capture-key-facts}

> [!meta]
> type: concept

```edges
prereqs: sf-gb-scanned, sf-total-time
related: sf-partition-pruning, sf-spill-network
```

Write down GB scanned, total elapsed time, queue time, rows produced, partitions scanned, spill, and the dominant operators so the investigation stays evidence-based.

#### Map Back To dbt {#sf-map-back-to-dbt}

> [!meta]
> type: concept

```edges
prereqs: sf-dbt-query-comments, sf-dbt-manifest-lineage
related: sf-dbt-compiled-sql, sf-query-tags
```

Use query comments, tags, compiled SQL, and dbt artifacts together to map the Snowflake statement back to the owning model and its upstream lineage.

#### Decide The Fix {#sf-decide-the-fix}

> [!meta]
> type: principle

```edges
prereqs: sf-map-back-to-dbt, sf-most-expensive-nodes
related: sf-partition-pruning, sf-warehouse-context
```

Choose the fix that matches the observed bottleneck: better filters and clustering for broad scans, better join shape for row explosions, or warehouse changes for queue and memory pressure.
