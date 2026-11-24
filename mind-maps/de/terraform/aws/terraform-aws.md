# Terraform for AWS {#terraform-aws}

> type: category

The AWS provider lets Terraform manage resources in Amazon Web Services through the AWS APIs. HashiCorp maintains it under the source address `hashicorp/aws` ([provider docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)). A second provider, `hashicorp/awscc`, reaches AWS through the Cloud Control API instead ([awscc docs](https://registry.terraform.io/providers/hashicorp/awscc/latest/docs)). The [Terraform map](../terraform.md) explains Terraform itself.

The unit is one AWS account and one region. Each provider block reaches one account through a chain of identities, and it works in one region by default ([The Identity Chain](#the-identity-chain-the-identity-chain)). AWS has hundreds of services. A data engineer meets a few of them:

- S3 (Simple Storage Service) buckets hold data files, such as Iceberg tables and files for Snowpipe.
- IAM roles and policies decide who may read and write those files. IAM is AWS Identity and Access Management.
- Secrets Manager holds passwords and keys for programs.
- Queues in SQS (Simple Queue Service), topics in SNS (Simple Notification Service) and Lambda functions react to new files.
- Services in ECS (Elastic Container Service) and databases in RDS (Relational Database Service) run the apps that produce or serve data.

Four ideas explain most AWS plans:

- Many AWS settings are one document, such as the notification settings of a bucket. Each document can have only one owner in Terraform ([One Document, One Owner](#one-document-one-owner-one-document-one-owner)).
- Some resources add one item to a set, and others own the whole set ([Adders and Owners](#adders-and-owners-adders-and-owners)).
- Access between AWS and another system needs trust on both sides ([Trust Has Two Sides](#trust-has-two-sides-trust-has-two-sides)).
- Every value that Terraform reads or writes, secrets included, sits in the state ([State Holds Secret Values](#state-holds-secret-values-state-holds-secret-values)).

Deletes also differ by service, and each provider version changes some rules ([Deletes Differ by Service](#deletes-differ-by-service-deletes-differ-by-service); [Provider Generations](#provider-generations-provider-generations)).

Now compare Snowflake. In Snowflake, the role that creates an object owns it, and grants decide access. In AWS, the account owns every resource, and IAM policies decide access. A Snowflake grant connects a role to an object; an IAM policy connects an identity to actions on resources.

The map has four parts. Mental Models explains the ideas above. Reading an AWS Configuration says what you will see and where to look first. Pitfalls lists the failures that experienced users plan around. Good and Bad Decisions compares the main design choices.

## Mental Models {#aws-mental-models}

> type: category

The models come in dependency order: each model uses only the models before it.

### The Identity Chain {#the-identity-chain}

> type: concept

Terraform starts with some credentials, for example the keys of a pipeline. The provider can then assume an IAM role in the target account with `assume_role`, and it can assume several roles in a row ([role chaining](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/index.html.markdown#L77)). The final role decides which account the plan touches and what it may do. So read the `role_arn` expression of every provider block. When it depends on the workspace, the workspace picks the account.

Two settings help. `allowed_account_ids` stops a run in the wrong account. `default_tags` adds the same tags to every resource of the provider ([provider arguments](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/index.html.markdown#L372-L380)). The backend that stores the state has its own chain, set in the `backend "s3"` block.

A configuration can have several provider blocks with aliases, for example one for a DNS account and one for the region `us-east-1`. A resource then picks its block with `provider = aws.<alias>`.

### One Document, One Owner {#one-document-one-owner}

> type: concept

> edge.related: one-notification-configuration, inline-rules-win

Several AWS settings are a single document that the API replaces as a whole:

- The notification configuration of an S3 bucket ([notification note](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/s3_bucket_notification.html.markdown#L13)).
- The lifecycle configuration and the policy of a bucket.
- The access policy of an SQS queue or an SNS topic.
- The inline rules of a security group, which is a firewall for network traffic.

If two resources, or a resource and a person, write the same document, each write removes the other's part. Plans then show the same change again and again. Give each document exactly one owner in Terraform.

Since provider version 4, an S3 bucket has one resource for each document: `aws_s3_bucket_versioning`, `aws_s3_bucket_lifecycle_configuration`, `aws_s3_bucket_policy`, `aws_s3_bucket_notification` and others ([version 4 upgrade](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/guides/version-4-upgrade.html.markdown#L1324-L1335)).

### Adders and Owners {#adders-and-owners}

> type: concept

IAM attaches policies to roles. Two styles of resource do this:

- Adders add one item and leave the rest alone. Examples are `aws_iam_role_policy_attachment` and `aws_iam_role_policy`.
- Owners own the whole set and remove items that they do not list. Examples are the `inline_policy` and `managed_policy_arns` arguments of `aws_iam_role` ([role note](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/iam_role.html.markdown#L15)), and the `*_exclusive` resources.

One owner is a trap. `aws_iam_policy_attachment` owns every attachment of one policy across the whole account ([warning](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/iam_policy_attachment.html.markdown#L13)). If another configuration attaches the same policy elsewhere, the next apply detaches it there. Use `aws_iam_role_policy_attachment` instead.

### Trust Has Two Sides {#trust-has-two-sides}

> type: concept

> edge.related: deleted-principals-break-trust

An IAM role has a trust policy. It names who may assume the role, and it can demand an external ID, a shared value that the caller must present ([external ID](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios_third-party.html)). Another system that uses the role, such as Snowflake or a Kafka Connect connector, must know the role's ARN (Amazon Resource Name, the unique ID of an AWS resource) and the external ID. So each side needs values from the other side.

When the other system generates its values, someone must copy them into the trust policy, for example in a second apply. When you choose the external ID yourself and fix the role name in advance, both sides know every value before the first apply. AWS does not treat the external ID as a secret. The [Snowflake map](../snowflake/terraform-snowflake.md#two-step-trust-with-aws-two-step-trust-with-aws) shows the three ways that configurations solve this loop for Snowflake.

### State Holds Secret Values {#state-holds-secret-values}

> type: concept

> edge.related: secrets-in-state

The state stores every attribute that Terraform reads or writes, in plain text:

- `aws_iam_access_key` stores the secret key of an IAM user ([access key doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/iam_access_key.html.markdown#L77)).
- `aws_secretsmanager_secret_version` stores the secret value.
- `aws_db_instance` stores the master password.
- A data source that reads a secret stores it too.

Write-only arguments, such as `secret_string_wo` and `password_wo`, keep a value out of the state. They need Terraform 1.11 or later ([secret version doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/secretsmanager_secret_version.html.markdown#L15)). A program can also read a secret at run time by its ARN, so the value never passes through Terraform.

### Deletes Differ by Service {#deletes-differ-by-service}

> type: concept

A destroy in the plan means different things for different services:

| Resource | What a destroy does |
| --- | --- |
| `aws_s3_bucket` | Fails if the bucket holds objects, unless `force_destroy` is true; then it deletes all objects and versions first ([bucket doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/s3_bucket.html.markdown#L58)). |
| `aws_secretsmanager_secret` | Schedules the secret for deletion after a recovery window, 30 days by default ([secret doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/secretsmanager_secret.html.markdown#L44)). |
| `aws_db_instance` | Fails while `deletion_protection` is on; otherwise it takes a final snapshot unless `skip_final_snapshot` is true ([db_instance doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/db_instance.html.markdown#L332)). |
| `aws_kms_key` | Schedules the key for deletion after a waiting period of 7 to 30 days ([key doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/kms_key.html.markdown#L316)). |
| `aws_sns_topic_policy` | Resets the policy to the default instead of deleting it ([provider code](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/internal/service/sns/topic_policy.go#L108-L122)). |

So `-` in a plan can mean "gone now", "gone in 30 days" or "reset".

### Provider Generations {#provider-generations}

> type: concept

> edge.related: version-bugs

Each major version of the provider changed how code looks:

| Version | Released | Main change |
| --- | --- | --- |
| 4.0.0 | 2022-02-10 | S3 bucket settings moved into separate resources. |
| 5.0.0 | 2023-05-25 | Tag handling changed, and old arguments were removed. |
| 6.0.0 | 2025-06-18 | Each resource can set its own `region`. |

The upgrade guides list every change ([version 6 guide](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/guides/version-6-upgrade.html.markdown)). HashiCorp advises moving to the latest minor version first, getting a clean plan, and only then raising the major version ([upgrade steps](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/guides/version-6-upgrade.html.markdown#L109-L116)).

`awscc` is generated from AWS CloudFormation schemas. Every optional attribute in it is also computed, so it cannot detect drift in values that you did not set, and it ignores `default_tags` ([awscc docs](https://github.com/hashicorp/terraform-provider-awscc/blob/v1.110.0/docs/index.md#L20-L26)). HashiCorp built it to bring new AWS resources to Terraform faster, and expects it to run next to `hashicorp/aws` ([using both providers](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/guides/using-aws-with-awscc-provider.html.markdown)).

## Reading an AWS Configuration {#reading-aws-configuration}

> type: category

These nodes say what you will find in the files, in which order to read them, and what each kind of plan line means.

### What You Will See {#aws-what-you-will-see}

> type: concept

| Area | Resources | What to check |
| --- | --- | --- |
| Data buckets | `aws_s3_bucket`, `aws_s3_bucket_versioning`, `aws_s3_bucket_lifecycle_configuration`, `aws_s3_bucket_policy`, `aws_s3_bucket_notification`, `aws_s3_bucket_public_access_block` | `force_destroy`, versioning, expiry rules, one notification resource per bucket |
| Access | `aws_iam_role`, `aws_iam_policy`, `aws_iam_role_policy_attachment`, `data.aws_iam_policy_document` | Trust principals and external IDs, adders or owners |
| Users with keys | `aws_iam_user`, `aws_iam_access_key` | Where the secret key goes |
| Secrets | `aws_secretsmanager_secret`, `aws_secretsmanager_secret_version` | Value source, recovery window, `prevent_destroy` |
| Events | `aws_sns_topic`, `aws_sqs_queue`, `aws_sqs_queue_policy`, `aws_lambda_function`, `aws_lambda_event_source_mapping` | Queue policies, dead-letter queues, timeouts |
| Apps | `aws_ecs_service`, `aws_ecs_task_definition` | Image tags, `ignore_changes` |
| Databases | `aws_db_instance` | `deletion_protection`, `engine_version`, password source |
| Network | `aws_security_group`, `aws_vpc_security_group_ingress_rule` | Inline rules or separate rule resources |

`data.aws_iam_policy_document` builds policy JSON from HCL blocks. It gives stable JSON, so whitespace or key order cannot cause diffs.

### Where to Look First {#aws-where-to-look-first}

> type: concept

Read an AWS configuration in this order:

1. The `terraform` block: the backend, its key and its lock setting, and the version of `hashicorp/aws`. A committed `.terraform.lock.hcl` tells you the exact build.
2. Every `provider "aws"` block: the region, the `assume_role` role ARN, aliases, `allowed_account_ids` and `default_tags`. Together they tell you where each resource lands.
3. The inputs per environment: workspaces, variable files or YAML files. The same code can run in several accounts.
4. IAM: the trust policies, the external IDs, the attachment style, and every IAM user with an access key.
5. The data buckets: `force_destroy`, versioning, expiry, notifications and policies. Data loss and event wiring live here.
6. The secrets: where each value comes from, the recovery window and `prevent_destroy`.
7. The `lifecycle` blocks, and the `moved`, `removed` and `import` blocks. They show what Terraform will not touch and what moved.
8. The event paths: queue and topic policies, dead-letter queues, and Lambda timeouts.
9. The lookups of other states, such as data sources by name or tag, and SSM parameters. They are the contracts between configurations.

### Plan Signals {#aws-plan-signals}

> type: concept

| Line in the plan | Meaning |
| --- | --- |
| `-/+ aws_s3_bucket` | A new bucket name; S3 cannot rename a bucket, so the data stays in the old one or is lost with it. |
| The same `~` on a notification, policy or security group in every plan | Two owners write one document. |
| `-/+ aws_iam_policy` after a description change | A new description forces a new policy ([policy doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/iam_policy.html.markdown#L45)). |
| `~ engine_version` back to an older value on `aws_db_instance` | AWS upgraded the minor version; AWS will reject a downgrade. |
| `~ tags_all` with no tag change | Tags from `default_tags` differ from the live tags. |
| `- aws_secretsmanager_secret` | The secret is scheduled for deletion; its name stays taken during the recovery window. |

## Pitfalls {#aws-pitfalls}

> type: category

The pitfalls fall into four groups: data loss, event wiring, identity and secrets, and provider versions.

### Data Loss Pitfalls {#aws-data-loss-pitfalls}

> type: concept

These pitfalls delete or freeze data.

#### Force Destroy Empties the Bucket {#force-destroy-empties-bucket}

> type: warning

With `force_destroy = true`, a destroy of the bucket first deletes every object and every version, locked objects included ([bucket doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/s3_bucket.html.markdown#L58)). A rename, a moved resource without a `moved` block, or a deleted module call can plan that destroy. The setting takes effect only after an apply has stored it in the state. Keep it false on buckets with data, and add `prevent_destroy`.

#### Versioning Cannot Be Turned Off {#versioning-cannot-be-turned-off}

> type: warning

`Disabled` is valid only when a bucket is created or imported. A change from `Enabled` or `Suspended` to `Disabled` fails, because S3 cannot return a bucket to the unversioned state ([versioning doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/s3_bucket_versioning.html.markdown#L103-L104)). Use `Suspended` instead. After you first enable versioning, AWS advises waiting 15 minutes before writes ([versioning doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/s3_bucket_versioning.html.markdown#L17)).

#### Database Versions and Replacements {#database-versions-and-replacements}

> type: warning

With automatic minor upgrades on, AWS raises the engine version of an RDS instance by itself. A configuration that pins `engine_version = "16.4"` then plans a downgrade, and AWS rejects it. The provider docs advise a version prefix, such as `"16"`, in that case ([db_instance doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/db_instance.html.markdown#L342)). Other changes replace the instance: for example `engine`, `username`, `db_name` or `storage_encrypted`. Keep `deletion_protection` on and a final snapshot set.

### Event Wiring Pitfalls {#aws-event-wiring-pitfalls}

> type: concept

These pitfalls break the path from a new file to the code that handles it. Files keep arriving, but nothing reacts.

#### One Notification Configuration per Bucket {#one-notification-configuration}

> type: warning

`aws_s3_bucket_notification` writes the whole notification configuration of a bucket, so a bucket can have only one such resource ([notification note](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/s3_bucket_notification.html.markdown#L13)). A second resource, a console edit, or a Snowflake pipe with auto-ingest overwrites the others. S3 also rejects two rules with overlapping prefixes or suffixes for the same event type ([event filters](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-filtering.html)). To send one event to several readers, send it to one SNS topic or to EventBridge, and fan out from there.

#### Queue Timeouts and Policies {#queue-timeouts-and-policies}

> type: warning

When a Lambda function reads an SQS queue, AWS advises a visibility timeout of at least six times the function timeout ([Lambda with SQS](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)). For example, a function with a 60-second timeout needs a queue visibility timeout of at least 360 seconds. With 300 seconds, a slow batch can reach a second reader before the first one finishes. Two more traps:

- Creating or updating a queue with a queue policy hangs unless the policy sets `Version = "2012-10-17"` ([queue note](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/sqs_queue.html.markdown#L13)).
- A queue policy for SNS or S3 should name the sender with the condition `aws:SourceArn`, so no other topic or bucket can write to the queue ([confused deputy](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)).

#### Lambda Zips Built at Plan Time {#lambda-zips-built-at-plan-time}

> type: warning

The data source `archive_file` builds the zip file of a Lambda function during the plan ([archive provider](https://github.com/hashicorp/terraform-provider-archive/blob/v2.8.1/docs/data-sources/file.md#L10)). The apply needs the same file. When plan and apply run on different machines, the apply cannot find it. `source_code_hash` detects only the code changes that pass through Terraform. The newer argument `code_sha256` also catches code changed outside Terraform ([function doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/lambda_function.html.markdown#L715)).

### Identity and Secret Pitfalls {#aws-identity-pitfalls}

> type: concept

These pitfalls break access or leak secrets. Plans show them only indirectly.

#### Deleted Principals Break Trust {#deleted-principals-break-trust}

> type: warning

When IAM deletes a user or role that a trust policy names, it replaces the ARN in the policy with an internal ID. A new role with the same name does not restore the trust, because it has a new ID ([principals](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html)). So a replaced role silently locks out the systems that it trusted, and the systems that trusted it. Trust policies that name temporary session ARNs have the same weakness.

#### Secrets in State {#secrets-in-state}

> type: warning

An access key, a secret value or a database password in the state is readable by anyone who can read the state bucket. A sensitive mark only hides the value in plan output. Three ways reduce the exposure: write-only arguments, programs that read secrets at run time by ARN, and roles instead of access keys. An imported access key has no secret in the state at all ([access key doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/iam_access_key.html.markdown#L122)).

#### Deleted Secrets Hold Their Names {#deleted-secrets-hold-names}

> type: warning

A deleted secret waits in a recovery window, 30 days by default ([DeleteSecret](https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_DeleteSecret.html)). During that time, a new secret with the same name fails with "The secret is scheduled for deletion" ([CreateSecret](https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_CreateSecret.html)). So a rename or a replacement of a secret resource cannot reuse the name at once. The recovery window is also the only way back from a wrong destroy. `recovery_window_in_days = 0` deletes with no recovery. `prevent_destroy` stops the destroy in the plan.

#### IAM Size Limits {#iam-size-limits}

> type: warning

The inline policies of one role can hold 10,240 characters in total, and one managed policy can hold 6,144 ([IAM quotas](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html)). A role can have 20 attached managed policies by default, and 25 at most. A growing `for_each` that adds statements can pass the plan and fail the apply at one of these limits.

### Provider Version Pitfalls {#aws-provider-version-pitfalls}

> type: concept

These pitfalls come from Terraform and provider versions.

#### Old S3 Backend Arguments {#old-s3-backend-arguments}

> type: warning

Terraform 1.10.0 removed the old top-level arguments for assuming a role in `backend "s3"`, such as `role_arn`; it requires an `assume_role` block ([Terraform 1.10.0 changes](https://github.com/hashicorp/terraform/blob/v1.10.0/CHANGELOG.md#L35)). A configuration with the old form therefore keeps the whole pipeline on Terraform 1.9. That also blocks write-only arguments, which need 1.11. Terraform 1.11.0 deprecated DynamoDB locking in favor of `use_lockfile` ([Terraform 1.11.0 changes](https://github.com/hashicorp/terraform/blob/v1.11.0/CHANGELOG.md#L10)).

#### Inline Security Group Rules Win {#inline-rules-win}

> type: warning

A security group with inline `ingress` or `egress` blocks owns all its rules. Its next apply removes rules that someone added by hand, by script or from another configuration. The provider warns against mixing inline rules with separate rule resources ([security group note](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/security_group.html.markdown#L13-L15)). Use one separate resource for each rule, such as `aws_vpc_security_group_ingress_rule`.

#### Version Bugs {#version-bugs}

> type: warning

Each provider release fixes bugs, and some releases add them. For example, on versions 6.0 to 6.8, a tag change on `aws_ecs_service` appeared in every plan, but the apply never wrote it; 6.9.0 fixed it ([issue 43570](https://github.com/hashicorp/terraform-provider-aws/issues/43570)). A patch-level pin, such as `~> 6.8.0`, keeps such a bug. With no committed lock file, a loose constraint such as `>= 6.5.0` installs each new major version as soon as it appears. Commit the lock file, and raise versions on purpose.

## Good and Bad Decisions {#aws-decisions}

> type: category

Each node compares a choice with its alternative.

### Roles Instead of Access Keys {#roles-instead-of-access-keys}

> type: concept

An IAM user with an access key has a long-lived secret, and Terraform stores it in the state. A role gives temporary credentials, and there is no secret to store. AWS recommends temporary credentials for workloads ([IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)). Use access keys only for a system that cannot assume a role, and plan their rotation.

### Choose the External ID {#aws-choose-the-external-id}

> type: concept

When both sides of a trust use an external ID that you chose, one apply creates the role and the other system's integration together. Nobody copies values, and a recreated integration keeps working. Because AWS does not treat the external ID as a secret, it can live in configuration ([external ID](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios_third-party.html)).

### Protect Buckets That Hold Data {#protect-buckets-that-hold-data}

> type: concept

A bucket for test files can use `force_destroy`, so it is easy to remove. A bucket with data that cannot be rebuilt should keep `force_destroy` false and add `lifecycle { prevent_destroy = true }`. Versioning and expiry rules for old versions then protect against overwrites without growing the bucket forever.

### Contracts Between Configurations {#contracts-between-configurations}

> type: concept

One configuration can need a value that another one created, such as a certificate ARN or a queue ARN. Three ways exist:

- Copy the value by hand into variables. It is simple, but the copy goes stale without warning.
- Look the resource up by name or tag with a data source. The name becomes the contract.
- Publish the value as an SSM parameter in one configuration and read it with `data.aws_ssm_parameter` in the other. The parameter name becomes the contract, and a change flows on the next plan.

The last two follow a change; the first one does not.

## About the Sources {#aws-about-the-sources}

> type: comment

The provider facts come from the `hashicorp/aws` repository at tag v6.73.0 and the `hashicorp/awscc` repository at tag v1.110.0, the latest releases on 2026-11-23. The AWS facts come from docs.aws.amazon.com on the same day, and the Terraform facts from the Terraform changelogs at tags v1.10.0 and v1.11.0.
