# dbt Deep Mental Model {#dbt-deep-mental-model}

> type: category

dbt is easiest to reason about as a compiler plus an execution planner over warehouse relations. The useful mental model is not "SQL files in folders"; it is metadata, dependencies, environment-specific relation resolution, and materialization behavior.

## Core vs Platform {#dbt-root-core-vs-platform}

> type: concept

dbt Core is the open-source command-line framework and runtime: you bring the local environment, adapter packages, `profiles.yml`, orchestrator, artifact storage, docs hosting, credentials, and operational monitoring. The dbt platform, formerly dbt Cloud, wraps dbt execution in a managed product surface: browser or CLI development, managed environments, connection profiles, credentials, jobs, CI, scheduling, docs/catalog, alerts, metadata APIs, and features such as deferral and Mesh workflows that depend on shared production metadata.

The project code can be mostly the same in both, but the execution context is different. In Core, a command resolves context from local files, flags, env vars, and your orchestrator; in the dbt platform, the context usually comes from configured environments, jobs, developer or deployment credentials, platform-managed artifacts, and repository integrations.

## Compiler And DAG {#dbt-compiler-and-dag}

> type: category

The DAG is built before execution. dbt parses project files, resolves resource metadata, extracts dependency-producing Jinja calls, then compiles selected nodes into warehouse SQL in dependency order.

### Command Lifecycle {#dbt-command-lifecycle}

> type: procedure

For `run`, `build`, `test`, `compile`, and similar commands, dbt first constructs the full project graph, then narrows that graph to the requested work, then compiles and executes selected resources.

#### Load Invocation Context {#dbt-lifecycle-load-invocation-context}

> type: step
> edge.next: dbt-lifecycle-parse-project

Resolve the invocation before reading resources: command name, project directory, profile or connection profile, target, execution environment, invocation overrides, runtime variables, adapter type, and runtime paths.

The result of this step is a coherent context for the rest of the command: where the project is, which profile or platform environment is active, which target-like connection context is active, which overrides apply, and where dbt should read or write support files.

##### Project Directory {#dbt-lifecycle-project-directory}

> type: concept

The project directory is the folder dbt treats as the project root. It matters because `dbt_project.yml`, configured resource paths, packages, and default artifact paths are resolved from that root.

In Core or local Fusion CLI, dbt uses `--project-dir` when supplied; otherwise it searches from the current working directory upward until it finds `dbt_project.yml`. In the dbt platform, the run starts from the configured project repository/code checkout, and dbt evaluates the project inside that managed workspace.

##### Profile {#dbt-lifecycle-profile}

> type: concept

A profile is the named connection context that tells dbt how to connect to a warehouse. It keeps deployment and runtime connection details out of transformation SQL, but Core and the dbt platform store this context differently.

In local Core or Fusion CLI, `profile:` in `dbt_project.yml` names a profile in `profiles.yml`, and `--profile` can override it. Example: `profile: analytics` points to an `analytics:` block with Snowflake, BigQuery, or Databricks outputs.

In the dbt platform, projects do not require `profiles.yml` unless you develop from a local machine. Platform profiles are UI/API-managed objects that define connections, credentials, and attributes for deployment environments; development environments use individual developer credentials instead.

###### Reminder: Environment Split {#dbt-lifecycle-profile-environment-split-reminder}

> type: comment

Development environment vs deployment environment.

##### Platform Environment {#dbt-lifecycle-platform-environment}

> type: concept

A dbt platform environment is the managed execution context for Studio IDE, dbt CLI development, or scheduled jobs. It defines the dbt version, warehouse connection information, and code version to execute.

Development environments use personal development credentials for each developer. Deployment environments use assigned connection profiles and deployment credentials, with Production, Staging, and General deployment types.

##### Target {#dbt-lifecycle-target}

> type: concept

A target is the active warehouse execution context exposed to dbt and Jinja. In Core it is a named output inside `profiles.yml`; in the dbt platform it is assembled from environment, connection, credential, extended-attribute, and job or developer settings.

In Core, the profile can define `dev`, `ci`, and `prod` outputs; `--target prod` chooses the production output, while no flag falls back to the profile's default `target`.

In the dbt platform, `target` still exists in Jinja, but it is populated from platform settings. Orchestrator jobs define `target.name` per job and get other target attributes from the deployment connection; Studio IDE gets target values from the developer's connection and credentials.

##### Platform Connection Profile {#dbt-lifecycle-platform-connection-profile}

> type: concept

A platform connection profile is not a checked-in `profiles.yml` file. It is a project-level dbt platform object assigned to deployment environments and backed by account-level warehouse connections, deployment credentials, and extended attributes.

Example: a Production environment can use a Snowflake connection profile with a service account, production role, production warehouse, target database, and target schema. A Staging environment can use a different profile or the same profile with different environment settings.

##### Invocation Overrides {#dbt-lifecycle-invocation-overrides}

> type: concept

Invocation overrides are the values that change how one command behaves without changing source files. They exist so local development, CI, and production can use the same project with different runtime context.

Examples: `--target ci`, `--vars '{"load_days": 3}'`, `DBT_PROFILES_DIR`, and platform job environment variables. For global flags, dbt resolves CLI option > environment variable > `flags:` in `dbt_project.yml` > dbt default.

##### Runtime Paths {#dbt-lifecycle-runtime-paths}

> type: concept

Runtime paths tell dbt where to read support files and where to write command outputs. They matter because state comparison, deferral, docs, and observability depend on reading the right artifacts.

In Core, profiles directory defaults to the local profiles location unless `--profiles-dir` or `DBT_PROFILES_DIR` overrides it; `target-path` defaults to `target/`; `packages-install-path` defaults to `dbt_packages/`; logs default to a log path under the active project.

In the dbt platform, jobs run in managed execution infrastructure. The platform prepares the run environment, clones the configured code version, connects to the data platform, runs built-in and configured commands, and surfaces logs, run steps, timings, and artifacts in the job run UI.

