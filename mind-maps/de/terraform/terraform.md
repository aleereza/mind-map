# Terraform {#terraform}

> type: category

Terraform is an infrastructure as code tool from HashiCorp ([What is Terraform?](https://developer.hashicorp.com/terraform/intro)). Infrastructure as code (IaC) is a DevOps term. It means that you describe your infrastructure in text files and keep the files in Git. The files replace clicks in a web console and one-off SQL scripts. For a data engineer, infrastructure is the set of objects that the data lives in and runs on. Examples are a Snowflake database, warehouse, role or grant, an Aiven Kafka service, topic or ACL (access control list), and an S3 bucket for Iceberg files.

Terraform files are declarative. Declarative means that a file states the end result, not the steps. You write "a warehouse named TRANSFORM_DEV_WH with size XSMALL". Terraform finds the steps that get there: create the warehouse, change one setting, or delete it.

Four parts make this work. Each part uses the parts before it:

- Configuration: the `.tf` files that you write. They are made of blocks in HCL, the HashiCorp Configuration Language ([Configuration and HCL](#configuration-and-hcl-configuration-and-hcl)). A `resource` block declares one object that Terraform owns ([Resource](#resource-resource)).
- Provider: a plugin that turns resource blocks into API calls for one product. Terraform itself knows no product. The Snowflake provider turns a warehouse block into `CREATE WAREHOUSE` and `ALTER WAREHOUSE` statements ([Provider](#provider-provider)).
- State: a JSON file in which Terraform records which real object belongs to which resource block ([State](#state-state)). A backend is the place where the state file lives, for example an S3 bucket ([Backend](#backend-backend)).
- Plan: the list of creates, changes and deletes that makes the real objects match the configuration ([Plan](#plan-plan)).

The unit of work is one folder. Terraform reads all the `.tf` files in one folder as one configuration, which it calls a module. The folder where you run Terraform is the root module, and each root module has its own state ([Module](#module-module)). One repo can hold many root modules, for example one for each environment.

Each change goes through three commands. `terraform init` downloads the providers and connects to the backend. `terraform plan` reads the configuration, the variable values, the state and the real objects, and prints the plan ([terraform plan](#terraform-plan-terraform-plan)). A plan changes nothing. `terraform apply` makes the changes through the providers and writes the new state.

Now compare dbt, which is also declarative. In dbt, you write a `SELECT`, and dbt writes the `CREATE TABLE`. In Terraform, you write the warehouse that you want, and the provider writes the `CREATE WAREHOUSE`. The two tools manage different objects. Terraform plans changes only for the objects in its configuration and state. dbt creates the tables and views of its models each time it runs, so no Terraform state lists them ([Where Terraform Fits](#where-terraform-fits-where-terraform-fits)).

The map has three parts. Main Concepts defines the words. Reading a Terraform Repo says where to look and which file names Terraform treats in a special way ([File Names](#file-names-file-names)). Data Flow follows one change from the files to the real objects, and lists the data that goes into a plan.

## Main Concepts {#terraform-main-concepts}

> type: category

The concepts come in dependency order: each concept uses only the concepts before it.

### Where Terraform Fits {#where-terraform-fits}

> type: concept

Terraform can manage any product that has an API and a provider. The [Terraform Registry](https://registry.terraform.io/) publishes the providers. On 2026-11-16, the Registry listed these providers for the products in this stack:

| Product | Provider | Latest version | Example resource types |
| --- | --- | --- | --- |
| Snowflake | [`snowflakedb/snowflake`](https://registry.terraform.io/providers/snowflakedb/snowflake/latest/docs) | 2.23.0 | `snowflake_database`, `snowflake_schema`, `snowflake_warehouse`, `snowflake_account_role`, `snowflake_grant_privileges_to_account_role`, `snowflake_external_volume` |
| Aiven for Apache Kafka | [`aiven/aiven`](https://registry.terraform.io/providers/aiven/aiven/latest/docs) | 4.66.0 | `aiven_kafka`, `aiven_kafka_topic`, `aiven_kafka_user`, `aiven_kafka_acl`, `aiven_kafka_connector` |
| dbt Cloud | [`dbt-labs/dbtcloud`](https://registry.terraform.io/providers/dbt-labs/dbtcloud/latest/docs) | 1.12.13 | `dbtcloud_project`, `dbtcloud_environment`, `dbtcloud_job` |
| AWS, for the S3 bucket of Iceberg files | [`hashicorp/aws`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) | 6.73.0 | `aws_s3_bucket`, `aws_iam_role` |

Snowflake says that its provider manages "warehouses, databases, schemas, tables, roles, grants, and more" ([Snowflake Terraform provider](https://docs.snowflake.com/en/user-guide/terraform)).

Each object needs one owner. Terraform changes only the objects in its configuration and state. Other tools create other objects:

- dbt creates the tables and views of its models each time it runs. No Terraform state lists them. The [dbt map](../dbt/dbt-deep-mental-model.md) describes dbt.
- A Kafka producer writes events into a topic. Terraform can create the topic and its ACLs. The events are data, not infrastructure. The [Kafka map](../kafka/kafka.md) describes Kafka.
- An Iceberg sink writes Parquet and metadata files into a bucket. Terraform can create the bucket and the roles that may write to it. The [Iceberg map](../iceberg/iceberg.md) describes Iceberg.

### Configuration and HCL {#configuration-and-hcl}

> type: concept

Terraform code is called configuration. You write it in HCL, the HashiCorp Configuration Language, in files with the extension `.tf` ([syntax](https://developer.hashicorp.com/terraform/language/syntax/configuration)). A configuration is a list of blocks. A block has a type, zero or more labels, and a body of arguments in braces:

```hcl
resource "snowflake_warehouse" "transform" {
  name           = "TRANSFORM_${local.suffix}_WH"
  warehouse_size = "XSMALL"
  auto_suspend   = 60
}
```

Here `resource` is the block type. `snowflake_warehouse` is the resource type, which the Snowflake provider defines. `transform` is a local name that only Terraform uses. `name`, `warehouse_size` and `auto_suspend` are arguments. `${...}` inserts the value of an expression into a string.

A configuration uses these block types:

- `terraform`: the Terraform version, the providers and their versions, and the backend.
- `provider`: the connection settings of one provider.
- `resource`: an object that Terraform creates, changes and deletes.
- `data`: an object that Terraform only reads.
- `variable`: an input value.
- `locals`: named expressions inside one module.
- `output`: a value that the module returns.
- `module`: a call to another module.
- `moved` and `import`: instructions that change what the state records.

### Provider {#provider}

> type: concept

A provider is a plugin: a separate program that Terraform downloads and starts. The provider knows the API of one product. Terraform decides what to do and in which order, and the provider makes the API calls. For Snowflake, the provider builds SQL statements such as `CREATE WAREHOUSE` ([provider source](https://github.com/snowflakedb/terraform-provider-snowflake/blob/v2.23.0/pkg/sdk/warehouses_gen.go#L51-L55)).

Each provider has a source address of the form `[<hostname>/]<namespace>/<type>`. The hostname defaults to `registry.terraform.io`, so `snowflakedb/snowflake` means `registry.terraform.io/snowflakedb/snowflake`. The namespace is the publisher, and the type is the product ([source addresses](https://developer.hashicorp.com/terraform/language/providers/requirements#source-addresses)).

Two blocks set up a provider. The `terraform` block says which provider and which versions to use. The `provider` block says how to connect. These blocks come from About the Examples:

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    snowflake = {
      source  = "snowflakedb/snowflake"
      version = "2.23.0"
    }
  }
}

provider "snowflake" {
  organization_name = var.snowflake_organization
  account_name      = var.snowflake_account
  user              = var.snowflake_user
  role              = var.snowflake_role
  authenticator     = "SNOWFLAKE_JWT"
  private_key       = file(var.private_key_path)
}
```

Terraform acts as this user with this role. The role's privileges therefore limit what an apply can do in Snowflake. `SNOWFLAKE_JWT` selects key-pair authentication, which the [authentication map](../authentication/authentication.md) explains. The credentials come from input variables, so the `.tf` files hold no secret.

### Resource {#resource}

> type: concept

A resource block declares one object that Terraform creates, changes and deletes. The resource type comes from a provider. This topic comes from the Aiven provider's example ([`aiven_kafka_topic`](https://registry.terraform.io/providers/aiven/aiven/latest/docs/resources/kafka_topic)):

```hcl
resource "aiven_kafka_topic" "orders" {
  project      = "my-project"
  service_name = "my-kafka"
  topic_name   = "orders"
  partitions   = 3
  replication  = 3
}
```

Terraform names each object by its address: the resource type, a dot, and the local name, here `aiven_kafka_topic.orders`. In a child module, the address starts with the module path, as in `module.kafka.aiven_kafka_topic.orders`. A block with `for_each` makes one object for each key, as in `aiven_kafka_topic.cdc["orders"]` ([resource addressing](https://developer.hashicorp.com/terraform/cli/state/resource-addressing)). Plan output, the state, and `moved` and `import` blocks all use addresses.

Another block reads an attribute through the address, for example `aiven_kafka_topic.orders.topic_name`. Terraform infers from such a reference that the topic must exist first. The `depends_on` argument is only for a dependency that no reference shows ([depends_on](https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on)).

### Data Source {#data-source}

> type: concept

A `data` block reads an object that Terraform does not manage. Terraform never changes that object. Terraform reads it during the plan, unless the block depends on a value that only the apply can know ([data sources](https://developer.hashicorp.com/terraform/language/data-sources)). This example comes from the Snowflake provider's docs ([`snowflake_warehouses`](https://registry.terraform.io/providers/snowflakedb/snowflake/latest/docs/data-sources/warehouses)):

```hcl
data "snowflake_warehouses" "like_prefix" {
  like = "prefix%"
}
```

Other blocks read the result as `data.snowflake_warehouses.like_prefix.warehouses`. A data source suits an object that another team, another tool or another state owns.

### Variables, Locals, and Outputs {#variables-locals-outputs}

> type: concept

These three blocks are the inputs, the internal names and the results of a module:

- A `variable` block declares an input. Code reads it as `var.<name>`.
- A `locals` block names an expression, so that you write it once. Code reads it as `local.<name>`.
- An `output` block returns a value. The root module prints its outputs after an apply and keeps them in the state. A parent module reads the output of a child as `module.<name>.<output>`.

These blocks come from About the Examples:

```hcl
variable "environment" {
  type        = string
  description = "Short name of the environment, such as dev or prod."
}

locals {
  suffix = upper(var.environment)
}

output "raw_database_name" {
  value = snowflake_database.raw.name
}
```

The root module gets variable values from several sources. When two sources set the same variable, the first source in this list wins ([assign values](https://developer.hashicorp.com/terraform/language/values/variables#assign-values-to-variables)):

1. `-var` and `-var-file` options on the command line, in the order given.
2. `*.auto.tfvars` and `*.auto.tfvars.json` files, in lexical order.
3. The file `terraform.tfvars.json`.
4. The file `terraform.tfvars`.
5. Environment variables named `TF_VAR_<name>`.
6. The `default` argument of the `variable` block.

A child module gets its values from the arguments of its `module` block. The argument `sensitive = true` hides a value in the plan output. Terraform still stores the value in the state ([sensitive values](https://developer.hashicorp.com/terraform/language/values/variables#sensitive-values-in-variables)).

### Module {#module}

> type: concept

A module is the set of `.tf` files in one folder. Terraform does not read subfolders: each subfolder is a separate module ([files and directories](https://developer.hashicorp.com/terraform/language/files)). The root module is the folder where you run Terraform. A child module is a module that another module calls with a `module` block:

```hcl
module "orders_topic" {
  source     = "./modules/kafka-topic"
  topic_name = "orders"
  partitions = 3
}
```

`source` says where the module is. It can be a local path, which must begin with `./` or `../`, a Terraform Registry address with a `version`, or a Git URL ([module sources](https://developer.hashicorp.com/terraform/language/modules/sources)). The other arguments fill the variables of the child module. The addresses of the child's resources start with `module.orders_topic.`.

A module works like a function. Its variables are the parameters, its resources are the body, and its outputs are the return values. The root module and all the modules that it calls form one configuration, and all their objects go into one state.

### State {#state}

> type: concept

State is Terraform's record of the objects that it manages. Its main job is to bind each resource address to one real object ([purpose of state](https://developer.hashicorp.com/terraform/language/state/purpose)). Without the state, Terraform cannot tell whether a warehouse in Snowflake belongs to `snowflake_warehouse.transform` or to someone else.

By default, the state is the file `terraform.tfstate` in the root module folder. Terraform keeps the previous version in `terraform.tfstate.backup` ([state](https://developer.hashicorp.com/terraform/language/state)). The file is JSON. This excerpt comes from the import in About the Examples. The real file has 4,445 bytes, and the warehouse has 23 attributes. The excerpt leaves out most attributes and some other fields:

```json
{
  "version": 4,
  "terraform_version": "1.15.8",
  "resources": [
    {
      "mode": "managed",
      "type": "snowflake_warehouse",
      "name": "agent",
      "provider": "provider[\"registry.terraform.io/snowflakedb/snowflake\"]",
      "instances": [
        {
          "attributes": {
            "auto_resume": "true",
            "auto_suspend": 60,
            "id": "\"AGENT_WH\"",
            "name": "AGENT_WH",
            "warehouse_size": "XSMALL"
          }
        }
      ]
    }
  ]
}
```

`type` and `name` together give the address `snowflake_warehouse.agent`. `id` is the key with which the provider finds the object again. For a Snowflake warehouse, the ID is the name in double quotes. Do not edit the file by hand. The commands `terraform state list`, `terraform state show`, `terraform state mv` and `terraform state rm` read and change it ([state commands](https://developer.hashicorp.com/terraform/cli/state)).

#### State Holds Secrets {#state-holds-secrets}

> type: warning

The state file and saved plan files hold attribute values in plain text. That includes passwords and keys that the configuration gives to a resource ([manage sensitive data](https://developer.hashicorp.com/terraform/language/manage-sensitive-data)). `sensitive = true` hides a value only in the output. Keep the state in a backend with access control, and never commit it to Git ([.gitignore advice](https://developer.hashicorp.com/terraform/language/style#gitignore)). A variable with `ephemeral = true` stays out of state and plan files, with some limits on where you can use it.

#### Import {#import}

> type: concept

> edge.related: outside-changes-get-reverted

Import brings an object that already exists under Terraform management. Terraform reads the object and writes it into the state. Import does not change the object ([import](https://developer.hashicorp.com/terraform/language/import)). A team uses import to adopt objects that someone created by hand.

The command form writes the state at once. In About the Examples, this command read an existing warehouse with a read-only role:

`terraform import snowflake_warehouse.agent '"AGENT_WH"'`

Each provider's docs give the ID format. For a Snowflake warehouse, it is the name in double quotes ([warehouse import](https://registry.terraform.io/providers/snowflakedb/snowflake/latest/docs/resources/warehouse#import)). An `import` block in a `.tf` file does the same job through a normal plan and apply, so a reviewer sees the import in the plan first ([import block](https://developer.hashicorp.com/terraform/language/block/import)). After an import, read the next plan line by line. Outside Changes Get Reverted shows the first plan after this import.

#### Moved Block {#moved-block}

> type: concept

By default, Terraform reads a changed address as a delete at the old address and a create at the new address ([refactoring](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring)). So a new local name, or a move into a module, would drop the object and create a new one. A `moved` block tells Terraform that the object now has a new address:

```hcl
moved {
  from = snowflake_warehouse.transform
  to   = module.warehouses.snowflake_warehouse.transform
}
```

The next plan then changes only the address in the state. The real object stays as it is.

### Backend {#backend}

> type: concept

A backend decides where the state lives. Without a `backend` block, the state is the local file `terraform.tfstate`. A team keeps the state in a shared, remote place, so that every run sees the same state. Terraform has backends for S3, Azure Storage (`azurerm`), Google Cloud Storage (`gcs`), PostgreSQL (`pg`), Kubernetes and others ([backends](https://developer.hashicorp.com/terraform/language/backend)). A `cloud` block in place of a `backend` block stores the state in HCP Terraform, the hosted service of HashiCorp ([terraform block](https://developer.hashicorp.com/terraform/language/block/terraform)). This is an S3 backend:

```hcl
terraform {
  backend "s3" {
    bucket       = "my-terraform-state"
    key          = "snowflake/terraform.tfstate"
    region       = "us-west-2"
    use_lockfile = true
  }
}
```

A backend block cannot use variables, locals or data sources. To change the backend for each environment, a repo leaves some arguments out of the block and gives them to `terraform init`, for example `terraform init -backend-config=prod.s3.tfbackend`. HashiCorp calls this a partial configuration and recommends the file name `*.<backend name>.tfbackend` ([partial configuration](https://developer.hashicorp.com/terraform/language/backend#partial-configuration)).

A lock stops two runs from writing the same state at the same time ([state locking](https://developer.hashicorp.com/terraform/language/state/locking)). With `use_lockfile = true`, the S3 backend writes a lock file next to the state, named `<key>.tflock`. Some repos set `dynamodb_table` in place of it, which locks through a DynamoDB table. The docs mark DynamoDB locking as deprecated ([S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)).

### Workspace {#workspace}

> type: concept

A CLI workspace is one more state for the same configuration and the same backend ([workspaces](https://developer.hashicorp.com/terraform/language/state/workspaces)). Every backend starts with the workspace `default`. The commands `terraform workspace new prod` and `terraform workspace select prod` choose the state that the next plan uses. Code reads the current name as `terraform.workspace`:

```hcl
resource "snowflake_warehouse" "transform" {
  name           = "TRANSFORM_${upper(terraform.workspace)}_WH"
  warehouse_size = terraform.workspace == "prod" ? "LARGE" : "XSMALL"
}
```

With the S3 backend, the state of the workspace `prod` lives at `env:/prod/<key>` in the bucket ([S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)). HashiCorp advises against CLI workspaces for environments that need separate credentials and access controls ([CLI workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)). HCP Terraform also uses the word workspace, for a different object.

### Plan {#plan}

> type: concept

> edge.related: terraform-plan

A plan is the list of actions that makes the real objects match the configuration. A plan changes no real object. Each action has a symbol ([plan output](https://developer.hashicorp.com/terraform/cli/commands/plan#understand-plan-output)):

| Symbol | Action | Meaning |
| --- | --- | --- |
| `+` | Create | The object does not exist. Terraform creates it. |
| `~` | Update in place | Terraform changes the object without deleting it. |
| `-` | Destroy | Terraform deletes the object. |
| `-/+` | Replace | Terraform deletes the object and then creates a new one. |

The plan in About the Examples creates a database and a warehouse in Snowflake. This is the warehouse part of its output and the summary:

```text
  # snowflake_warehouse.transform will be created
  + resource "snowflake_warehouse" "transform" {
      + auto_resume                         = "default"
      + auto_suspend                        = 60
      + enable_query_acceleration           = "default"
      + fully_qualified_name                = (known after apply)
      + id                                  = (known after apply)
      + max_concurrency_level               = (known after apply)
      + name                                = "TRANSFORM_DEV_WH"
      + parameters                          = (known after apply)
      + query_acceleration_max_scale_factor = -1
      + show_output                         = (known after apply)
      + statement_queued_timeout_in_seconds = (known after apply)
      + statement_timeout_in_seconds        = (known after apply)
      + warehouse_size                      = "XSMALL"
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

`(known after apply)` marks a value that only the API can give, such as the ID. The values from the configuration, such as the name, appear in full. `"default"` and `-1` are special values of the Snowflake provider. They mean "use the Snowflake default" ([warehouse resource](https://registry.terraform.io/providers/snowflakedb/snowflake/latest/docs/resources/warehouse)).

`terraform plan` alone makes a speculative plan, which is a preview. `terraform plan -out=dev.tfplan` saves the plan in a file, and `terraform apply dev.tfplan` then makes exactly those changes ([plan command](https://developer.hashicorp.com/terraform/cli/commands/plan)). The saved file in About the Examples is a zip archive of 4,375 bytes. It holds the plan, two state snapshots, a copy of each `.tf` file and the lock file.

#### Replacement Destroys Data {#replacement-destroys-data}

> type: warning

Some arguments cannot change on a live object. A change to such an argument makes a replace (`-/+`), and the plan marks the argument with `# forces replacement`. For an `aiven_kafka_topic`, a change to `topic_name`, `project` or `service_name` forces a new topic ([Aiven topic docs](https://registry.terraform.io/providers/aiven/aiven/latest/docs/resources/kafka_topic)). A new topic name in the configuration therefore deletes the topic with all its messages, and creates an empty topic.

`lifecycle { prevent_destroy = true }` in a resource block makes Terraform reject every plan that destroys the object. It does not protect the object when someone deletes the whole block, because Terraform then no longer reads the setting ([lifecycle](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)).

#### Outside Changes Get Reverted {#outside-changes-get-reverted}

> type: warning

Terraform makes each argument that it manages match the configuration. A plan therefore reverts a change that someone made outside Terraform, for example in Snowsight. This is the first plan after the import in About the Examples. The real warehouse had `AUTO_SUSPEND = 60`, and the configuration said 300. The `show_output` lines are left out:

```text
snowflake_warehouse.agent: Refreshing state... [id="AGENT_WH"]

  # snowflake_warehouse.agent will be updated in-place
  ~ resource "snowflake_warehouse" "agent" {
      ~ auto_resume                         = "true" -> "default"
      ~ auto_suspend                        = 60 -> 300
      ~ enable_query_acceleration           = "true" -> "default"
      - generation                          = "2" -> null
        id                                  = "\"AGENT_WH\""
      - max_cluster_count                   = 1 -> null
      - min_cluster_count                   = 1 -> null
        name                                = "AGENT_WH"
      ~ query_acceleration_max_scale_factor = 8 -> -1
      - scaling_policy                      = "STANDARD" -> null
      - warehouse_type                      = "STANDARD" -> null
        # (8 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Only the `auto_suspend` line comes from the configuration. The other eight lines are arguments that the configuration does not set:

- The three `~` lines get the provider's value `"default"` or `-1`, which means the Snowflake default. For this warehouse, the Snowflake defaults are the current values: `AUTO_RESUME` is `TRUE`, `ENABLE_QUERY_ACCELERATION` is `TRUE` for a standard warehouse, and `QUERY_ACCELERATION_MAX_SCALE_FACTOR` is 8 ([CREATE WAREHOUSE](https://docs.snowflake.com/en/sql-reference/sql/create-warehouse)). A warehouse with a value other than the default would lose that value.
- The five `-` lines set an argument to null. For `warehouse_type`, the provider docs say that the provider then sets the type to `STANDARD`. For the other four, the docs do not say what an apply sends to Snowflake ([warehouse resource](https://registry.terraform.io/providers/snowflakedb/snowflake/latest/docs/resources/warehouse)).

Write each setting that matters into the configuration, so that the plan shows no line for it. To keep a change made outside Terraform, copy the value into the configuration. To make Terraform leave one argument alone, add `lifecycle { ignore_changes = [auto_suspend] }` ([lifecycle](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)).

## Reading a Terraform Repo {#reading-a-terraform-repo}

> type: category

A repo can hold many configurations. These nodes say which file names matter, how to find each configuration, how repos are laid out, and in which order to read.

### File Names {#file-names}

> type: concept

Terraform reads every `.tf` and `.tf.json` file in a folder as one document. The split into files has no effect on what Terraform does ([files and directories](https://developer.hashicorp.com/terraform/language/files)). So a resource in `main.tf` works the same in `warehouses.tf`. Only these names have a meaning for Terraform:

| Name | What Terraform does with it | In Git |
| --- | --- | --- |
| `*.tf`, `*.tf.json` | Reads all of them in the folder as configuration. | Yes |
| `override.tf`, `*_override.tf` | Reads them last, and merges each block into the block of the same name ([override files](https://developer.hashicorp.com/terraform/language/files/override)). | Yes |
| `terraform.tfvars`, `terraform.tfvars.json` | Loads the variable values automatically. | Yes, without secrets |
| `*.auto.tfvars`, `*.auto.tfvars.json` | Loads the variable values automatically, in lexical order. | Yes, without secrets |
| Any other `*.tfvars` | Loads the values only with `-var-file=<file>`. | Yes, without secrets |
| `.terraform.lock.hcl` | Records the provider versions and checksums that `init` chose ([lock file](https://developer.hashicorp.com/terraform/language/files/dependency-lock)). | Yes |
| `.terraform/` | Holds the downloaded providers and modules. | No |
| `terraform.tfstate`, `terraform.tfstate.backup` | Local state. | No |
| `.terraform.tfstate.lock.info` | Local state lock. | No |
| The file after `plan -out=` | Saved plan. | No |
| `*.tftest.hcl` | Tests for `terraform test` ([test files](https://developer.hashicorp.com/terraform/language/files/tests)). | Yes |

The "In Git" column follows the [style guide](https://developer.hashicorp.com/terraform/language/style#gitignore).

Other names are a convention for readers. The style guide recommends these names ([file names](https://developer.hashicorp.com/terraform/language/style#file-names)):

- `terraform.tf`: the `terraform` block with `required_version` and `required_providers`. The public [S3 bucket module](https://github.com/terraform-aws-modules/terraform-aws-s3-bucket) names this file `versions.tf`.
- `backend.tf`: the backend configuration.
- `providers.tf`: the `provider` blocks.
- `variables.tf`, `outputs.tf`, `locals.tf`: the `variable`, `output` and `locals` blocks.
- `main.tf`: the resource and data blocks.

A configuration can split `main.tf` by subject, for example into `warehouses.tf`, `topics.tf` or `grants.tf`. Files such as `moved.tf` or `imports.tf` collect `moved` and `import` blocks. Terraform treats all of them as ordinary `.tf` files.

### Root Module Folders {#root-module-folders}

> type: concept

> edge.related: module

Each folder in which someone runs Terraform is a root module with its own state. Two signs mark such a folder:

- A `backend` or `cloud` block, which says where its state lives.
- A committed `.terraform.lock.hcl`. Terraform writes the lock file in the folder where `init` runs, and the style guide says to commit it. Some repos do not commit it, so the backend block is the more reliable sign.

These commands list the candidates from the repo root:

```bash
find . -name .terraform.lock.hcl -not -path '*/.terraform/*'
grep -rl --include='*.tf' -E '^\s*(backend "|cloud \{)' .
```

A folder that other folders only call through `module` blocks is a child module. It needs neither sign, because it uses the backend and the lock file of the root module that calls it. The standard module structure puts such modules under `modules/` ([standard module structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure)).

### Repo Layouts {#repo-layouts}

> type: concept

The style guide shows three ways to separate environments and parts of the system ([multiple environments](https://developer.hashicorp.com/terraform/language/style#multiple-environments)):

- One folder for each environment, such as `dev/`, `staging/` and `prod/`. Each folder has its own `backend.tf`, so each environment has its own state. All of them call the shared modules in `modules/`.
- One folder for each part of the system, such as `networking/`, `database/` and `compute/`. Each part has its own state, so a change in one part cannot touch the objects of another part.
- One configuration with one workspace for each environment. The Workspace node describes the limits of CLI workspaces.

Values that differ by environment can sit in files such as `prod.tfvars`, which a pipeline passes with `-var-file`. The style guide also compares one repo for all configurations, a monorepo, with one repo for each module ([repository structure](https://developer.hashicorp.com/terraform/language/style#repository-structure)).

### Reading Order {#reading-order}

> type: concept

This order answers the main questions about a configuration with the least reading:

1. Find the root modules. Root Module Folders shows how.
2. In each root module, read the `terraform` block. It names the providers, their versions and the backend, so you learn which products the folder changes and where the state lives.
3. Read the `provider` blocks. They name the account, the user and the role that Terraform acts as, and where the credentials come from.
4. Read `variables.tf` and the `*.tfvars` files. They show what differs between environments.
5. List the resource types. This command counts the blocks in a folder:

   `grep -rhoE '^(resource|data|module) "[^"]+"' --include='*.tf' . | sort | uniq -c`

   In the example of About the Examples, it prints `1 resource "snowflake_database"` and `1 resource "snowflake_warehouse"`.
6. Follow each `module` block to its `source` folder, and read that module in the same order.
7. Read the CI pipeline file, if the repo has one. It shows which folder and which workspace each job uses, and which step needs an approval before apply.

## Data Flow {#terraform-data-flow}

> type: category

One change flows like this:

`.tf` files and variable values → `terraform init` → `terraform plan`, which also reads the state and the real objects → the plan → `terraform apply` → provider API calls → changed real objects and a new state

### terraform init {#terraform-init}

> type: concept

#### Inputs

- The `terraform` block: `required_providers` and the backend.
- The `source` of each `module` block.
- The lock file `.terraform.lock.hcl`, if it exists.
- `-backend-config` values for a partial backend configuration.

#### Outputs

- The `.terraform/` folder with the providers and the remote modules. In About the Examples, init put the Snowflake provider in `.terraform/providers/registry.terraform.io/snowflakedb/snowflake/2.23.0/darwin_arm64`, and the folder had 52 MB.
- The lock file. In the example, it held one entry, for `registry.terraform.io/snowflakedb/snowflake`, with `version = "2.23.0"` and 13 checksums.
- A connection to the backend. Init does not change the state.

#### Under the hood

For each provider, init uses the version in the lock file. Without a lock entry, it takes the newest version that matches the `version` constraint and records it. Init checks each download against the checksums in the lock file. `terraform init -upgrade` takes newer versions within the constraints ([lock file](https://developer.hashicorp.com/terraform/language/files/dependency-lock)). Init is safe to run many times ([init command](https://developer.hashicorp.com/terraform/cli/commands/init)).

### terraform plan {#terraform-plan}

> type: concept

#### Inputs

This list is all the data that goes into Terraform:

- The configuration: every `.tf` file of the root module and of the modules that it calls.
- The variable values, from the sources that Variables, Locals, and Outputs lists.
- The prior state, from the backend.
- The real objects. The provider reads each object in the state again. This step is the refresh. In About the Examples, the plan printed `snowflake_warehouse.agent: Refreshing state... [id="AGENT_WH"]`.
- The results of the data sources, which the providers read.
- The credentials of each provider and of the backend, from `provider` blocks, variables or environment variables.

#### Outputs

- The plan on the screen. With `-out`, also a saved plan file.
- No change to any real object. The state file does not change either: in About the Examples, its SHA-1 checksum was the same before and after a plan.

#### Under the hood

The plan command does three things ([plan command](https://developer.hashicorp.com/terraform/cli/commands/plan)). First, it reads the current state of the real objects. Next, it compares the configuration with that state. Then, it proposes the actions that remove the differences. Terraform orders the actions by a dependency graph, which it builds from the references between blocks. Objects that do not depend on each other are handled in parallel, 10 at a time by default. In About the Examples, a read-only Snowflake role was enough for every plan.

### terraform apply {#terraform-apply}

> type: concept

#### Inputs

- A saved plan file. Terraform then makes exactly the planned changes, without a prompt.
- Or no file. Terraform then makes a new plan, shows it, and waits for you to type `yes`. `-auto-approve` skips that question ([apply command](https://developer.hashicorp.com/terraform/cli/commands/apply)).
- Credentials with the privileges to make the changes.

#### Outputs

- Changed real objects, through the API calls of the providers. For Snowflake, the calls are SQL statements.
- A new state in the backend.
- The values of the root module outputs, on the screen.

#### Under the hood

Apply holds the state lock while it runs. When an API call fails, Terraform reports the error, saves the state with the changes that it already made, unlocks the state and stops. It does not roll back the finished changes. After you fix the cause, you run apply again ([errors during apply](https://developer.hashicorp.com/terraform/cli/commands/apply#errors-during-an-apply-operation)). With HCP Terraform, a speculative plan runs for each pull request, and an apply starts after the merge ([speculative plans on pull requests](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/run/ui#speculative-plans-on-pull-requests)).

## About the Examples {#about-the-examples}

> type: comment

All outputs in this map come from runs on 2026-11-16, with Terraform 1.15.8 on macOS (darwin_arm64) and the provider `snowflakedb/snowflake` 2.23.0. The newest Terraform release on that day was 1.17.2. No `terraform apply` ran, so nothing changed in Snowflake.

The first example is a folder with six files:

```hcl
# terraform.tf
terraform {
  required_version = ">= 1.5"

  required_providers {
    snowflake = {
      source  = "snowflakedb/snowflake"
      version = "2.23.0"
    }
  }
}

# providers.tf
provider "snowflake" {
  organization_name = var.snowflake_organization
  account_name      = var.snowflake_account
  user              = var.snowflake_user
  role              = var.snowflake_role
  authenticator     = "SNOWFLAKE_JWT"
  private_key       = file(var.private_key_path)
}

# variables.tf
variable "environment" {
  type        = string
  description = "Short name of the environment, such as dev or prod."
}

variable "private_key_path" {
  type = string
}

variable "snowflake_account" {
  type = string
}

variable "snowflake_organization" {
  type = string
}

variable "snowflake_role" {
  type = string
}

variable "snowflake_user" {
  type = string
}

# main.tf
locals {
  suffix = upper(var.environment)
}

resource "snowflake_database" "raw" {
  name    = "RAW_${local.suffix}"
  comment = "Landing zone for Kafka and Iceberg data."
}

resource "snowflake_warehouse" "transform" {
  name           = "TRANSFORM_${local.suffix}_WH"
  warehouse_size = "XSMALL"
  auto_suspend   = 60
}

# outputs.tf
output "raw_database_name" {
  value = snowflake_database.raw.name
}

# dev.tfvars
environment = "dev"
```

These commands ran in that folder. The role was a read-only role. The `TF_VAR_` variables carry the account and the key, so no file holds them:

```bash
export TF_VAR_snowflake_organization=<org> TF_VAR_snowflake_account=<account>
export TF_VAR_snowflake_user=<user> TF_VAR_snowflake_role=<read-only role>
export TF_VAR_private_key_path=<path to the .p8 key>
terraform init -input=false
terraform plan -input=false -var-file=dev.tfvars -out=dev.tfplan
unzip -l dev.tfplan
```

The second example is a copy of `terraform.tf`, `providers.tf` and `variables.tf` without the variable `environment`, and this `main.tf`:

```hcl
resource "snowflake_warehouse" "agent" {
  name           = "AGENT_WH"
  warehouse_size = "XSMALL"
  auto_suspend   = 300
  comment        = "Shared warehouse for agent read-only queries"
}
```

`AGENT_WH` is an existing X-Small warehouse with `AUTO_SUSPEND = 60`. These commands ran with the same environment variables:

```bash
terraform init -input=false
terraform import snowflake_warehouse.agent '"AGENT_WH"'
shasum terraform.tfstate
terraform plan -input=false
shasum terraform.tfstate
```
