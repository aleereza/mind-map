# Terraform and Azure DevOps {#terraform-azure-devops}

> type: category

Azure DevOps plays two parts in a Terraform setup. It is a system that Terraform manages, through the provider `microsoft/azuredevops` ([provider docs](https://registry.terraform.io/providers/microsoft/azuredevops/latest/docs)). It is also the runner: Azure Pipelines runs `terraform plan` and `apply` for every other system, such as AWS, Snowflake and Aiven. This map covers both parts. The [Terraform map](../terraform.md) explains Terraform itself.

The unit is one Azure DevOps organization, which holds projects. A project holds Git repositories, pipelines and the resources that pipelines use. A pipeline is a YAML file in a repository plus a pipeline definition in Azure DevOps that points to it. Pipelines reach other systems through these resources:

- A service connection stores the credentials for one outside system, such as an AWS account ([Service Connections](#service-connections-service-connections)).
- A variable group stores named values and secrets for many pipelines.
- An environment is a named target, such as `production`, that can carry checks, such as approvals.
- A secure file stores a file, such as a private key, for use in a run.

Three gates decide whether a run may use such a resource ([Three Gates on a Resource](#three-gates-on-a-resource-three-gates-on-a-resource)). The most important gate, the checks, lives outside the YAML file ([Checks Live Outside the YAML](#checks-live-outside-the-yaml-checks-live-outside-the-yaml)).

When Azure Pipelines runs Terraform, three more ideas explain what happens:

- Each job starts on a fresh machine. A plan file from one job is missing in the next job, unless the pipeline publishes it ([Each Job Starts Fresh](#each-job-starts-fresh-each-job-starts-fresh)).
- The pipeline task that runs Terraform decides which credentials reach Terraform ([The Task Shapes the Run](#the-task-shapes-the-run-the-task-shapes-the-run)).
- Azure Pipelines hides secrets in logs, but only when they appear in full ([Masking Is Best Effort](#masking-is-best-effort-masking-is-best-effort)).

Now compare dbt Cloud. A dbt Cloud job holds the commands and the schedule of a dbt run. An Azure pipeline holds the commands and the triggers of a Terraform run, and its environment checks play the role of a release approval.

The map has four parts. Mental Models explains the ideas above. Reading Pipelines and Configurations says where to look first. Pitfalls lists the failures that experienced users plan around. Good and Bad Decisions compares the main design choices.

## Mental Models {#azdo-mental-models}

> type: category

The models come in dependency order: each model uses only the models before it.

### Service Connections {#service-connections}

> type: concept

A service connection holds the credentials for one outside system. For AWS, it holds an access key or an OIDC setup, and optionally a role to assume ([AWS connection doc](https://github.com/microsoft/terraform-provider-azuredevops/blob/v1.16.0/website/docs/r/serviceendpoint_aws.html.markdown)). A pipeline refers to a connection by its name. Authorizations and checks refer to it by its ID. The secrets can be written but never read back, so Terraform cannot see a secret that someone changed in the web interface.

OIDC stands for OpenID Connect. With OIDC, Azure DevOps gives each run a short-lived token, and the outside system trusts that token instead of a stored key. Microsoft recommends this workload identity federation over stored secrets ([security overview](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/overview)). The [authentication map](../../authentication/authentication.md#workload-identity-federation-workload-identity-federation) explains the idea.

### Three Gates on a Resource {#three-gates-on-a-resource}

> type: concept

> edge.related: new-connections-not-authorized

A run can use a service connection, a variable group or an environment only after three gates ([resource permissions](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/resources)):

1. User roles decide who may see, use or administer the resource.
2. Pipeline permissions decide which pipelines may use it. A run of an unauthorized pipeline stops with "This pipeline needs permission to access a resource before this run can continue" ([service connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints)).
3. Checks, such as an approval or a branch control, run each time a stage uses the resource.

In Terraform, `azuredevops_pipeline_authorization` grants the second gate. Without `pipeline_id`, it authorizes every pipeline of the project ([authorization doc](https://github.com/microsoft/terraform-provider-azuredevops/blob/v1.16.0/website/docs/r/pipeline_authorization.html.markdown#L111)).

### Checks Live Outside the YAML {#checks-live-outside-the-yaml}

> type: concept

Microsoft puts it this way: "Users modifying the pipeline yaml file can't modify the checks" ([approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals)). Administrators of the resource manage its checks in the web interface or in Terraform. So a check on an environment or a service connection protects production even when someone edits the pipeline in a pull request.

Gates written in YAML are weaker. A `condition` on a stage, such as "only on the main branch", lives in the same file that a pull request can change. The task `ManualValidation` pauses a run in YAML, and any user with the Queue builds permission can resume it ([manual validation](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/manual-validation-v0)).

### Triggers Come from Two Places {#triggers-come-from-two-places}

> type: concept

For a repository in Azure Repos, a continuous integration (CI) trigger comes from the `trigger` section of the YAML file. A pull request trigger does not come from YAML: a branch policy on the target branch starts the run ([PR triggers](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git)). So the `paths` in the YAML decide when a merge starts a run, while the path filter of the branch policy decides when a pull request starts one. The two lists must match by hand.

### Each Job Starts Fresh {#each-job-starts-fresh}

> type: concept

> edge.related: apply-runs-unreviewed-plan

Each job of a pipeline can run on a different machine. A saved plan file from the plan job therefore does not exist in the apply job, unless the pipeline publishes it as an artifact and downloads it again. HashiCorp describes what that needs: the whole working directory, with the same files, at the same path ([running Terraform in automation](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform)). Without it, the apply job makes a new plan and applies that one.

### The Task Shapes the Run {#the-task-shapes-the-run}

> type: concept

> edge.related: connection-role-ignored

A pipeline can run Terraform through a task, such as the community task TerraformCLI. The task turns a service connection into environment variables and command-line options. TerraformCLI, for example, exports only the access key, the secret key and the region of an AWS connection ([task code](https://github.com/jason-johnson/azure-pipelines-tasks-terraform/blob/1.2.2/tasks/terraform-cli/src/providers/aws.ts#L23-L27)). It also adds `-auto-approve` to every `apply` ([task code](https://github.com/jason-johnson/azure-pipelines-tasks-terraform/blob/1.2.2/tasks/terraform-cli/src/commands/tf-apply.ts#L20)). So read the task's source or docs to learn what a run really does.

### Masking Is Best Effort {#masking-is-best-effort}

> type: concept

> edge.related: multi-line-secrets-leak

Azure Pipelines replaces each secret value with `***` in the logs. It does not mask parts of a secret: "We never mask substrings of secrets" ([set secret variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-secret-variables)). A secret that a script splits, encodes or prints line by line can therefore appear in clear text. Microsoft advises against structured data, such as JSON, as a secret value.

## Reading Pipelines and Configurations {#reading-azdo}

> type: category

These nodes say where to look first in a pipeline that runs Terraform, and in a Terraform configuration that manages Azure DevOps.

### Reading a Terraform Pipeline {#reading-a-terraform-pipeline}

> type: concept

Read the pipeline YAML of a Terraform root module in this order:

1. Find the variable or parameter that names the working directory. It binds the pipeline to its root module.
2. Read `trigger` and its `paths`, then the branch policy of the main branch. Pull request runs come only from the policy.
3. Follow each `template:` reference. The steps can live in shared template files.
4. Find the condition of the apply step: the branch, `Build.Reason` and any override parameter. It tells you what can reach production.
5. Look for `deployment:` jobs with an `environment:`, then read that environment's checks in the web interface. They are the real gate.
6. List the service connection inputs of the Terraform tasks. For each connection, read its role, its pipeline permissions and its checks.
7. Read how the run selects its state: the workspace, `-backend-config` values and `-var-file` values.
8. List the secret sources: variable groups, `$(variable)` macros, `TF_VAR_*` environment variables and secure files.
9. Check whether the apply uses the saved plan file, and whether the Terraform version is pinned.

### Reading an Azure DevOps Configuration {#reading-an-azdo-configuration}

> type: concept

| Resource | Azure DevOps object | What to check |
| --- | --- | --- |
| `azuredevops_project` | project | Usually looked up with a data source, not created |
| `azuredevops_serviceendpoint_aws` | AWS service connection | Static keys or `use_oidc`, role to assume |
| `azuredevops_pipeline_authorization` | permission of a pipeline to use a resource | `pipeline_id` set, or open to all pipelines |
| `azuredevops_serviceendpoint_permissions` | user roles on a connection | Which groups may use or administer it |
| `azuredevops_variable_group` | variable group | Secret values in the state |
| `azuredevops_environment`, `azuredevops_check_approval` | environment and its checks | Who approves, and for which stages |
| `azuredevops_build_definition` | pipeline definition | YAML path, and triggers from YAML or the definition |

Read the version constraint and the lock file first, because older versions behave differently. Then find the provider's credential: a personal access token (PAT), a service principal or a managed identity ([provider docs](https://github.com/microsoft/terraform-provider-azuredevops/blob/v1.16.0/website/docs/index.html.markdown)). Last, compare each connection, variable group and environment with the `azuredevops_pipeline_authorization` resources. A resource without one was authorized by hand.

## Pitfalls {#azdo-pitfalls}

> type: category

The pitfalls fall into two groups: running Terraform in pipelines, and managing Azure DevOps with Terraform.

### Pipeline Pitfalls {#azdo-pipeline-pitfalls}

> type: concept

These pitfalls affect every system that a pipeline deploys.

#### Apply Runs an Unreviewed Plan {#apply-runs-unreviewed-plan}

> type: warning

When the apply step runs without the saved plan file, Terraform makes a new plan and applies it at once, because the task adds `-auto-approve`. That new plan can differ from the one that reviewers saw: drift, a merge in between, or a changed secret all count. HashiCorp says a plan made without `-out` gives no guarantee about what the apply will do ([plan command](https://developer.hashicorp.com/terraform/cli/commands/plan)). Publish the plan file and apply exactly that file. A saved plan already holds the variable values, so the apply step must not pass `-var` again ([apply command](https://developer.hashicorp.com/terraform/cli/commands/apply)).

#### Connection Role Ignored {#connection-role-ignored}

> type: warning

An AWS service connection can name a role to assume. TerraformCLI exports only the keys and the region, so Terraform runs as the IAM user behind the keys, not as the role. Configurations work around this with `assume_role` in the provider block. A run that relies on the connection's role instead fails with "AccessDenied" or reaches the wrong account.

#### Multi-Line Secrets Leak {#multi-line-secrets-leak}

> type: warning

A private key in PEM format has many lines. Passed on the command line as `-var key=$(secret)`, each line can appear in the log, because Azure Pipelines masks only the full value ([secrets](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/secrets)). Pass such secrets as `TF_VAR_*` environment variables or as secure files, and rotate any secret that has leaked. Watch for a trailing newline, which can make Terraform see a changed value.

#### Locks and Waiting Runs {#locks-and-waiting-runs}

> type: warning

With `batch: true`, a new CI run waits until the current run completes ([batching](https://learn.microsoft.com/en-us/azure/devops/pipelines/repos/azure-repos-git)). A run that waits for an approval therefore holds back every later merge. The YAML setting `lockBehavior` has effect only together with an exclusive lock check on the resource ([exclusive lock](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals)). Terraform's own state lock is separate. A plan with `-lock=false` can read state while another run writes it.

#### Misspelled Environments {#misspelled-environments}

> type: warning

When a pipeline names an environment that does not exist, Azure Pipelines creates it automatically when it can ([environments](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/environments)). The new environment has no checks. So a typo in the environment name of a production stage removes its approval without any error.

### Provider Pitfalls {#azdo-provider-pitfalls}

> type: concept

These pitfalls affect configurations that manage Azure DevOps objects.

#### New Connections Are Not Authorized {#new-connections-not-authorized}

> type: warning

A service connection that Terraform creates, or replaces, has no pipeline permission. The next run of each pipeline that uses it stops and waits for permission ([issue 1008](https://github.com/microsoft/terraform-provider-azuredevops/issues/1008)). Add an `azuredevops_pipeline_authorization` for each pipeline in the same configuration. Find the pipeline ID with the data source `azuredevops_build_definition`, instead of writing the number by hand; a recreated pipeline gets a new ID.

#### Secrets in State {#azdo-secrets-in-state}

> type: warning

The secret key of an AWS connection and the secret values of a variable group sit in the state in plain text. Anyone who can read the state bucket can read them. Because the provider never reads these secrets back, a secret changed in the web interface never shows in a plan. A connection with `use_oidc` holds no secret at all ([AWS connection doc](https://github.com/microsoft/terraform-provider-azuredevops/blob/v1.16.0/website/docs/r/serviceendpoint_aws.html.markdown#L55)).

#### Renamed and Idle Connections {#renamed-and-idle-connections}

> type: warning

A pipeline refers to a service connection by name, and an AWS role that trusts Azure DevOps through OIDC can also name it. Renaming the connection in Terraform therefore breaks every YAML file and trust policy that uses the old name. Separately, Azure Pipelines can disable a connection after 100 days without use ([service connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints)). The provider has no field for this state, so the plan stays clean while runs fail.

#### Token Lifetimes {#token-lifetimes}

> type: warning

A PAT expires, and on organizations that use Microsoft Entra ID it becomes inactive after 90 days without use ([personal access tokens](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate)). Microsoft is also retiring global PATs. The same page names 2027-01-27 in one place and 2026-12-01 in another as the end date. A configuration whose provider uses a PAT stops working when the token does.

## Good and Bad Decisions {#azdo-decisions}

> type: category

Each node compares a choice with its alternative.

### Apply the Saved Plan {#apply-the-saved-plan}

> type: concept

Plan once, publish the plan file with the working directory, review it, and apply that file. HashiCorp recommends this for automation ([running Terraform in automation](https://developer.hashicorp.com/terraform/tutorials/automation/automate-terraform)). Treat the plan file as a secret, because it holds the variable values and parts of the state. Re-planning in the apply job is simpler, but what it applies was never reviewed.

### Gates That YAML Cannot Change {#gates-that-yaml-cannot-change}

> type: concept

Put the approval for production on the environment or the service connection, as a check. A pull request cannot remove such a check. A stage `condition` or a `ManualValidation` step lives in YAML, so a pull request can change it. Pull request review alone is the weakest gate.

### One Identity per Environment {#one-identity-per-environment}

> type: concept

Microsoft recommends a branch control check on each service connection, and workload identity federation instead of stored secrets ([security overview](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/overview)). A separate connection for each environment, with production restricted to the main branch, keeps a pull request build away from production credentials. A plan on a pull request needs only read access; only the apply needs write access.

### Pin Terraform and Providers {#azdo-pin-terraform}

> type: concept

Pin the Terraform version in the install step, and commit `.terraform.lock.hcl`. A pipeline that installs `latest` can change behavior on any day without a code change. Pinning moves upgrades into reviewed pull requests.

## About the Sources {#azdo-about-the-sources}

> type: comment

The provider facts come from the `microsoft/azuredevops` repository at tag v1.16.0, the latest release on 2026-11-23. The task facts come from the TerraformCLI task repository at tag 1.2.2. The Azure Pipelines facts come from learn.microsoft.com, and the Terraform facts from developer.hashicorp.com, on the same day.