#### Parse Project Files {#dbt-lifecycle-parse-project}

> type: step
> edge.next: dbt-lifecycle-resolve-metadata

Find candidate files from static path config in `dbt_project.yml`. Defaults are relative to the project root: `model-paths: ["models"]`, `macro-paths: ["macros"]`, `seed-paths: ["seeds"]`, `snapshot-paths: ["snapshots"]`, and `test-paths: ["tests"]`; custom paths replace those defaults.

dbt reads SQL models, Python models where supported, resource `.yml` files, macros, seeds, snapshots, analyses, singular tests, generic test macros, exposures, metrics, semantic models, and sources from their configured resource paths. Sources and unit tests are discovered under `model-paths`; seeds are `.csv` files under `seed-paths`; installed packages are read from `packages-install-path`, defaulting to `dbt_packages`. Static parsing and partial parsing can reuse prior parse state from `target/partial_parse.msgpack`, but changes to project config, packages, dbt version, profile inputs, or relevant env vars can force broader re-parsing.

##### Resource Paths {#dbt-lifecycle-resource-paths}

> type: concept

Resource paths are the folders dbt scans for project resources. dbt does not scan the entire repository; it scans the paths configured in `dbt_project.yml`.

Examples: `model-paths: ["models"]` means models and source YAML are read from `models/`; `macro-paths: ["macros"]` means reusable Jinja macros are read from `macros/`.

##### Code Checkout Context {#dbt-lifecycle-code-checkout-context}

> type: concept

Code checkout context is where the source files come from before parsing. Core reads the local filesystem; the dbt platform runs against the code version configured for the active environment or job.

Example: a local Core command parses the branch currently checked out in your shell. A platform deployment job reclones the repository and uses the environment's default branch or custom branch before running job commands.

##### Model And Property Files {#dbt-lifecycle-model-and-property-files}

> type: concept

Model files contain executable model logic, while property files attach metadata to resources. This split lets SQL stay focused on transformation and YAML carry docs, tests, contracts, sources, and versions.

Example: `models/marts/orders.sql` defines a model; `models/marts/orders.yml` can define `description`, `columns`, `data_tests`, and `versions` for that model. A `sources:` block in a YAML file under `model-paths` declares external inputs.

##### Macro And Test Files {#dbt-lifecycle-macro-and-test-files}

> type: concept

Macros are reusable Jinja functions, and tests are SQL assertions or generic test macros. dbt parses them early because they can affect compilation, materializations, test nodes, and dependency extraction.

Examples: `macros/generate_schema_name.sql` can override schema naming; `tests/assert_positive_revenue.sql` is a singular test; a generic `test_not_empty_string` macro can be referenced from a column test.

##### Seed Snapshot And Analysis Files {#dbt-lifecycle-seed-snapshot-analysis-files}

> type: concept

Seeds, snapshots, and analyses are separate resource classes with different runtime behavior. dbt discovers each from its configured path and turns it into the appropriate resource type.

Examples: `seeds/country_codes.csv` can be loaded as a seed relation; `snapshots/customers_snapshot.sql` defines historical change capture; `analyses/revenue_debug.sql` can compile SQL without becoming a built model.

##### Parser Cache {#dbt-lifecycle-parser-cache}

> type: concept

The parser cache lets dbt reuse parse results when files and relevant context have not changed. It exists to make large projects faster during repeated local and CI invocations.

In Core, the cache is commonly stored as `target/partial_parse.msgpack` in the local project output path. In the dbt platform, parse caching is affected by the managed run environment, dbt version or engine, code version, environment settings, and available platform metadata.

Changes to dbt version, installed packages, project config, profile-sensitive inputs, or parse-relevant environment variables can invalidate enough context that dbt must re-parse more broadly.

#### Resolve Resource Metadata {#dbt-lifecycle-resolve-metadata}

> type: step
> edge.next: dbt-lifecycle-extract-dependencies

Turn discovered files into resources with package namespace, resource type, name, path, original file path, enabled state, config, properties, columns, tests, docs, groups, contracts, versions, and source declarations.

Identity comes from resource type + package name + resource name: model names usually come from file names, source identities come from `sources:` name plus table name, and package namespaces come from each package's `dbt_project.yml` `name`. Config precedence is most-specific-wins: inline `config()` in a resource > `config:` in a resource `.yml` file > path-scoped config in `dbt_project.yml`; within `dbt_project.yml`, deeper resource paths beat broader paths, and root-project config can override installed package config. Most configs clobber less-specific values; important merge exceptions are additive `tags`, merged `meta`, additive hooks, and key-level freshness overrides.

##### Resource Identity {#dbt-lifecycle-resource-identity}

> type: concept

Resource identity is the stable name dbt uses after parsing. It is what selectors, artifacts, lineage, tests, and docs join on.

Examples: a model can become `model.analytics.orders`; a source table can become `source.analytics.stripe.payments`; a generic test becomes its own test node tied to the resources it checks.

##### Package Namespace {#dbt-lifecycle-package-namespace}

> type: concept

Package namespace prevents resources from different projects or packages from colliding. It comes from each package's `dbt_project.yml` `name`.

Example: `ref('orders')` looks in the current namespace first, while `ref('finance', 'orders')` points at a model in the `finance` namespace when cross-package or cross-project references are valid.

##### Config Precedence {#dbt-lifecycle-config-precedence}

> type: concept

Config precedence decides which materialization, schema, tags, contracts, and adapter settings win when the same setting is defined in multiple places.

For a model, inline `{{ config(materialized='table') }}` > model YAML `config:` > path-scoped `dbt_project.yml` config. Within `dbt_project.yml`, a deeper path such as `models/marts/finance` beats a broader `models/marts` default.

##### Properties {#dbt-lifecycle-properties}

> type: concept

Properties describe resources and their contracts with humans and tools. They are not just comments; they drive docs, tests, contracts, source freshness, exposures, and semantic metadata.

