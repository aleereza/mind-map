# Terraform for Snowflake {#terraform-snowflake}

> type: category

The Snowflake provider lets Terraform create, change and drop Snowflake objects. Snowflake maintains it under the source address `snowflakedb/snowflake` ([provider docs](https://registry.terraform.io/providers/snowflakedb/snowflake/latest/docs)). The provider works in SQL. It opens a session, runs statements such as `CREATE WAREHOUSE` or `GRANT`, and reads objects back with `SHOW` and `DESCRIBE`. The [Terraform map](../terraform.md) explains providers, state and plans in general.

The unit is one Snowflake account, seen through one session. Each provider block opens one session, as one user with one role. In Snowflake, the role that creates an object owns it ([object ownership](https://docs.snowflake.com/en/user-guide/security-access-control-overview)). So the provider role owns everything that Terraform creates ([One Session, One Role](#one-session-one-role-one-session-one-role)).

The provider has resources of two kinds ([Objects and Edges](#objects-and-edges-objects-and-edges)):

- An object resource stands for one Snowflake object, such as a database, a warehouse, a schema or a storage integration. Its destroy drops the object.
- An edge resource connects objects that it does not own. Grants, ownership transfers and tag associations are edges. Their destroy revokes or unsets.

Four more ideas explain most plans:

- Names are exact, case-sensitive strings. Each resource ID is built from them ([Names Are Exact Strings](#names-are-exact-strings-names-are-exact-strings)).
- The state holds only the arguments that you set. Snowflake keeps the defaults, and the provider shows them in read-only outputs ([State Holds Only What You Set](#state-holds-only-what-you-set-state-holds-only-what-you-set)).
- A grant resource can grant on one object, on all objects that exist now, on future objects, or by inheritance. The four kinds look alike in code, but they act differently ([Four Kinds of Grant](#four-kinds-of-grant-four-kinds-of-grant)).
- Resources come in stability tiers, and Snowflake supports only the newest stable version ([Versions and Stability](#versions-and-stability-versions-and-stability)).

Snowflake also acts in AWS. An integration lets Snowflake assume an IAM role, and the role must trust Snowflake back ([Two-Step Trust with AWS](#two-step-trust-with-aws-two-step-trust-with-aws)). Last, the resource `snowflake_execute` runs any SQL that you give it. It manages objects that have no resource of their own, and it detects no drift ([Raw SQL Pitfalls](#raw-sql-pitfalls-raw-sql-pitfalls)).

Now compare dbt. A common split gives Terraform the containers and the access: databases, schemas, warehouses, roles, grants and integrations. dbt then builds tables and views inside them at each run. The provider's design notes call it common to run tools such as dbt on infrastructure that Terraform created ([grants design](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/grants_redesign_design_decisions.md#L15)).

The map has four parts. Mental Models explains the ideas above. Reading a Snowflake Configuration says what you will see and where to look first. Pitfalls lists the failures that experienced users plan around. Good and Bad Decisions compares the main design choices.

## Mental Models {#snowflake-mental-models}

> type: category

The models come in dependency order: each model uses only the models before it.

### One Session, One Role {#one-session-one-role}

> type: concept

A `provider "snowflake"` block holds the session settings: organization, account, user, authenticator, role and warehouse. The `role` and `warehouse` arguments are the defaults of the session ([provider configuration](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/index.md#L167-L177)). Every statement that Terraform runs uses this role. The role owns each object that it creates, so later grants and drops also need it.

Snowflake divides the admin work between system roles ([access control considerations](https://docs.snowflake.com/en/user-guide/security-access-control-considerations)):

- USERADMIN creates and manages users and roles.
- SECURITYADMIN holds the global MANAGE GRANTS privilege. It can grant or revoke any privilege in the account.
- SYSADMIN creates warehouses, databases and database objects. Custom roles should sit under SYSADMIN.

The same page advises against ACCOUNTADMIN for automated scripts. It also warns that by default "not even the ACCOUNTADMIN role can modify or drop objects" that a custom role created, unless that role is granted up the hierarchy.

To act as several roles, a configuration declares several provider blocks with aliases. A resource then picks its session with `provider = snowflake.<alias>`. The provider docs forbid `USE ROLE` and `USE WAREHOUSE` inside SQL, because they make the provider's results inconsistent ([execute warning](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/execute.md#L12)).

### Objects and Edges {#objects-and-edges}

> type: concept

An object resource manages one Snowflake object, such as `snowflake_database` or `snowflake_warehouse`. It has a computed `fully_qualified_name`. An edge resource manages a relation between objects that it does not own, so it has no such field ([identifier guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/identifiers_rework_design_decisions.md#L62-L65)). The provider has one edge resource for each kind of grant ([grant resources](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/grants_redesign_design_decisions.md#L17-L30)):

- `snowflake_grant_privileges_to_account_role` grants privileges to an account role.
- `snowflake_grant_privileges_to_database_role` grants privileges to a database role.
- `snowflake_grant_account_role` and `snowflake_grant_database_role` grant a role to a role or a user.
- `snowflake_grant_ownership` moves ownership of objects to another role.
- `snowflake_grant_privileges_to_share` and `snowflake_grant_application_role` cover shares and apps.

An edge depends on objects that may live outside Terraform. A grant can name a warehouse that someone made by hand, or a table that dbt made. The pitfall Grants on Unmanaged Objects shows what happens when such an object disappears.

### Names Are Exact Strings {#names-are-exact-strings}

> type: concept

> edge.related: case-sensitive-names

Every field that holds an identifier is case-sensitive ([identifier guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/identifiers_rework_design_decisions.md#L100-L106)). In Snowflake SQL, an unquoted name turns into upper case. So `name = "TEST"` and `name = "test"` are two different objects, and the second one needs quotes in every query. Quotes alone, however, cause no diff. The provider ignores the difference between `AGENT_WH` and `"AGENT_WH"` ([quoting](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/identifiers_rework_design_decisions.md#L56-L60)).

The resource ID follows one rule. If one Snowflake identifier describes the object, the ID is its fully qualified name. Otherwise the ID joins all needed parts with pipes ([ID format](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/identifiers_rework_design_decisions.md#L84-L88)). Import blocks use the same strings:

| Resource | Import ID |
| --- | --- |
| Account object, such as a warehouse | `'"NAME"'` |
| Schema | `'"DB"."SCHEMA"'` |
| Schema object, such as a typed stage | `'"DB"."SCHEMA"."OBJ"'` |
| Grant to an account role | seven parts with pipes, as in the example below |
| `snowflake_execute` | a random UUID ([execute import](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/execute.md#L154-L160)) |

About the Sources shows how this grant was imported from a real account and planned:

```text
"AGENT_READ_ONLY"|false|false|USAGE|OnAccountObject|WAREHOUSE|"AGENT_WH"
```

The seven parts are the role, `with_grant_option`, `always_apply`, the privileges, the kind of target, the object type and the object name. The configuration said `account_role_name = "AGENT_READ_ONLY"` without quotes. The state stored `"\"AGENT_READ_ONLY\""` with quotes, and the plan still reported "No changes".

To refer to another object, read its `fully_qualified_name` instead of joining names by hand ([identifier guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/identifiers_rework_design_decisions.md#L62-L83)).

### State Holds Only What You Set {#state-holds-only-what-you-set}

> type: concept

The provider does not store an optional argument that the configuration leaves out ([config values in state](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/v1-preparations/CHANGES_BEFORE_V1.md#L46-L51)). Snowflake then keeps its own default for it. To show those values, each resource has three read-only outputs ([raw output](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/v1-preparations/CHANGES_BEFORE_V1.md#L53-L59)):

- `show_output` holds the result of `SHOW <objects>`.
- `describe_output` holds the result of `DESCRIBE <object>`.
- `parameters` holds the result of `SHOW PARAMETERS IN <object>`.

Some arguments use special values for "not set" ([empty values](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/v1-preparations/CHANGES_BEFORE_V1.md#L29-L34)):

- `"default"` for a boolean that the provider stores as a string.
- `-1` for a number whose valid range includes 0, such as `auto_suspend`.
- `"<Snowflake Terraform Provider string default>"` for a string that can be empty.

These values appear in plans. A plan that sets an argument to `"default"` asks Snowflake to go back to its own default. The [Terraform map](../terraform.md#outside-changes-get-reverted-outside-changes-get-reverted) shows a real warehouse plan with such lines. To keep a value, write it in the configuration.

### Four Kinds of Grant {#four-kinds-of-grant}

> type: concept

> edge.related: schema-level-future-grants-win, all-grants-revoke-widely

`snowflake_grant_privileges_to_account_role` grants in one of four ways. Three of them work like this:

| Kind | Code | What it grants | What destroy does |
| --- | --- | --- | --- |
| One object | `on_account_object`, or `on_schema_object` with `object_name` | Privileges on one named object | Revokes them |
| All | `on_schema_object { all { ... } }` | Privileges on every object of a type that exists at apply time | Revokes on every current object |
| Future | `on_schema_object { future { ... } }` | A rule for objects created later | Removes the rule and keeps the grants it made |

The provider treats a future grant as a trigger ([on_future decision](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/grants_redesign_design_decisions.md#L85-L91)). It does not track which objects an `all` grant touched ([always_apply decision](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/grants_redesign_design_decisions.md#L77-L84)). So configurations pair an `all` grant with a `future` grant: the first covers today's objects, and the second covers tomorrow's.

The fourth kind is the inherited grant, which the provider supports since 2.19.0 with an `inherited` block. One inherited grant covers every current and future object of a type in a container. The provider's migration guide calls the `all` plus `future` pair a common pattern that an inherited grant replaces ([inherited grants](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/MIGRATION_GUIDE.md#L1986-L1990)). Inherited grants are a preview feature in Snowflake. They allow no exception for a single object: a `REVOKE` on one table appears to succeed, and the role keeps the privilege ([inherited grants](https://docs.snowflake.com/en/user-guide/inherited-grants-intro)).

By default, a grant resource manages only the privileges that it granted. Other grants on the same object stay. The argument `strict_privilege_management` makes the resource revoke privileges that it did not grant ([design decisions](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/grants_redesign_design_decisions.md#L67-L75)).

Future grants in a database need the global MANAGE GRANTS privilege. By default, only SECURITYADMIN and ACCOUNTADMIN hold it ([GRANT privilege](https://docs.snowflake.com/en/sql-reference/sql/grant-privilege)).

### Versions and Stability {#versions-and-stability}

> type: concept

> edge.related: old-pins-and-preview-resources

The provider's resources come in three tiers:

- Stable resources are supported.
- Preview resources work only when you list them in `preview_features_enabled`. They can break in a minor version, and Snowflake does not support them ([disclaimer](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/index.md#L17)).
- Experiments change provider behavior. Each one starts as opt-in, then becomes the default, and then is promoted ([experiments](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/index.md#L998-L1016)).

Snowflake supports only the latest stable version of the provider, from 2.0.0 onward ([Snowflake Terraform provider](https://docs.snowflake.com/en/user-guide/terraform)). Fixes for Snowflake behavior changes therefore arrive only in new versions. The version pin in `required_providers` is the support boundary.

The source address changed twice. It was `chanzuckerberg/snowflake`, then `Snowflake-Labs/snowflake` from 2022-05-25 ([CZI upgrade](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/CZI_UPGRADE.md#L3-L7)), and now `snowflakedb/snowflake`. A team moves its state to the new address with `terraform state replace-provider Snowflake-Labs/snowflake snowflakedb/snowflake` ([namespace migration](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/SNOWFLAKEDB_MIGRATION.md#L3-L9)). The old address stops at version 1.0.5.

### Two-Step Trust with AWS {#two-step-trust-with-aws}

> type: concept

> edge.related: external-id-resets

Three Snowflake objects act in AWS through an IAM role: a storage integration for stages and Snowpipe, an external volume for Iceberg tables, and a notification integration. The [Iceberg in Snowflake map](../../snowflake/iceberg/snowflake-iceberg.md) explains external volumes and catalog integrations.

Each side needs a value from the other side. Snowflake must know the role ARN. The role's trust policy must name Snowflake's IAM user and an external ID. Snowflake uses one IAM user for each account ([storage integration guide](https://docs.snowflake.com/en/user-guide/data-load-s3-config-storage-integration)). Configurations break this loop in one of three ways:

1. They apply twice. Someone copies Snowflake's IAM user ARN and external ID into the AWS code between the two applies.
2. They fix the role ARN in advance by a naming rule. The integration is created first, and the trust policy then reads `describe_output` of the integration.
3. They choose the external ID themselves with `STORAGE_AWS_EXTERNAL_ID`. Both sides then know every value before the first apply. Snowflake documents this option so that one external ID can serve several integrations and volumes.

The [AWS map](../aws/terraform-aws.md) explains the IAM side of the trust.

## Reading a Snowflake Configuration {#reading-snowflake-configuration}

> type: category

These nodes say what you will find in the files, in which order to read them, and what each kind of plan line means.

### What You Will See {#snowflake-what-you-will-see}

> type: concept

| Resource | Snowflake object | What to check |
| --- | --- | --- |
| `snowflake_database`, `snowflake_schema` | database, schema | `is_transient` and a schema's `database` force replacement. Look for `prevent_destroy`, and for schemas that dbt also writes. |
| `snowflake_warehouse` | warehouse | `warehouse_size`, `auto_suspend`, `warehouse_type` |
| `snowflake_account_role`, `snowflake_grant_account_role` | role and role hierarchy | Is each custom role granted up to SYSADMIN? |
| `snowflake_grant_privileges_to_account_role` | privileges | Which of the four kinds of grant? `always_apply`? |
| `snowflake_grant_ownership` | ownership | Which role owns the objects after the apply? |
| `snowflake_service_user` | user with `TYPE = SERVICE` | Key pair or token, and its network policy |
| `snowflake_storage_integration_aws`, `snowflake_external_volume`, `snowflake_notification_integration` | integrations with AWS | Where do the role ARN and the external ID come from? |
| `snowflake_catalog_integration_open_catalog` | catalog integration | Where does the OAuth secret come from? |
| `snowflake_stage_external_s3` | external stage | A change of `directory.auto_refresh` recreates the stage. |
| `snowflake_tag`, `snowflake_tag_association` | tag and its use | Each association ID should be unique. |
| `snowflake_execute` | any SQL | Do `execute` and `revert` match? Any secrets? |

Three older resources are deprecated: `snowflake_stage`, `snowflake_storage_integration` and `snowflake_file_format` ([deprecated resources](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/index.md#L989-L995)). Typed resources, such as `snowflake_stage_external_s3`, replace the first two.

### Where to Look First {#snowflake-where-to-look-first}

> type: concept

Read a Snowflake configuration in this order:

1. `required_providers`. The source tells you the namespace, and the version tells you how far the code is from supported behavior. A committed `.terraform.lock.hcl` tells you the exact build.
2. Each `provider "snowflake"` block. Note the authenticator, the role, the warehouse, the aliases, `preview_features_enabled` and `experimental_features_*`.
3. The resource types in use. Mark the deprecated and preview types, and count the `snowflake_execute` blocks.
4. Each `snowflake_execute`. Check that `revert` undoes `execute`. Look for `IF NOT EXISTS`, `OR REPLACE` and secrets in the SQL.
5. The grants. For each one, find its kind, its level (database or schema) and the role that owns its target.
6. The `lifecycle` blocks. Look for `prevent_destroy`, `ignore_changes` and `replace_triggered_by`.
7. The integrations and their IAM roles. Find the source of each external ID and IAM user ARN.
8. The identifier strings. Check the case and any names joined by hand.
9. The `import`, `moved` and `removed` blocks. They show what Terraform adopted and what it renamed.
10. The backend and workspaces. Find which state, and so which databases, each run touches.

### Plan Signals {#snowflake-plan-signals}

> type: concept

| Line in the plan | Meaning |
| --- | --- |
| `-/+` on `snowflake_execute` | Any change to `execute` replaces the resource: Terraform runs `revert`, then `execute` ([execute.go](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/pkg/resources/execute.go#L63-L65)). |
| `"default"` or `-1` as a new value | The argument goes back to the Snowflake default. |
| `(known after apply)` on `describe_output` | An argument that feeds the output changed ([storage integration](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/pkg/resources/storage_integration_aws.go#L131-L132)). On schemas it can appear in every plan when the role cannot see the objects inside ([issue 3296](https://github.com/snowflakedb/terraform-provider-snowflake/issues/3296)). |
| A grant that appears in every plan | The grant sets `always_apply`, which grants again at each apply ([warning](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/grant_privileges_to_account_role.md#L9)). |
| `(sensitive value)` in place of SQL | A sensitive variable inside the SQL hides the whole string from review. |
| `# forces replacement` on a database, schema or integration | Terraform drops the object and creates it again. |

## Pitfalls {#snowflake-pitfalls}

> type: category

The pitfalls fall into four groups: raw SQL, grants, integrations, and lifecycle.

### Raw SQL Pitfalls {#raw-sql-pitfalls}

> type: concept

`snowflake_execute` takes three SQL strings: `execute` on create, `revert` on destroy, and an optional `query` on read. The provider calls it dangerous, because it runs any statement ([execute warning](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/execute.md#L10)).

#### No Drift Detection {#execute-no-drift-detection}

> type: warning

A refresh of `snowflake_execute` runs only `query` and stores the rows in `query_results` ([read code](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/pkg/resources/execute.go#L109-L161)). So if someone drops the object or revokes the grant by hand, the plan stays empty. A failing `query` gives only a warning ([note](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/execute.md#L18)). `query` also does not decide whether `execute` runs. With `CREATE ... IF NOT EXISTS`, a changed definition does nothing when the object exists. Check the real object with `SHOW` or `DESCRIBE`, and prefer a native resource when one exists.

#### An Edit Drops and Recreates {#execute-edit-recreates}

> type: warning

Any change to the `execute` text replaces the resource. Terraform runs `revert` and then the new `execute` ([execute.go](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/pkg/resources/execute.go#L63-L65); [delete](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/pkg/resources/execute.go#L163-L176)). A rotated OAuth secret inside a `CREATE CATALOG INTEGRATION` statement is such a change. Snowflake refuses to drop or replace a catalog integration while Iceberg tables use it ([DROP CATALOG INTEGRATION](https://docs.snowflake.com/en/sql-reference/sql/drop-catalog-integration)), so the apply fails halfway. `ALTER CATALOG INTEGRATION ... SET` changes the secret in place, and so does the native resource `snowflake_catalog_integration_open_catalog`.

#### Secrets in SQL Text {#execute-secrets-in-sql}

> type: warning

The provider does not mark `execute`, `revert`, `query` or `query_results` as sensitive ([sensitive values](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/execute.md#L14)). A secret inside the SQL is stored in the state in plain text. If the secret comes from a sensitive variable, Terraform hides the whole string, and reviewers see `(sensitive value)` instead of the SQL change.

### Grant Pitfalls {#grant-pitfalls}

> type: concept

These pitfalls change who can read what. Three of the four pass the apply without an error, and users notice them later as missing privileges.

#### Schema-Level Future Grants Win {#schema-level-future-grants-win}

> type: warning

When a database and one of its schemas both have future grants for the same object type, Snowflake uses only the schema-level grants. It ignores the database-level ones in that schema, for every role ([future grants](https://docs.snowflake.com/en/sql-reference/sql/grant-privilege)). So one schema-level future grant, made by anyone, silently turns off the database-level future grants of every other role there. New tables in that schema then lack the expected privileges. Check with `SHOW FUTURE GRANTS IN SCHEMA <db>.<schema>`. The same rule applies to future ownership since behavior change bundle BCR-1870 ([grant_ownership note](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/grant_ownership.md#L12)).

#### ALL Grants Revoke Widely {#all-grants-revoke-widely}

> type: warning

The provider never reads back an `all` grant, so it cannot show drift for it ([always_apply decision](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/grants_redesign_design_decisions.md#L77-L84)). Its destroy runs `REVOKE ... ON ALL`. When the provider role holds MANAGE GRANTS, Snowflake revokes every instance of the privilege, whoever granted it ([REVOKE privilege](https://docs.snowflake.com/en/sql-reference/sql/revoke-privilege)). So removing one `all` grant can also take away grants that dbt or a person made. `all_privileges` has the same blind spot ([read code](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/pkg/resources/grant_privileges_to_account_role.go#L1034-L1037)).

#### Renamed Grants Race {#renamed-grants-race}

> type: warning

A new address for a grant, for example a new `for_each` key, plans a destroy at the old address and a create at the new one. Terraform runs both at the same time, so the old `REVOKE` can land after the new `GRANT` and remove it ([issue 3826](https://github.com/snowflakedb/terraform-provider-snowflake/issues/3826)). A `moved` block avoids the race. A `moved` block cannot cross resource types in this provider. For a change of type, use `removed { lifecycle { destroy = false } }` and an `import` ([known issues](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/KNOWN_ISSUES.md#L31-L37); [migration guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/resource_migration.md#L33-L54)).

#### Grants on Unmanaged Objects {#grants-on-unmanaged-objects}

> type: warning

A grant can target an object that Terraform does not manage. If that object disappears, the refresh drops the grant from state with a warning, and the next plan creates it again. Destroying such a grant failed with "does not exist or not authorized" until the experiment `GRANTS_SAFE_DESTROY` became the default in 2.22.0 ([safe destroy](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/index.md#L1209-L1220)).

### Integration Pitfalls {#integration-pitfalls}

> type: concept

These pitfalls cross into AWS or into an Iceberg catalog. Terraform sees only the Snowflake side of each connection.

#### External ID Resets {#external-id-resets}

> type: warning

If a storage integration is created or recreated without an explicit external ID, it gets a new one. The IAM trust policy still names the old one, so every `AssumeRole` call fails ([storage integration guide](https://docs.snowflake.com/en/user-guide/data-load-s3-config-storage-integration)). The apply shows no error: queries and pipes fail afterwards. On `snowflake_storage_integration_aws`, a change of `name` or `storage_provider` forces such a replacement ([schema](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/pkg/resources/storage_integration_aws.go#L23-L41)). An `ignore_changes` on the external ID hides the change from the plan.

#### External Volumes Freeze {#external-volumes-freeze}

> type: warning

Snowflake refuses to drop or replace an external volume while Iceberg tables use it ([DROP EXTERNAL VOLUME](https://docs.snowflake.com/en/sql-reference/sql/drop-external-volume)). It also refuses to remove the active storage location ([provider note](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/resources/external_volume.md#L8)). So a plan that replaces a volume, for example through `replace_triggered_by`, fails at apply. Before 2.19.0, the provider could remove and re-add an unchanged location, which failed with error 393926 ([migration guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/MIGRATION_GUIDE.md#L2138-L2146)).

#### Catalog-Linked Databases Sync Late {#catalog-linked-databases-sync-late}

> type: warning

A catalog-linked database mirrors the namespaces of a remote Iceberg catalog. The provider has no resource for it yet, so configurations create it with `snowflake_execute` ([issue 4201](https://github.com/snowflakedb/terraform-provider-snowflake/issues/4201)). Snowflake syncs the tables in the background, every 30 seconds by default ([CREATE DATABASE (catalog-linked)](https://docs.snowflake.com/en/sql-reference/sql/create-database-catalog-linked)). Grants that run right after the create can therefore miss tables, so configurations add a wait. Its default `ALLOWED_WRITE_OPERATIONS = ALL` also sends table drops on to the remote catalog.

### Lifecycle Pitfalls {#lifecycle-pitfalls}

> type: concept

These pitfalls concern names, drops and versions.

#### Cascading Drops {#cascading-drops}

> type: warning

`DROP SCHEMA` uses `CASCADE` by default, so it drops every object in the schema ([DROP SCHEMA](https://docs.snowflake.com/en/sql-reference/sql/drop-schema)). A schema that dbt fills is therefore one destroy away from losing all its tables. Moving a schema to another database recreates it ([renaming guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/object_renaming_guide.md#L21-L47)). Recovery is a manual `UNDROP` within the Time Travel period, then an import ([restore guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/restoring_lost_data.md#L42-L50)). Time Travel keeps data for 1 day by default ([Time Travel](https://docs.snowflake.com/en/user-guide/data-time-travel)).

#### Case-Sensitive Names {#case-sensitive-names}

> type: warning

A lower-case name in Terraform creates a lower-case object. For example, `name = "tags"` creates the schema `"tags"`. Then `SHOW SCHEMAS LIKE 'TAGS'` and every unquoted `tags` in SQL miss it. The identifier guide shows the same trap with a role ([identifier guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/identifiers_rework_design_decisions.md#L100-L106)). Write names in upper case unless a lower-case name is intended.

#### Old Pins and Preview Resources {#old-pins-and-preview-resources}

> type: warning

An old provider pin keeps old bugs and old defaults:

- Before 2.23.0, a new warehouse without `warehouse_type` gets Snowflake's default type. Snowflake is changing that default from STANDARD to ADAPTIVE. Since 2.23.0 the provider sends STANDARD ([migration guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/MIGRATION_GUIDE.md#L307-L330)).
- Before 2.22.0, safe destroy of grants is opt-in (see Grants on Unmanaged Objects).
- A preview resource can break in a minor version. `snowflake_table` is one ([disclaimer](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/index.md#L17)).
- On the deprecated `snowflake_stage`, a change of `directory` recreates the stage, which can break its pipes ([issue 4059](https://github.com/snowflakedb/terraform-provider-snowflake/issues/4059)).
- Snowflake behavior change bundles (BCRs) can create drift. With BCR-2371 enabled, `SHOW GRANTS` reports an Iceberg table as `ICEBERG TABLE`. A database-role grant that names it with `object_type = "TABLE"` then shows a permanent diff ([BCR guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/SNOWFLAKE_BCR_MIGRATION_GUIDE.md#L69-L100)).

Each minor version has a section in the [migration guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/MIGRATION_GUIDE.md). Read it before you raise the pin.

## Good and Bad Decisions {#snowflake-decisions}

> type: category

Each node compares a choice with its alternative.

### Native Resources Before Raw SQL {#native-before-raw-sql}

> type: concept

A native resource reads the object back, shows drift and changes it in place. `snowflake_execute` does none of this. Use it only where no resource exists, such as for a catalog-linked database today. Catalog integrations have stable resources since 2.18.0 ([migration guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/MIGRATION_GUIDE.md#L2173)). Moving a catalog integration out of `snowflake_execute` removes the drop-and-recreate on each secret change.

### Grants as Resources or as SQL {#grants-as-resources-or-sql}

> type: concept

Some repos render grant SQL from a template and run it with `snowflake_execute`. The table compares this with native grant resources:

| | Native grant resources | Grant SQL in `snowflake_execute` |
| --- | --- | --- |
| Number of resources | One resource for each grant | One resource for many grants |
| Drift detection | Yes, except for `all` grants | No |
| Removing extra grants | Yes, with `strict_privilege_management` | No |
| Rename safety | `moved` blocks | Any text change revokes and grants again |

Native resources cost more lines, but they show the truth in the plan. Inherited grants since 2.19.0 replace each pair of `all` and `future` grants with one resource ([migration guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/MIGRATION_GUIDE.md#L1986-L1988)).

### One Role per Duty {#one-role-per-duty}

> type: concept

A single Terraform role with MANAGE GRANTS can do everything, so every root module and every environment shares its blast radius. Snowflake's advice splits the work: USERADMIN for users and roles, SECURITYADMIN for grants, and SYSADMIN for objects ([access control considerations](https://docs.snowflake.com/en/user-guide/security-access-control-considerations)). Terraform follows that split with one provider alias for each role. The cost is more provider blocks and more `provider =` arguments.

### Choose the External ID {#choose-the-external-id}

> type: concept

When you choose the external ID yourself, the integration and the IAM trust policy get it in the same apply. Nobody copies values, and a recreated integration keeps working. When Snowflake generates the ID, someone copies it into the trust policy, and every recreate breaks the trust (see External ID Resets). AWS does not treat an external ID as a secret ([AWS external ID](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios_third-party.html)), so it can sit in configuration.

### Protect What Cannot Be Rebuilt {#protect-what-cannot-be-rebuilt}

> type: concept

A database or a table that holds data cannot come back from code. `lifecycle { prevent_destroy = true }` makes any plan that destroys it fail. The cost is a deliberate config edit before a planned removal ([migration guide](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/docs/guides/resource_migration.md#L92-L94)). Pair it with `moved` blocks for renames, so a refactor never plans a destroy.

## About the Sources {#snowflake-about-the-sources}

> type: comment

The provider facts come from the `snowflakedb/snowflake` repository at tag v2.23.0, the latest release on 2026-11-23. The Snowflake facts come from docs.snowflake.com on the same day.

The grant example ran on 2026-11-23 with Terraform 1.15.8 and provider 2.23.0, against a real account and with a read-only role. Terraform only read the grant and wrote a local state file. The folder held the `terraform.tf`, `providers.tf` and `variables.tf` files from the About the Examples node of the [Terraform map](../terraform.md), and this `main.tf`:

```hcl
resource "snowflake_grant_privileges_to_account_role" "agent_wh_usage" {
  account_role_name = "AGENT_READ_ONLY"
  privileges        = ["USAGE"]

  on_account_object {
    object_type = "WAREHOUSE"
    object_name = "AGENT_WH"
  }
}
```

These commands ran with the same `TF_VAR_` environment variables as in that node:

```bash
terraform init -input=false
terraform import snowflake_grant_privileges_to_account_role.agent_wh_usage '"AGENT_READ_ONLY"|false|false|USAGE|OnAccountObject|WAREHOUSE|"AGENT_WH"'
terraform plan -input=false
terraform state show snowflake_grant_privileges_to_account_role.agent_wh_usage
```
