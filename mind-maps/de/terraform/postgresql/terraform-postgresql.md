# Terraform for PostgreSQL {#terraform-postgresql}

> type: category

The PostgreSQL provider lets Terraform manage the objects inside a PostgreSQL server: roles, databases, schemas, grants, default privileges and extensions. A community maintainer publishes it under the source address `cyrilgdn/postgresql` ([provider docs](https://registry.terraform.io/providers/cyrilgdn/postgresql/latest/docs)). It replaced HashiCorp's older provider ([README](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/README.md)). The [Terraform map](../terraform.md) explains Terraform itself.

The provider is a SQL client, not an API client. It opens a database connection as an admin role and runs SQL such as `CREATE ROLE` and `GRANT`. Even a refresh queries the system catalogs, so every plan needs a live connection to the server ([A SQL Client](#a-sql-client-a-sql-client)).

The unit is one server and, by default, one database. A cloud provider can run the server, for example Amazon RDS (Relational Database Service). Then two providers share the work ([Two Planes, Two Credentials](#two-planes-two-credentials-two-planes-two-credentials)):

- The AWS provider creates the RDS instance, through the AWS API.
- The PostgreSQL provider creates what lives inside the instance, through a database connection.

Inside the server, access follows three rules that shape every plan:

- A grant resource owns the full privilege set of one role on one kind of object in one schema ([A Grant Owns Its Set](#a-grant-owns-its-set-a-grant-owns-its-set)).
- The owner of an object has all privileges on it without any grant ([Owners Need No Grants](#owners-need-no-grants-owners-need-no-grants)).
- Default privileges apply to objects that one named role creates later ([Default Privileges Follow the Creator](#default-privileges-follow-the-creator-default-privileges-follow-the-creator)).

On RDS, the admin role is not a full superuser, which limits what the provider can do ([The RDS Admin Is Not a Superuser](#the-rds-admin-is-not-a-superuser-the-rds-admin-is-not-a-superuser)).

Now compare Snowflake. A Snowflake future grant and a PostgreSQL default privilege both cover objects that do not exist yet. Snowflake keys the rule on a database or schema; PostgreSQL keys it on the role that creates the objects. Default Privileges Follow the Creator compares the two.

The map has four parts. Mental Models explains the ideas above. Reading a PostgreSQL Configuration says what you will see and where to look first. Pitfalls lists the failures that experienced users plan around. Good and Bad Decisions compares the main design choices.

## Mental Models {#pg-mental-models}

> type: category

The models come in dependency order: each model uses only the models before it.

### A SQL Client {#a-sql-client}

> type: concept

> edge.related: plan-needs-the-database

Each read, create, update and delete of a resource first opens a connection to the server ([connect code](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/postgresql/helpers.go#L14-L36)). The provider settings say where and how ([provider settings](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/index.html.markdown#L141-L188)):

- `host`, `port`, `database`, `username` and `password`, or the environment variables `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER` and `PGPASSWORD`.
- `sslmode`, whose default `require` encrypts the connection but does not check the server certificate. `verify-full` checks it.
- `superuser`, which must be `false` on RDS.
- `expected_version`. Without it, each new connection runs `SELECT VERSION()` to detect the server's features ([config code](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/postgresql/config.go#L318-L327)).

So the machine that runs the plan must reach the server's port 5432 and log in with a valid password.

### Two Planes, Two Credentials {#two-planes-two-credentials}

> type: concept

The AWS API and IAM build the RDS instance. A database connection and a database role build what is inside it. Each plane needs its own credentials. A provider setting can use a data source, so the PostgreSQL provider can read its host or password from AWS ([provider docs](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/index.html.markdown#L118-L124)). It cannot safely use an attribute of an instance that the same run creates. So keep the instance and the objects inside it in separate root modules.

### A Grant Owns Its Set {#a-grant-owns-its-set}

> type: concept

> edge.related: grant-takes-the-whole-set

One `postgresql_grant` covers one combination of role, database, schema and object type, such as "role `app`, tables in schema `public`". On every create and update, it revokes all privileges of that combination and then grants its own list, in one transaction ([grant code](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/postgresql/resource_postgresql_grant.go#L194-L204)). So any other grant on the same combination, made by hand or by another resource, disappears at the next update.

`objects = []` means all objects of that type that exist at apply time. A refresh checks every one of them, so a table created later shows as a difference.

### Owners Need No Grants {#owners-need-no-grants}

> type: concept

> edge.related: owner-table-grant-drifts

The owner of a table has all privileges on it, and PostgreSQL's docs say there is no need to grant privileges to the owner ([GRANT](https://www.postgresql.org/docs/current/sql-grant.html)). An owner can, however, revoke its own privileges ([privileges](https://www.postgresql.org/docs/current/ddl-priv.html)). So a `REVOKE` that a grant resource runs against the owner really removes the owner's rights.

A table's access list shows these rights as letters. For example, `app=arwdxt/app` means that role `app` holds these privileges, granted by `app`:

| Letter | Privilege |
| --- | --- |
| `r` | SELECT |
| `a` | INSERT |
| `w` | UPDATE |
| `d` | DELETE |
| `D` | TRUNCATE |
| `x` | REFERENCES |
| `t` | TRIGGER |

This entry has no `D`, so `app` cannot run `TRUNCATE` on the table, even if it owns it ([ACL letters](https://www.postgresql.org/docs/current/ddl-priv.html)).

### Default Privileges Follow the Creator {#default-privileges-follow-the-creator}

> type: concept

> edge.related: default-privileges-miss-objects

`postgresql_default_privileges` runs `ALTER DEFAULT PRIVILEGES FOR ROLE <owner>`. It applies only to objects that the `owner` role creates after the rule exists ([resource doc](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/r/postgresql_default_privileges.html.markdown#L33)). PostgreSQL states that new objects are only affected by the default privileges of the role that creates them, and that existing objects do not change ([ALTER DEFAULT PRIVILEGES](https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html)).

| | PostgreSQL default privileges | Snowflake future grants |
| --- | --- | --- |
| Keyed by | The role that creates the objects | The database or schema that holds them |
| Existing objects | Unchanged; add a grant on existing tables | Unchanged; add an `all` grant |
| When two rules overlap | Schema-level rules add to the global ones | Schema-level rules replace database-level ones |

The Snowflake column comes from Snowflake's [GRANT docs](https://docs.snowflake.com/en/sql-reference/sql/grant-privilege); the [Snowflake map](../snowflake/terraform-snowflake.md#schema-level-future-grants-win-schema-level-future-grants-win) explains its trap.

### The RDS Admin Is Not a Superuser {#the-rds-admin-is-not-a-superuser}

> type: concept

On Amazon RDS, the master user is a member of the role `rds_superuser`, but it is not a real superuser ([RDS roles](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.Roles.rds_superuser.html)). The provider must know this: set `superuser = false`. Then some features are off; for example, the provider cannot read a role's password back from the database ([provider docs](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/index.html.markdown#L155-L156)). A password changed outside Terraform then never shows in a plan.

## Reading a PostgreSQL Configuration {#reading-pg-configuration}

> type: category

These nodes say what you will find in the files, in which order to read them, and what each kind of plan line means.

### What You Will See {#pg-what-you-will-see}

> type: concept

| Resource | PostgreSQL object | What to check | Import ID |
| --- | --- | --- | --- |
| `postgresql_role` | role, or a user when `login = true` | Password source, `roles` list | `name` |
| `postgresql_database` | database and its owner | Owner, `prevent_destroy` | `name` |
| `postgresql_schema` | schema | Owner | `db.schema` |
| `postgresql_extension` | extension, such as `uuid-ossp` | Which database | `db.extension` |
| `postgresql_grant` | privileges of a role on one kind of object | Grantee, object type, `objects`, `privileges` | No import |
| `postgresql_default_privileges` | rules for future objects | The `owner` role | No import |
| `postgresql_grant_role` | membership of a role in another role | Conflicts with the `roles` list of `postgresql_role` | No import |

### Where to Look First {#pg-where-to-look-first}

> type: concept

Read a PostgreSQL configuration in this order:

1. `required_providers` and the Terraform version. Provider 1.24.0 is "the first version compatible with Postgresql 16" ([v1.24.0](https://github.com/cyrilgdn/terraform-provider-postgresql/releases/tag/v1.24.0)). The ephemeral `password_wo` arrived in 1.26.0 ([v1.26.0](https://github.com/cyrilgdn/terraform-provider-postgresql/releases/tag/v1.26.0)) and needs Terraform 1.11.
2. The `provider "postgresql"` block, or its absence. Find where the host, user, password and `sslmode` come from, and whether `superuser = false` is set.
3. The pipeline. Find how the runner reaches port 5432, and which credentials each stage gets.
4. The environment inputs. Find which role is the app's user and owner in each environment.
5. `postgresql_role`: the password source, `ignore_changes`, and `skip_reassign_owned` or `skip_drop_role`.
6. `postgresql_database`: the owner, `prevent_destroy`, and the arguments that force a replacement.
7. `postgresql_grant`: is the grantee also the owner, is `objects` empty, and do two resources cover one combination?
8. `postgresql_default_privileges`: is `owner` the role that creates the tables?

### Plan Signals {#pg-plan-signals}

> type: concept

| Line in the plan | Meaning |
| --- | --- |
| The plan fails before any diff with `error detecting capabilities` or a timeout | The runner cannot reach the server, or the login fails. |
| `~ privileges` on a table grant with no code change | New tables appeared; the apply will run `REVOKE ALL` and grant again. |
| `-` or `-/+` on `postgresql_database` | The database will be dropped, with its data. |
| `-` on `postgresql_role` | The role's objects will be reassigned and dropped in the default database first. |

## Pitfalls {#pg-pitfalls}

> type: category

The pitfalls fall into three groups: connection, grants, and destroy.

### Connection Pitfalls {#pg-connection-pitfalls}

> type: concept

These pitfalls stop the plan or point it at the wrong server.

#### Plan Needs the Database {#plan-needs-the-database}

> type: warning

When the state holds PostgreSQL resources, the plan connects to the server. Without a network path or with a wrong password, it fails before any diff, and each attempt can wait up to the 180-second `connect_timeout` ([provider settings](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/postgresql/provider.go#L181-L187)). The provider has no built-in bastion option; a SOCKS5 proxy through `ALL_PROXY` works ([provider docs](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/index.html.markdown#L333-L337)). A runner inside the private network avoids the problem.

#### Workspace and Server Can Disagree {#workspace-and-server-disagree}

> type: warning

The workspace chooses the state, but the `PG*` environment variables choose the server. When the object names are the same in every environment, a local run with the wrong `PGHOST` changes the wrong server, and the plan looks normal. Set `host` from a data source of the environment's instance, so the workspace decides both.

### Grant Pitfalls {#pg-grant-pitfalls}

> type: concept

These pitfalls remove privileges without an error.

#### A Grant Takes the Whole Set {#grant-takes-the-whole-set}

> type: warning

Two `postgresql_grant` resources on the same combination of role, database, schema and object type revoke each other's privileges at each update. A grant made by hand on that combination disappears the same way. Give each combination exactly one resource, with every privilege it needs.

#### Owner Table Grant Drifts {#owner-table-grant-drifts}

> type: warning

Some configurations grant table privileges to the role that also owns the tables, while the app's migrations create new tables after each apply. New tables have no explicit access list, so the next refresh shows the privileges as missing, and the plan shows `~ privileges` with no code change. The apply then runs `REVOKE ALL` and grants the listed privileges again. If the list lacks `TRUNCATE`, the owner loses `TRUNCATE` on its existing tables ([issue 627](https://github.com/cyrilgdn/terraform-provider-postgresql/issues/627)). Do not grant table privileges to the owner: it already has them.

#### Default Privileges Miss Objects {#default-privileges-miss-objects}

> type: warning

A rule for future tables helps only when its `owner` is the role that really creates the tables. If migrations run as the app role, but the rule names the Terraform admin, new tables get no privileges for the reader, and queries fail with "permission denied". The plan shows nothing, because the rule itself is unchanged. Use one rule for each role that creates objects, and a grant for the tables that exist already.

### Destroy Pitfalls {#pg-destroy-pitfalls}

> type: concept

These pitfalls turn a refactor into a drop.

#### Dropping a Role That Owns Objects {#dropping-a-role-that-owns-objects}

> type: warning

When a `postgresql_role` leaves the configuration, the provider runs `REASSIGN OWNED` and `DROP OWNED` before `DROP ROLE` ([role doc](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/r/postgresql_role.html.markdown#L14-L22)). It does this only in the provider's database. If the role owns objects in another database, `DROP ROLE` fails. A rename of the role is an in-place `ALTER ROLE ... RENAME`. A new resource address without a `moved` block, however, plans a drop and a create.

#### Destroying a Database Forces Disconnects {#destroying-a-database-forces-disconnects}

> type: warning

A destroy of `postgresql_database` terminates the open connections and then runs `DROP DATABASE ... WITH (FORCE)` on PostgreSQL 13 and later ([database code](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/postgresql/resource_postgresql_database.go#L240-L290)). A drop cannot be undone ([DROP DATABASE](https://www.postgresql.org/docs/current/sql-dropdatabase.html)). A change of `template`, `encoding`, `lc_collate` or `lc_ctype` also replaces the database. Add `prevent_destroy` to every database with data.

#### Passwords in State {#pg-passwords-in-state}

> type: warning

All arguments of `postgresql_role`, including the password, sit in the state in plain text ([role doc](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/r/postgresql_role.html.markdown#L24)). So does the master password of an RDS instance that the AWS provider creates. `password_wo` keeps a role password out of the state ([role doc](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/r/postgresql_role.html.markdown#L27-L29)).

## Good and Bad Decisions {#pg-decisions}

> type: category

Each node compares a choice with its alternative.

### One Login Role per App {#one-login-role-per-app}

> type: concept

Give each app its own login role, and let that role own the app's database. The predefined role `pg_database_owner` owns the `public` schema, so each database owner governs that schema ([predefined roles](https://www.postgresql.org/docs/current/predefined-roles.html)). Dashboards and other readers then get their own roles with only `SELECT`. Sharing the RDS master user between apps and dashboards is simpler, but one leaked password then opens every database on the instance.

### Separate Migration and Reader Roles {#separate-migration-and-reader-roles}

> type: concept

The role that runs migrations creates the tables, so it owns them. A reader role gets a grant on the tables that exist and a default-privileges rule `FOR ROLE <migration role>` for the tables to come. This pair covers past and future tables, and nothing in it drifts when migrations add tables.

### Reach the Database Privately {#reach-the-database-privately}

> type: concept

A public database endpoint with an allowlist for the runner's address is quick to set up, but it exposes the server to the internet. A runner inside the private network, or a proxy, keeps port 5432 private. With `sslmode = "verify-full"`, the provider also checks that it talks to the right server ([provider settings](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/index.html.markdown#L157-L163)).

### Close the Default Doors {#close-the-default-doors}

> type: concept

PostgreSQL grants `CONNECT` and `TEMPORARY` on every new database to `PUBLIC`, which means every role ([privileges](https://www.postgresql.org/docs/current/ddl-priv.html)). So any login role on a shared server can connect to every app's database. The grant resource docs show how to revoke these defaults ([grant doc](https://github.com/cyrilgdn/terraform-provider-postgresql/blob/v1.27.0/website/docs/r/postgresql_grant.html.markdown#L57-L66)).

## About the Sources {#pg-about-the-sources}

> type: comment

The provider facts come from the `cyrilgdn/postgresql` repository at tag v1.27.0, the latest release on 2026-11-23. The PostgreSQL facts come from postgresql.org/docs/current, and the RDS facts from docs.aws.amazon.com, on the same day.