Examples: a column property can add a `not_null` test and description; a source property can declare `database`, `schema`, `identifier`, and `freshness`; a model property can declare `access: public` or versions.

##### Merge Exceptions {#dbt-lifecycle-merge-exceptions}

> type: principle

Most config values are overwritten by the most-specific value, but some are intentionally merged so broad metadata does not disappear when a local override is added.

Examples: tags are additive, `meta` is merged by key, hooks are additive, and source freshness can be overridden at a more-specific source or table level.

#### Extract Dependencies {#dbt-lifecycle-extract-dependencies}

> type: step
> edge.next: dbt-lifecycle-build-manifest

Infer graph edges from dbt-aware declarations, not from arbitrary SQL text. dbt looks inside parse-time Jinja and YAML structures for `ref()`, `source()`, `config()`, generic and singular tests, exposures, metrics, semantic models, saved queries, and docs references.

`ref()` looks up a model, seed, or snapshot by name, optional package or project namespace, and optional version, then creates a dependency and later compiles to a relation. `source()` looks up a declared `sources:` entry by source name and table name, then creates a dependency and later compiles to the declared database, schema, and identifier. Raw `database.schema.table` strings, dynamic SQL that hides the target from parsing, and warehouse objects not declared as dbt resources do not create reliable dbt lineage.

##### `ref()` Edge {#dbt-lifecycle-ref-edge}

> type: concept

`ref()` tells dbt that one dbt-managed resource depends on another dbt-managed resource. It creates a graph edge and later compiles to the physical relation name for the active target.

Example: `select * from {{ ref('stg_orders') }}` means this model cannot run until `stg_orders` is available. `ref('orders', version=1)` pins a versioned model contract.

##### `source()` Edge {#dbt-lifecycle-source-edge}

> type: concept

`source()` tells dbt that a model depends on an externally loaded or externally owned relation declared in YAML. It creates lineage and enables source tests, docs, and freshness.

Example: `{{ source('stripe', 'payments') }}` looks up `sources: - name: stripe` and `tables: - name: payments`, then compiles to the configured database, schema, and identifier.

##### YAML-Derived Edges {#dbt-lifecycle-yaml-derived-edges}

> type: concept

Some dependencies come from YAML resources rather than model SQL. This is why dependency extraction is broader than scanning for `ref()` strings.

Examples: a relationship test depends on both the child model and referenced parent; an exposure can depend on models; semantic models and metrics can depend on model resources.

##### Runtime Relation Strings {#dbt-lifecycle-runtime-relation-strings}

> type: warning

Raw warehouse relation names are invisible to dbt lineage because dbt cannot know whether a string is an intended dependency, a comment, or dynamic SQL.

Example: `select * from raw.stripe.payments` may run in the warehouse, but dbt will not add source freshness, docs, selection, or lineage behavior unless that relation is declared and used through `source()`.

##### Dynamic Dependency Risk {#dbt-lifecycle-dynamic-dependency-risk}

> type: warning

Dynamic Jinja can make dependencies difficult to parse and review. It is useful for advanced macros, but risky when it hides the graph that build order and ownership depend on.

Example: choosing a ref name from an environment variable can surprise state selection and reviewers; prefer explicit `ref()` or documented macro patterns when dependency shape matters.

#### Build Manifest And DAG {#dbt-lifecycle-build-manifest}

> type: step
> edge.next: dbt-lifecycle-apply-selection

Assemble the parsed project into the manifest graph before command selection is applied. The manifest includes enabled resources plus metadata collections such as `nodes`, `sources`, `macros`, `docs`, `metrics`, `exposures`, `groups`, selectors, disabled resources, `parent_map`, `child_map`, and `group_map`.

The manifest represents the whole parsed project, not just the nodes that this command will execute. dbt can use the graph in memory during the invocation and can also persist it for docs, state comparison, deferral, Mesh, observability, and other artifact consumers.

##### Manifest {#dbt-lifecycle-manifest}

> type: concept

The manifest is dbt's parsed, machine-readable view of the project. It is the operational schema for lineage, docs, selection, CI, deferral, observability, and Mesh metadata.

Example: `target/manifest.json` records a model's unique id, original file path, compiled path, config, columns, depends-on list, and relation metadata.

##### Manifest Collections {#dbt-lifecycle-manifest-collections}

> type: concept

Manifest collections group parsed resources by type. They let external tools inspect the project without re-reading every SQL and YAML file.

Examples: `nodes` contains models, tests, seeds, snapshots, and analyses; `sources` contains declared source tables; `macros` contains parsed macro definitions; `exposures` and `metrics` contain downstream semantic metadata.

##### Parent And Child Maps {#dbt-lifecycle-parent-child-maps}

> type: concept

Parent and child maps are fast graph indexes derived from dependency extraction. They let dbt and tools answer upstream and downstream questions directly.

Example: `parent_map["model.analytics.fct_orders"]` lists its upstream refs and sources; `child_map["model.analytics.stg_orders"]` lists downstream resources affected by changes.

##### Disabled Resources {#dbt-lifecycle-disabled-resources}

> type: concept

Disabled resources are parsed enough to be known, but not active in the executable graph. Keeping them in metadata helps explain why a resource exists but is not selected or built.

Example: a model with `enabled: false` may appear in disabled metadata, while downstream refs to it should fail because it is not an enabled build target.

##### Artifact Location {#dbt-lifecycle-artifact-location}

> type: concept

Artifact location is where the parsed graph and later execution results are written. It matters because CI, docs, deferral, and state comparison must read the artifact from the intended run.

In Core, the default persisted manifest location is `target/manifest.json`; using `--target-path target/ci` keeps CI artifacts separate from local development artifacts. In the dbt platform, persisted graph metadata is associated with the managed run, environment, job history, and platform metadata features.

#### Apply Selection Policy {#dbt-lifecycle-apply-selection}

