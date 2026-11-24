# Terraform for dbt Cloud {#terraform-dbt-cloud}

> type: category

The dbt Cloud provider lets Terraform manage the objects of a dbt Cloud account: projects, connections, credentials, environments, environment variables and jobs. dbt Labs maintains it under the source address `dbt-labs/dbtcloud` ([provider docs](https://registry.terraform.io/providers/dbt-labs/dbtcloud/latest/docs)). It started as the community provider `GtheSheep/dbt-cloud` ([README](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/README.md#L5-L7)). The provider calls the dbt Cloud API with a token. The [Terraform map](../terraform.md) explains Terraform itself, and the [dbt map](../../dbt/dbt-deep-mental-model.md) explains dbt.

The unit is one dbt Cloud account. The provider block names the account ID, the API host and the token ([provider configuration](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/index.md#L44-L54)). Your dbt models live in Git. The dbt Cloud objects say where, how and when those models run. Terraform manages these run settings, not the models.

The objects sit at two levels ([Account and Project](#account-and-project-account-and-project)):

- The account holds objects that many projects share, such as global connections to a data platform, service tokens and user groups.
- A project holds a Git repository, credentials, environments, environment variables and jobs.

Inside a project, two objects do most of the work:

- An environment decides where and how the code runs: the connection, the credential, the dbt version and the Git branch ([The Environment Decides Where](#the-environment-decides-where-the-environment-decides-where)).
- A job decides what runs and when: the dbt commands, the triggers and the target name. Each job belongs to one environment.

Three more ideas explain most plans. The type of an environment switches dbt features on and off ([Environment Types Are Switches](#environment-types-are-switches-environment-types-are-switches)). The provider reads most settings back from dbt Cloud, but never secrets ([What the Provider Reads Back](#what-the-provider-reads-back-what-the-provider-reads-back)). And provider versions 0.x and 1.x have different schemas, so the docs must match the pinned version ([Two Provider Generations](#two-provider-generations-two-provider-generations)).

Now compare Snowflake. A dbt Cloud credential names a Snowflake user, role, database and warehouse as plain text. Other Terraform, or a person, creates those Snowflake objects. The [Snowflake map](../snowflake/terraform-snowflake.md) covers them.

The map has four parts. Mental Models explains the ideas above. Reading a dbt Cloud Configuration says what you will see and where to look first. Pitfalls lists the failures that experienced users plan around. Good and Bad Decisions compares the main design choices.

## Mental Models {#dbtcloud-mental-models}

> type: category

The models come in dependency order: each model uses only the models before it.

### Account and Project {#account-and-project}

> type: concept

A global connection, `dbtcloud_global_connection`, belongs to the account and holds the details of one data platform, such as a Snowflake account. A project does not point to a connection. Each environment of the project does, with `connection_id` ([environment doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/environment.md#L72)). So one connection can serve many projects.

A project holds the rest:

- `dbtcloud_repository` names the Git repository. `dbtcloud_project_repository` links it to the project.
- A credential, such as `dbtcloud_snowflake_credential`, holds the user, role, schema and secret that one deployment environment uses.
- `dbtcloud_environment` combines a connection, a credential and code settings.
- `dbtcloud_environment_variable` holds one variable with a value for each environment.
- `dbtcloud_job` holds the commands that run in one environment.

Provider version 1.0.0 removed the older connection resources, such as `dbtcloud_connection` and `dbtcloud_project_connection`. `dbtcloud_global_connection` replaces them ([upgrade guide](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/guides/90_upgrading_from_v0_to_v1.md#L7-L12)).

### The Environment Decides Where {#the-environment-decides-where}

> type: concept

An environment is either `development` or `deployment`. A project has one development environment and as many deployment environments as needed ([environment doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/environment.md#L11)). Each environment sets four things ([arguments](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/environment.md#L72-L83)):

- `connection_id`: the data platform.
- `credential_id`: the identity for deployment runs. A development environment has none, because each developer sets their own credentials in dbt Cloud.
- `dbt_version`: for example `latest` or `versionless`.
- `custom_branch`: the Git branch to run.

A job adds what to run and when: `execute_steps` holds the dbt commands, `triggers` holds the trigger flags, and `target_name` sets the value of `target.name` in the models ([job doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/job.md#L216-L246)). A model that checks `target.name` therefore behaves according to the job, not the environment.

### Environment Types Are Switches {#environment-types-are-switches}

> type: concept

A deployment environment has a `deployment_type`: `production`, `staging`, or empty for a general environment ([environment doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/environment.md#L76)). The value is more than a label. dbt uses it to decide how references between projects resolve in dbt Mesh, and it sets the variable `DBT_CLOUD_ENVIRONMENT_TYPE` in each run ([project dependencies](https://docs.getdbt.com/docs/mesh/govern/project-dependencies); [special variables](https://docs.getdbt.com/docs/build/environment-variables)). That variable is `prod`, `staging`, or empty for a general environment.

The environment `type`, development or deployment, cannot change in place. A new value replaces the environment ([schema](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/pkg/framework/objects/environment/schema.go#L234-L240)), and the pitfall Replacements Delete Jobs shows what follows.

### What the Provider Reads Back {#what-the-provider-reads-back}

> type: concept

> edge.related: ui-edits-get-reverted

A refresh reads most settings of jobs and environments from dbt Cloud. So an edit in the dbt Cloud UI to such a setting shows in the next plan, and the apply reverts it. The [Terraform map](../terraform.md#outside-changes-get-reverted-outside-changes-get-reverted) shows the same effect with Snowflake.

Secrets are the exception. The dbt Cloud API masks them, so the provider cannot read them back. For an environment variable whose name starts with `DBT_ENV_SECRET`, the provider keeps the values from its own state ([environment variable code](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/pkg/framework/objects/environment_variable/resource.go#L161-L182)). A secret that someone changes in the UI therefore never shows in a plan, and the next apply does not restore it either.

### Two Provider Generations {#two-provider-generations}

> type: concept

> edge.related: version-gap

Provider 1.0.0 rewrote the provider on HashiCorp's newer Plugin Framework. It changed the schema of several resources and removed the old connection resources ([upgrade guide](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/guides/90_upgrading_from_v0_to_v1.md)). The Terraform Registry shows the docs of the latest version by default. A configuration pinned to 0.x needs the 0.x docs, which GitHub keeps under the tag, for example [v0.3.26](https://github.com/dbt-labs/terraform-provider-dbtcloud/tree/v0.3.26/docs).

Some arguments still exist but are deprecated:

- `triggers.custom_branch_only`: the docs say to remove it; jobs now use the environment's branch ([job doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/job.md#L216)).
- `timeout_seconds` on a job: use `execution.timeout_seconds` instead ([job doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/job.md#L244)).
- `deferring_job_id`: use `deferring_environment_id` instead ([job doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/job.md#L14)).

## Reading a dbt Cloud Configuration {#reading-dbtcloud-configuration}

> type: category

These nodes say what you will find in the files, in which order to read them, and what each kind of plan line means.

### What You Will See {#dbtcloud-what-you-will-see}

> type: concept

| Resource | dbt Cloud object | What to check | Import ID |
| --- | --- | --- | --- |
| `dbtcloud_project` | project | Its name and subdirectory | `project_id` |
| `dbtcloud_global_connection` | connection to a data platform | Snowflake account, OAuth settings, how other code finds it | `connection_id` |
| `dbtcloud_repository`, `dbtcloud_project_repository` | Git repository and its link | Remote URL and provider | `project_id:repository_id` |
| `dbtcloud_snowflake_credential` | deployment credential | `auth_type`, database, schema, role, warehouse, secret source | `project_id:credential_id` |
| `dbtcloud_environment` | environment | `type`, `deployment_type`, `connection_id`, `credential_id`, `dbt_version` | `project_id:environment_id` |
| `dbtcloud_environment_variable` | variable with values per environment | Secret prefix, environment names as keys | `project_id:NAME` |
| `dbtcloud_job` | job | `execute_steps`, `triggers`, `target_name`, `num_threads` | `job_id` |

The import IDs come from the Import section of each [resource doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/tree/v1.13.1/docs/resources). A Terraform configuration can create jobs whose IDs other systems need, for example a pipeline that starts runs through the API. Find where the configuration outputs those IDs.

### Where to Look First {#dbtcloud-where-to-look-first}

> type: concept

Read a dbt Cloud configuration in this order:

1. `required_providers` and the lock file. A 0.x pin means the Registry docs do not match; read the docs at the pinned tag.
2. The `provider "dbtcloud"` block. Check `host_url` and where the token comes from.
3. The backend and the workspaces. Find which environments share one state, because one plan then touches all of them.
4. `dbtcloud_project`, and the `import` and `moved` blocks. They show which objects were first made in the UI.
5. The connection, or the data source that finds it, and the `connection_id` of every environment.
6. The environments: `type`, `deployment_type`, the credential wiring, `dbt_version` and the branch.
7. The credentials: `auth_type`, the Snowflake names, and where the private key comes from.
8. The jobs: `execute_steps`, `target_name`, `triggers`, the schedule fields and `num_threads`.
9. The environment variables, and which values are secret.
10. The outputs and pipelines that use job IDs, and the Snowflake code that creates the named database, role and warehouse.

### Plan Signals {#dbtcloud-plan-signals}

> type: concept

| Line in the plan | Meaning |
| --- | --- |
| `~ triggers` or `~ schedule_*` on a job without a code change | Someone changed the job in the UI; the apply will revert it. |
| `-/+ dbtcloud_environment` | The environment is replaced; dbt deletes its jobs with it. |
| `-/+ dbtcloud_job ... triggers` | The job changes type between CI, merge and other jobs; it gets a new ID. |
| `+ credential_id = (known after apply)` right after an upgrade to 1.x | The new version stores empty values as null instead of 0 ([issue 519](https://github.com/dbt-labs/terraform-provider-dbtcloud/issues/519)). |
| `target_name = "default"` on a new job | The job uses the provider default; models that check `target.name` may skip production logic. |

## Pitfalls {#dbtcloud-pitfalls}

> type: category

The pitfalls fall into three groups: drift, replacements, and job settings.

### Drift Pitfalls {#dbtcloud-drift-pitfalls}

> type: concept

These pitfalls come from the split between the UI and the code.

#### UI Edits Get Reverted {#ui-edits-get-reverted}

> type: warning

dbt Cloud lets anyone with the right role edit a job in the UI. When the job is in Terraform, the next apply reverts that edit, with no warning beyond the plan line. dbt Cloud keeps the settings history of each job for 90 days, so you can see what changed ([deploy jobs](https://docs.getdbt.com/docs/deploy/deploy-jobs)). Make every change to a Terraform-managed job in code.

#### Secrets in State and Out of Sight {#secrets-in-state}

> type: warning

The private key and passphrase of a credential are stored in the state in plain text; `sensitive` only hides them in plan output ([credential doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/snowflake_credential.md#L62)). The values of environment variables are not even marked sensitive ([variable doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/environment_variable.md#L39)). Newer 1.x versions add write-only arguments such as `private_key_wo`, which keep a secret out of the state; they need Terraform 1.11 or later ([credential doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/snowflake_credential.md#L26-L27)). And a secret changed in the UI stays invisible to Terraform, as What the Provider Reads Back explains.

#### Lookups by Name {#lookups-by-name}

> type: warning

A configuration can find a global connection by its name, with the data source `dbtcloud_global_connections` and a filter. If someone renames the connection, the lookup returns nothing and the environment create fails. Two connections can also share one name; the provider's changelog suggests a check that names do not repeat ([changelog](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v0.3.26/CHANGELOG.md#L118-L121)). Pass the connection ID from the state that owns the connection, or keep both in one state.

### Replacement Pitfalls {#dbtcloud-replacement-pitfalls}

> type: concept

These pitfalls delete jobs or change their IDs.

#### Replacements Delete Jobs {#replacements-delete-jobs}

> type: warning

dbt deletes an environment's jobs together with the environment ([delete an environment](https://docs.getdbt.com/faqs/Environments/delete-environment-job)). So a plan that replaces an environment, for example after a change of `type`, also takes its jobs with it. A job also gets a new ID when its triggers move it between kinds: CI, merge, or other ([job code](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/pkg/framework/objects/job/resource.go#L128-L157)). Any pipeline or service that starts the job by ID then calls a job that no longer exists.

#### Setting is_active to False Deletes the Job {#is-active-false-deletes}

> type: warning

`is_active = false` sounds like a pause, but the docs say it "is the same as creating a job in a deleted state" ([job doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/job.md#L230)). To pause a job, set every trigger to `false` instead ([triggers](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/job.md#L216)).

#### Authoritative Group Lists {#authoritative-group-lists}

> type: warning

`dbtcloud_user_groups` owns the full list of groups of a user. It removes any group that someone added in the UI ([user groups doc](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/resources/user_groups.md#L17)). A team that manages some groups in code and others by hand loses the hand-made ones at the next apply.

### Job Setting Pitfalls {#dbtcloud-job-setting-pitfalls}

> type: concept

These pitfalls come from defaults that differ from the UI.

#### Job Defaults Differ from the UI {#job-defaults-differ}

> type: warning

The provider has its own defaults for two job settings ([job schema](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/pkg/framework/objects/job/schema.go#L541-L553)):

- `target_name` defaults to `"default"`. A macro or model that runs only when `target.name == 'prod'` silently skips that logic.
- `num_threads` defaults to 1. The UI suggests 4 threads ([deploy jobs](https://docs.getdbt.com/docs/deploy/deploy-jobs)), so a Terraform job without the setting runs one model at a time.

Set both on every job.

#### Jobs Started by the API {#jobs-started-by-api}

> type: warning

A job with all triggers off runs only when something calls the API. The scheduler queues such runs, and it does not cancel over-scheduled runs that the API started ([job scheduler](https://docs.getdbt.com/docs/deploy/job-scheduler)). A slow job that a pipeline calls every hour can therefore build a growing queue. dbt also deactivates a job after 100 failed runs in a row. A successful call to start a run says nothing about the result; the caller must poll the run.

#### Version Gap {#version-gap}

> type: warning

A constraint such as `~> 0.3` with no committed lock file stays on 0.3.26, the last 0.x release. The Registry docs then describe a different schema. An upgrade to 1.x needs care: the first 1.x plans can show `connection_id` and `credential_id` changes ([issue 429](https://github.com/dbt-labs/terraform-provider-dbtcloud/issues/429); [issue 519](https://github.com/dbt-labs/terraform-provider-dbtcloud/issues/519)). Read the upgrade guide and the changelog, upgrade in one step to a recent 1.x version, and review the first plan line by line.

## Good and Bad Decisions {#dbtcloud-decisions}

> type: category

Each node compares a choice with its alternative.

### Who Schedules the Jobs {#who-schedules-the-jobs}

> type: concept

dbt Cloud can start jobs on a schedule, on a merge, or on a pull request. A team can instead turn all triggers off and start jobs from its own orchestrator through the API. The second way gives one scheduler for all systems and quiet plans. Its costs: the dbt Cloud UI shows no schedule, runs can queue (see Jobs Started by the API), and the orchestrator must poll each run for its result.

### Import What the UI Built {#import-what-the-ui-built}

> type: concept

Many dbt Cloud accounts start in the UI. `import` blocks adopt those objects without a destroy, and `moved` blocks keep them safe during refactors. dbt Labs offers the tool `dbtcloud-terraforming`, which writes both the configuration and the import blocks for an account ([import guide](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v1.13.1/docs/guides/3_importing_existing_configuration.md#L8-L10)).

### Key Pairs for Snowflake {#key-pairs-for-snowflake}

> type: concept

dbt Cloud no longer lets you create Snowflake credentials with a user name and password. New credentials default to key pairs, and dbt recommends PKCS#8 keys ([connect Snowflake](https://docs.getdbt.com/docs/platform/connect-data-platform/connect-snowflake)). In Terraform, pass the key from a secret file at run time, and use `private_key_wo` on Terraform 1.11 or later, so the key stays out of the state. The [authentication map](../../authentication/authentication.md#private-key-private-key) explains key-pair sign-in.

### Leave Some Objects Out {#leave-some-objects-out}

> type: concept

Not every dbt Cloud object belongs in Terraform. Two examples:

- Any change to a service token replaces it, because the API does not allow changes to an existing token. The new token has a new secret ([changelog](https://github.com/dbt-labs/terraform-provider-dbtcloud/blob/v0.3.26/CHANGELOG.md#L17)).
- `dbtcloud_user_groups` removes groups that people add by hand (see Authoritative Group Lists).

Objects that people change often, or whose replacement breaks callers, can stay in the UI. Write down which objects Terraform owns.

## About the Sources {#dbtcloud-about-the-sources}

> type: comment

The provider facts come from the `dbt-labs/dbtcloud` repository at tag v1.13.1, the latest release on 2026-11-23, and at tag v0.3.26, the last 0.x release. The dbt facts come from docs.getdbt.com on the same day. No dbt Cloud API was called to write this map.