> type: step
> edge.next: dbt-lifecycle-compile-selected

Start from the full manifest, then narrow it to the command's active work set. dbt reads CLI `--select` and `--exclude`, selector definitions, graph operators such as `+`, set operators, selector methods such as `tag:`, `path:`, `source:`, `config:`, `state:`, and command-specific resource-type rules.

Selection order is match methods first, then graph expansion, then set operations and exclusions, then final filtering by command type. For example, `dbt run` keeps runnable models after selection, while `dbt test` keeps tests; tests can also be selected indirectly through selected parents according to indirect-selection policy. State selectors read a comparison manifest from `--state`; deferral reads a prior manifest from `--state` or `--defer-state` and resolves unbuilt upstream refs to relations described there.

##### Select And Exclude {#dbt-lifecycle-select-and-exclude}

> type: concept

`--select` chooses candidate resources and `--exclude` removes resources from that candidate set. This lets one parsed project support local, CI, backfill, and production command scopes.

Example: `dbt build --select tag:nightly --exclude config.materialized:view` starts with nightly-tagged resources and removes view models from that invocation.

##### Selector Methods {#dbt-lifecycle-selector-methods}

> type: concept

Selector methods are lenses over the manifest. They match resources by metadata rather than by only file path or name.

Examples: `tag:finance`, `path:models/marts`, `source:stripe`, `config.materialized:incremental`, and `state:modified` each select from a different manifest attribute.

##### Graph Operators {#dbt-lifecycle-graph-operators}

> type: concept

Graph operators expand a selection through upstream or downstream dependencies. They are useful because real impact usually follows lineage, not folder boundaries.

Example: `+fct_orders` includes ancestors of `fct_orders`; `stg_orders+` includes descendants; `+stg_orders+` includes both sides around that node.

##### Indirect Test Selection {#dbt-lifecycle-indirect-test-selection}

> type: concept

Indirect test selection controls which tests are included because their parent resources were selected. It prevents test behavior from being accidental in CI or targeted commands.

Example: selecting `fct_orders` can bring in tests attached to `fct_orders`; a relationship test with multiple parents may behave differently depending on whether all or only some parents are selected.

##### State And Deferral Inputs {#dbt-lifecycle-state-and-deferral-inputs}

> type: concept

State and deferral read a previous manifest so dbt can compare code changes or resolve unbuilt upstream relations. They exist to make slim CI and local partial builds practical.

In Core, you provide artifact paths explicitly with flags such as `--state path/to/prod_artifacts` and optionally `--defer-state`. Example: `--select state:modified+ --defer` selects changed resources and can resolve unbuilt parents from the comparison manifest.

In the dbt platform, Studio IDE and dbt CLI development can natively defer to production metadata, and CI or merge jobs can compare against production environment artifacts without you manually managing a local artifact directory.

#### Compile Selected Nodes {#dbt-lifecycle-compile-selected}

> type: step
> edge.next: dbt-lifecycle-execute-dag

Render selected resources using the active Jinja context: `target`, `var()`, `env_var()`, `this`, `model`, `graph`, adapter macros, package macros, dispatch rules, and command flags. Macro lookup uses namespace and dispatch search order, so root-project macros and configured dispatch can override package or adapter behavior.

`ref()` resolves to the selected or deferred relation for the referenced model, seed, or snapshot; `source()` resolves to the declared source database, schema, and identifier; `this` resolves to the current node's target relation after database, schema, alias, quoting, custom schema macros, and adapter rules are applied. The output is compiled SQL or executable materialization logic under the target path, while only executed nodes usually receive full compiled execution details in artifacts.

##### Jinja Context {#dbt-lifecycle-jinja-context}

> type: concept

The Jinja context is the set of variables and functions available while dbt renders a resource. It bridges project metadata, target information, macros, and SQL text.

Examples: `{{ target.name }}` can branch dev vs prod behavior, `{{ var('lookback_days') }}` reads invocation/project vars, and `{{ env_var('RAW_DATABASE') }}` reads environment configuration.

In Core, `target` values come from the active `profiles.yml` output. In the dbt platform, `target` values come from environment, connection, credentials, extended attributes, and job or developer settings.

##### Relation Resolution {#dbt-lifecycle-relation-resolution}

> type: concept

Relation resolution turns logical dbt references into physical warehouse object names. This is where environment-specific database, schema, alias, identifier, and quoting rules become concrete.

Example: `{{ ref('stg_orders') }}` might compile to `analytics_dev.alice.stg_orders` locally and `analytics_prod.marts.stg_orders` in production.

##### `this` Relation {#dbt-lifecycle-this-relation}

> type: concept

`this` is the relation dbt is currently building. It lets model SQL and materializations refer to the target object for the current node.

Example: an incremental model can compare incoming rows to `{{ this }}` so it only processes rows newer than the existing target relation.

##### Macro Dispatch {#dbt-lifecycle-macro-dispatch}

> type: concept

Macro dispatch lets dbt choose adapter-specific or project-overridden macro implementations. It is how the same materialization concept can compile differently across warehouses.

Example: `dateadd` or incremental merge logic can dispatch to Snowflake-specific, BigQuery-specific, or root-project override macros while the model calls a stable macro name.

##### Compiled SQL {#dbt-lifecycle-compiled-sql}

> type: concept

Compiled SQL is the SQL after Jinja, refs, sources, macros, config, schema naming, and quoting have been resolved. It is the best artifact for debugging what the warehouse actually saw.

Example: a model file containing `{{ ref('stg_orders') }}` compiles to a concrete relation name plus expanded CTEs, adapter syntax, and materialization-specific SQL.

#### Execute In DAG Order {#dbt-lifecycle-execute-dag}

> type: step
> edge.next: dbt-lifecycle-write-artifacts

Execute the selected subgraph in topological order: a node becomes eligible only after its selected upstream dependencies, required hooks, and materialization prerequisites are satisfied. Independent branches can run concurrently up to `threads`, adapter limits, warehouse limits, and engine scheduling behavior.

The active connection context and materialization macros decide the concrete warehouse operations: create view, create table, merge incremental data, build snapshot tables, load seed CSVs, run tests, or execute hooks. Runtime failures, `--fail-fast`, retries, transactions, and adapter responses affect what later nodes can run and what status is recorded.

##### Topological Order {#dbt-lifecycle-topological-order}

> type: concept

Topological order means every selected node runs only after its selected upstream dependencies are ready. It is the execution form of the DAG.

Example: `stg_orders` runs before `fct_orders`; two unrelated staging models can run at the same time if threads and warehouse capacity allow it.

##### Threads And Parallelism {#dbt-lifecycle-threads-parallelism}

> type: concept

Threads control how many dbt tasks can be in flight, but they do not override dependency order or warehouse limits. More threads help only when the DAG has enough independent work and the warehouse can handle it.

Example: `threads: 8` can run eight independent models concurrently, but a narrow chain of dependent models still runs mostly one at a time.

##### Materialization Macros {#dbt-lifecycle-materialization-macros}

> type: concept

Materialization macros convert a model resource into concrete warehouse operations. They are why `materialized='table'`, `view`, `incremental`, and `snapshot` behave differently.

Example: a table materialization may create or replace a relation; an incremental materialization may create a temp relation and merge it into `{{ this }}` using a `unique_key`.

##### Hooks And Operations {#dbt-lifecycle-hooks-operations}

> type: concept

Hooks and operations run SQL or macros around normal resource execution. They are useful for grants, audit logging, session settings, and adapter-specific setup.

Examples: `pre-hook` can set a warehouse role or delete a partition; `post-hook` can apply grants; `on-run-end` can log run metadata after all selected resources finish.

##### Platform Job Run {#dbt-lifecycle-platform-job-run}

> type: concept

A dbt platform job is more than a single local command. The platform prepares managed infrastructure, reclones the repository, connects to the data platform, automatically includes `dbt deps`, then chains the configured commands as run steps.

Example: a deployment job can run source freshness first, then `dbt build --select tag:hourly`, then docs generation. If a chained run step fails, later commands normally do not run and the job ends with an error status.

##### Runtime Status {#dbt-lifecycle-runtime-status}

> type: concept

Runtime status is dbt's record of what happened to each selected resource. It drives downstream behavior, alerts, and run result interpretation.

Examples: a node can be `success`, `error`, `fail`, `warn`, `skipped`, or adapter-specific response metadata; `--fail-fast` can stop later eligible work after the first serious failure.

#### Write Artifacts {#dbt-lifecycle-write-artifacts}

> type: step

Persist machine-readable outputs for the invocation. Common outputs are `manifest.json`, compiled SQL, `run_results.json`, `sources.json` for freshness, logs, timing data, and adapter responses.

These artifacts turn the command into durable evidence: what dbt understood, what it compiled, what it ran, what failed, how long it took, and which warehouse relations were involved.

##### Target Path {#dbt-lifecycle-target-path}

> type: concept

Target path is the output directory for dbt artifacts and compiled files. It matters because many later commands read artifacts from this directory.

In Core, default `target/` contains `manifest.json` and `run_results.json`; CI might use `--target-path target/ci` so local artifacts do not overwrite CI state. In the dbt platform, users usually reason about artifacts through run records and environment metadata rather than by manually managing a local target directory.

##### `manifest.json` Artifact {#dbt-lifecycle-manifest-json-artifact}

> type: concept

`manifest.json` is the persisted parsed project graph. It describes what dbt understood before execution results are considered.

Example: docs generation, lineage tools, state comparison, and Mesh metadata use the manifest to inspect resource configs, dependencies, owners, contracts, and compiled paths.

##### `run_results.json` Artifact {#dbt-lifecycle-run-results-json-artifact}

> type: concept

`run_results.json` is the persisted execution trace for the selected resources in one invocation. It answers what ran, how long it took, and what status or adapter response came back.

Example: observability tooling can join `unique_id`, timing, status, `adapter_response`, and `relation_name` to warehouse query history or scheduler metadata.

##### Freshness And Logs {#dbt-lifecycle-freshness-and-logs}

> type: concept

Freshness and logs capture non-model evidence about the invocation. They help separate upstream ingestion delay, dbt compilation behavior, and warehouse execution behavior.

Examples: `sources.json` records source freshness results; logs record command events, warnings, errors, timing, and useful context for debugging failed invocations.

##### Artifact Consumers {#dbt-lifecycle-artifact-consumers}

> type: concept

Artifact consumers are workflows that read dbt outputs after the command finishes. Their correctness depends on reading artifacts from the right environment and invocation.

Examples: docs reads manifest and catalog-style metadata; slim CI reads a previous manifest for `state:modified`; deferral reads prior relation locations; cost attribution joins run results to warehouse query history.

### Parse-Time Contract {#dbt-parse-time-contract}

> type: concept

Parse time is when dbt discovers resources, configs, properties, macros, and dependencies. Anything that affects graph shape or config should be stable enough to evaluate during parsing.

#### Static Analysis {#dbt-static-analysis}

> type: concept

The static parser can extract simple `ref()`, `source()`, and `config()` calls without fully rendering every model. Fusion expands this idea with native SQL understanding across dialects.

#### Partial Parsing {#dbt-partial-parsing}

> type: concept

Partial parsing reuses prior parse state so unchanged files are not re-parsed on every invocation. It speeds development, but graph-affecting env vars, project config, packages, dbt version, or key macros can force a full re-parse.

#### Volatile Parse Inputs {#dbt-volatile-parse-inputs}

> type: warning

Avoid using values such as `run_started_at`, `invocation_id`, or runtime-only flags to set dependencies, configs, or resource properties. They make parse-time state harder to cache and reason about.

### Node Identity {#dbt-node-identity}

> type: concept

Every resource has a stable identity such as `model.project_name.model_name` or `source.project_name.source_name.table_name`. That identity is the join key across selection, artifacts, lineage, tests, docs, and run results.

#### Package Namespace {#dbt-package-namespace}

> type: concept

The package or project name is part of resource identity. This matters when packages, project dependencies, or duplicated model names introduce ambiguity.

#### File Path Is Not Identity {#dbt-file-path-not-identity}

> type: principle

A folder layout is an authoring convenience; the manifest resource identity is what dbt uses operationally. Renaming paths and renaming resources have different blast radii.

### Config Resolution {#dbt-config-resolution}

> type: concept
> edge.related: dbt-manifest-operational-schema

dbt config is layered across project defaults, resource properties, inline `config()`, package behavior, vars, env vars, and adapter-specific settings. The compiled manifest is the place to inspect the resolved result.

#### `dbt_project.yml` Defaults {#dbt-project-yml-defaults}

> type: concept

`dbt_project.yml` sets project identity, resource paths, default configs, model subdirectory behavior, variables, seeds, snapshots, hooks, and package-level overrides.

#### Properties Files {#dbt-properties-files}

> type: concept

YAML properties files define resource metadata such as descriptions, tests, sources, exposures, groups, contracts, model versions, and source freshness.

#### Precedence Discipline {#dbt-config-precedence-discipline}

> type: principle

Put broad defaults high and exceptions close to the resource they affect. Scattered config is one of the fastest ways to make a dbt project hard to reason about.

### Dependency Producers {#dbt-dependency-producers}

> type: concept

`ref()`, `source()`, tests, exposures, metrics, semantic models, and some macros can create lineage. Raw SQL relation names do not create dbt lineage unless wrapped in a declared resource.

#### Jinja Is Metadata Plus SQL {#dbt-jinja-metadata-plus-sql}

> type: principle

Jinja is not just templating text. In dbt, specific Jinja calls mutate the project graph, configure nodes, or return adapter-aware `Relation` objects.

#### Hidden Dependency Risk {#dbt-hidden-dependency-risk}

> type: warning

Dynamic refs, adapter dispatch, conditional config, and raw relation strings can hide dependencies from reviewers. Prefer explicit declarations when the dependency affects build order or ownership.

### Selection Semantics {#dbt-selection-semantics}

> type: concept

Selectors operate on the graph after parsing. Graph operators such as ancestors and descendants are powerful because they select by dependency structure, not by folders alone.

#### Build Order {#dbt-build-order}

> type: concept

Within the selected graph, dbt schedules resources so dependencies are available before downstream nodes. Parallelism is constrained by DAG shape, adapter behavior, and thread or engine rules.

#### Indirect Test Selection {#dbt-indirect-test-selection}

> type: concept

Tests are graph resources too. A selected model can bring in tests directly or indirectly depending on selection mode, which is why CI commands need explicit test-selection policy.

## Data Boundaries {#dbt-data-boundaries}

> type: category

Most real-world dbt complexity comes from knowing which data boundary a relation belongs to: owned by this project, declared as an external warehouse input, imported as code, or exposed as an API from another project.

### `source()` External Boundary {#dbt-source-external-boundary}

> type: concept
> edge.contrasts: dbt-ref-owned-assets

Use `source()` for relations loaded or owned outside the current dbt project. It gives the warehouse object a semantic name, lineage edge, tests, docs, and freshness metadata without pretending dbt built it.

#### Source Naming Layer {#dbt-source-naming-layer}

> type: concept

`name`, `database`, `schema`, and `identifier` let you separate dbt-facing names from awkward physical names. This is the right place to normalize raw system naming without hiding the true warehouse location.

#### Freshness Contract {#dbt-source-freshness-contract}

> type: concept

Freshness checks make upstream load latency visible to dbt. They do not load data; they test whether upstream ingestion produced data recently enough for downstream assumptions.

#### Conditional Sources {#dbt-conditional-sources}

> type: concept

Environment-specific source databases can be resolved with `target` or `env_var()` while keeping one logical source in the DAG. The shape of the graph stays stable while physical relation names change by environment.

### `ref()` Owned Assets {#dbt-ref-owned-assets}

> type: concept

`ref()` is for dbt-managed models, seeds, and snapshots. It returns a relation, compiles to the correct physical object name, and creates a graph edge that controls build order and lineage.

#### Versioned Refs {#dbt-versioned-refs}

> type: concept

Versioned models let consumers pin a breaking-change boundary with `ref(..., version=1)`. Omitting the version means the consumer follows the latest version and accepts that upgrade behavior.

#### Access Levels {#dbt-access-levels}

> type: concept

Model access controls communicate ownership boundaries inside and across projects. Public models are the intended dependency surface; private implementation models should not become downstream contracts.

### Project Dependencies {#dbt-project-dependencies}

> type: concept
> edge.contrasts: dbt-packages

Project dependencies are declared in `dependencies.yml` and support dbt Mesh by resolving public models from another project through metadata rather than copying that project source code into your project.

#### Cross-Project Refs {#dbt-cross-project-refs}

> type: concept

A cross-project `ref()` treats an upstream public model as a dataset API. The producer owns quality and stability; the consumer depends on the published contract and lineage.

#### Producer Requirements {#dbt-producer-requirements}

> type: concept

The upstream project needs public models, a successful deployment run, and a production environment that publishes metadata. Staging metadata can also matter when downstream development resolves pre-prod references.

### Packages {#dbt-packages}

> type: concept

Packages are commonly declared in `packages.yml` and pull source code into the current project, like a software library. They are best for macros, reusable transformations, or starter projects, not for team-owned production data APIs.

### Direct Relation Names {#dbt-direct-relation-names}

> type: warning
> edge.contrasts: dbt-source-external-boundary, dbt-ref-owned-assets

Hard-coded `database.schema.table` references can be useful escape hatches, but they remove lineage, environment resolution, docs, freshness, and selection semantics. Use them only when that tradeoff is intentional.

### Other Resource Entrances {#dbt-other-resource-entrances}

> type: concept

Seeds, snapshots, analyses, exposures, metrics, semantic models, Python models, and adapter-specific external table patterns all add graph nodes or metadata. The key question is whether dbt owns the object, describes it, or merely queries it.

## Environments And Targets {#dbt-environments-and-targets}

> type: category

An environment is the execution context. A target is the active warehouse connection and naming context that compiles logical dbt resources into physical objects.

### Target Object {#dbt-target-object}

> type: concept

`target` exposes attributes such as profile name, target name, schema, adapter type, threads, and adapter-specific fields like Snowflake database, role, warehouse, and account.

#### `profiles.yml` Targets {#dbt-profiles-targets}

> type: concept

In dbt Core, targets come from `profiles.yml` and can be selected with `--target`. This is where local runs choose dev, qa, prod, or other warehouse credentials and object namespaces.

#### Platform Environments {#dbt-platform-environments}

> type: concept

In the dbt platform, deployment environments define dbt version or engine, warehouse connection settings, credentials, code version, and job behavior.

### Environment Isolation {#dbt-environment-isolation}

> type: principle

Keep logical resource names stable while isolating physical data, credentials, schemas, roles, branches, and permissions. The DAG should not fork just because dev and prod live in different places.

#### Developer Schemas {#dbt-developer-schemas}

> type: concept

Development usually isolates objects by schema, often with user-specific schema naming. This lets developers build selected models without overwriting shared production objects.

#### Production Source Of Truth {#dbt-production-source-of-truth}

> type: concept

One deployment environment should represent the authoritative production state. Features such as catalog, cross-project refs, and downstream metadata depend on knowing which run is production.

#### Staging Environment {#dbt-staging-environment}

> type: concept

Staging gives teams a pre-prod metadata and execution surface. It can support branch validation, restricted access to production data, staging catalog views, and Mesh resolution away from production.

### Deferral {#dbt-deferral}

> type: concept
> edge.related: dbt-manifest-operational-schema, dbt-state-selection

With `--defer --state`, unselected refs can resolve to relations from a saved manifest when they are not built in the current target. This enables slim CI and local development against production or staging parents.

#### Mixed-Environment Tests {#dbt-mixed-environment-tests}

> type: warning

Deferral can make tests query across environments. Relationship tests are the classic example: one parent may be built in dev while another resolves to production or staging.

### Conditional Configuration {#dbt-conditional-configuration}

> type: principle

Use conditional config for physical differences, not conceptual differences. If an environment condition changes model meaning, ownership, or grain, it should probably be a separate resource or contract decision.

## Performance Model {#dbt-performance-model}

> type: category

dbt performance is a combination of warehouse SQL shape, materialization choice, incremental correctness, graph design, parser speed, and orchestration policy.

### Materialization Choice {#dbt-materialization-choice}

> type: concept

Materialization decides how a model persists: view, table, incremental, ephemeral, materialized view, or a custom adapter-specific implementation. This is the biggest dbt-level performance lever.

#### Views In The Middle {#dbt-views-in-the-middle}

> type: principle

Views are cheap to build but can push expensive work to every downstream query. They are often fine near sources and dangerous when stacked under BI or many downstream models.

#### Tables At Boundaries {#dbt-tables-at-boundaries}

> type: principle

Tables make expensive or widely reused transformations concrete. They cost rebuild time and storage, but they protect downstream consumers from repeatedly paying the same transformation cost.

#### Ephemeral Models {#dbt-ephemeral-models}

> type: concept

Ephemeral models inline SQL into downstream models. They reduce object sprawl, but they can duplicate work and make compiled SQL larger when reused broadly.

### Incremental Design {#dbt-incremental-design}

> type: concept

Incremental models are performance features with correctness obligations. They need a clear grain, stable unique key when updates exist, and SQL that works for both full-refresh and incremental paths.

#### Filter Early {#dbt-incremental-filter-early}

> type: principle

Place `is_incremental()` filters as close to large upstream scans as the warehouse optimizer can benefit from. Late filters often save less compute than they appear to.

#### Unique Key And Strategy {#dbt-unique-key-and-strategy}

> type: concept

The `unique_key` defines update semantics for strategies such as merge. Nulls, changing grain, or weak surrogate keys turn incremental models into duplicate factories.

#### Incremental Predicates {#dbt-incremental-predicates}

> type: concept

`incremental_predicates` can limit how much existing target data a merge scans. It is an advanced cost-control lever and should match the data's late-arrival and correction behavior.

### Warehouse-Aware Config {#dbt-warehouse-aware-config}

> type: concept

Performance configs are adapter-specific: clustering, partitioning, sorting, distribution, warehouse selection, query tags, and incremental strategies all depend on the data platform.

#### Threads And Concurrency {#dbt-threads-and-concurrency}

> type: concept

More threads only help when the graph and warehouse can use them. Fusion may optimize parallelism differently by adapter, while Core treats configured threads more directly.

#### Pushdown Shape {#dbt-pushdown-shape}

> type: principle

Most dbt performance problems are warehouse query-shape problems: broad scans, fanout joins, skew, expensive windows, repeated CTEs, or models that hide predicates from the optimizer.

### Graph Cost Shape {#dbt-graph-cost-shape}

> type: concept

The same SQL can be cheap or expensive depending on graph placement. A heavy model upstream of many consumers deserves a different materialization and testing strategy than a one-off branch.

#### Test Cost {#dbt-test-cost}

> type: concept

Data tests are queries. Large relationship, uniqueness, and expression tests can dominate runtime unless scoped, indexed, filtered, sampled, or scheduled intentionally.

#### Rebuild Surface {#dbt-rebuild-surface}

> type: principle

Optimize the set of nodes you rebuild as much as individual SQL. State selection, source freshness, deferral, and orchestration policy determine how much of the DAG pays compute on each run.

## Artifacts And State {#dbt-artifacts-and-state}

> type: category

Artifacts are the durable machine-readable view of what dbt understood and did. They are the bridge between code, warehouse runs, docs, CI, Mesh metadata, lineage, and observability.

### Manifest Schema {#dbt-manifest-operational-schema}

> type: concept
> edge.related: dbt-parse-time-contract

`manifest.json` is an artifact, not source config. It contains the full parsed representation of enabled project resources: nodes, sources, macros, docs, selectors, parent map, child map, group map, and more.

#### Parent And Child Maps {#dbt-parent-child-maps}

> type: concept

`parent_map` and `child_map` are the operational DAG indexes. They let tools answer upstream, downstream, blast radius, and ownership questions without reparsing SQL.

#### Compiled SQL {#dbt-compiled-sql}

> type: concept

Compiled SQL shows what the warehouse actually receives after Jinja, refs, sources, macros, aliases, schemas, and adapter quoting are resolved.

### Run Results {#dbt-run-results}

> type: concept

`run_results.json` records what dbt executed: resource status, timing, failures, adapter response, compiled code, and relation names. It is the execution trace, not just a success flag.

#### Invocation Context {#dbt-invocation-context}

> type: concept

`invocation_id`, selected resources, logs, and timing records let you join dbt behavior to warehouse query history and scheduler metadata.

#### Query Comments {#dbt-query-comments}

> type: concept
> edge.related: dbt-run-results

Query comments and tags can stamp warehouse SQL with dbt node metadata. They make it possible to map warehouse cost or failures back to model identity and invocation context.

### Sources Artifact {#dbt-sources-artifact}

> type: concept

`sources.json` records source freshness results. It is useful for freshness-aware selection and for separating upstream load delays from dbt transformation failures.

### State Selection {#dbt-state-selection}

> type: concept

`state:modified` compares current project state with a prior manifest to select changed resources. It is powerful for CI and deployments, but it depends on saved artifacts being kept separate from the current target path.

#### State Caveats {#dbt-state-caveats}

> type: warning

Seeds over size thresholds, macro changes, vars, env vars, tests with multiple parents, and overwritten manifests can all make state comparison surprising. Treat state selection as a policy that needs tests.

### Docs And Catalog {#dbt-docs-and-catalog}

> type: concept

Docs, catalog, lineage, and governance tools are artifact consumers. Their quality depends on descriptions, tests, contracts, exposures, groups, and stable resource ownership in the manifest.

## Scale And Product Choices {#dbt-scale-and-product-choices}

> type: category

At scale, dbt design becomes less about writing models and more about choosing the right execution engine, collaboration boundary, ownership model, and metadata plane.

### Core vs Platform {#dbt-core-vs-platform}

> type: concept

dbt Core is the open-source local CLI framework. The dbt platform, formerly dbt Cloud, adds hosted development, scheduling, CI/CD, docs hosting, monitoring, alerts, metadata services, permissions, and integrated product surfaces.

#### Local Control {#dbt-local-control}

> type: concept

Core is simple to embed in external orchestrators and local workflows. You own dependency management, secrets, scheduling, artifact storage, docs hosting, and operational observability.

#### Managed Metadata Plane {#dbt-managed-metadata-plane}

> type: concept

The platform centralizes runs, environments, artifacts, docs, catalog, jobs, permissions, and metadata APIs. That shared metadata plane is what enables several Mesh and governance workflows.

### dbt Mesh {#dbt-mesh}

> type: concept

Mesh is an organizational pattern for scaling dbt across teams. It turns selected models into governed interfaces instead of forcing every consumer into the same monolithic project.

#### Public Model API {#dbt-public-model-api}

> type: principle

A public model should behave like an API: documented, tested, versioned when breaking changes happen, and owned by a team that can support downstream consumers.

#### Contracts And Versions {#dbt-contracts-and-versions}

> type: concept

Contracts and model versions make breaking changes explicit. They protect consumers from silent column, type, grain, or semantic changes.

#### Monorepo Or Mesh {#dbt-monorepo-or-mesh}

> type: concept

Monorepos optimize local refactoring and shared standards. Mesh optimizes team autonomy and stable interfaces. The right boundary is usually ownership and release cadence, not folder count.

### Fusion Engine {#dbt-fusion-engine}

> type: concept
> edge.related: dbt-static-analysis, dbt-state-aware-orchestration

Fusion is a Rust-based dbt engine with faster parsing, native SQL understanding, and a path toward more efficient large-DAG development and deployment.

#### Developer Feedback Loop {#dbt-fusion-developer-feedback}

> type: concept

Fusion-powered tooling can catch SQL issues earlier, trace column definitions, preview inline CTEs, and make large projects feel more interactive during development.

#### State-Aware Orchestration {#dbt-state-aware-orchestration}

> type: concept
> edge.related: dbt-rebuild-surface

State-aware orchestration uses code and data-change awareness to skip models that do not need rebuilding. It is a platform feature tied to Fusion and depends on metadata quality.

#### Fusion Adoption Boundary {#dbt-fusion-adoption-boundary}

> type: principle

Adopt Fusion by checking adapter support, feature coverage, CI behavior, package compatibility, and run semantics. Treat the engine as a compatibility surface, not only a faster binary.

### Governance Surface {#dbt-governance-surface}

> type: category

The mature dbt surface is a governed graph: owners, groups, access, contracts, tests, docs, exposures, semantic definitions, and artifacts all describe who can depend on what.

#### Ownership Is A Graph Property {#dbt-ownership-graph-property}

> type: principle

Ownership should follow dependency boundaries. If many teams depend on a model, the model needs stronger contract, documentation, and change-management expectations.

#### Semantic Layer Boundary {#dbt-semantic-layer-boundary}

> type: concept

The Semantic Layer moves metric definitions into dbt metadata so downstream tools query governed business concepts rather than re-implementing measures independently.
