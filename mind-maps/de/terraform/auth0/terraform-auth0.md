# Terraform for Auth0 {#terraform-auth0}

> type: category

The Auth0 provider lets Terraform configure an Auth0 tenant through the Auth0 Management API ([provider docs](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/index.md#L9-L14)). Auth0 maintains it under the source address `auth0/auth0`. Auth0 is a hosted sign-in service; the [authentication map](../../authentication/authentication.md#auth0-auth0) explains what it does and how OAuth works. The [Terraform map](../terraform.md) explains Terraform itself.

The unit is one tenant. A tenant is one isolated Auth0 space with its own users, applications and settings. One provider configuration manages exactly one tenant, and the API cannot create tenants, so someone creates each tenant by hand first ([tenant doc](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/tenant.md#L11-L12)). Terraform signs in to the tenant as one machine-to-machine application, and the scopes of that application decide what Terraform can read and change ([One Tenant, One Identity](#one-tenant-one-identity-one-tenant-one-identity)).

A tenant holds these objects:

- Applications, also called clients: the apps that sign users in, and the programs that call APIs.
- APIs, also called resource servers, with their scopes.
- Client grants, which allow a machine-to-machine application to call an API.
- Connections: the places where users sign in, such as a user database or a company directory.
- Roles, organizations, and actions, which are small programs that run during sign-in.
- Log streams, which send the tenant's logs to another system.
- Tenant-wide settings, such as branding and the sign-in pages.

The provider has three families of resources for these objects ([Three Families of Resources](#three-families-of-resources-three-families-of-resources)). The family decides what an import ID looks like and what a destroy does. Three more ideas explain most plans:

- For links between objects, a plural resource owns the whole list, and a singular resource adds one item ([Plural Owns, Singular Adds](#plural-owns-singular-adds-plural-owns-singular-adds)).
- An action is a draft until it is deployed, and it runs only when it is bound to a flow ([Actions Have Three States](#actions-have-three-states-actions-have-three-states)).
- The state holds client secrets, action secrets and identity-provider secrets ([State Is a Secrets Store](#state-is-a-secrets-store-state-is-a-secrets-store)).

Now compare Aiven. Both providers are remote controls for a service that someone else runs. In Auth0, however, people also change the same tenant in its dashboard every day, so drift is normal.

The map has four parts. Mental Models explains the ideas above. Reading an Auth0 Configuration says what you will see and where to look first. Pitfalls lists the failures that experienced users plan around. Good and Bad Decisions compares the main design choices.

## Mental Models {#auth0-mental-models}

> type: category

The models come in dependency order: each model uses only the models before it.

### One Tenant, One Identity {#one-tenant-one-identity}

> type: concept

> edge.related: empty-secret-without-scope

The provider signs in with a domain and the client ID and secret of a machine-to-machine application, or with a signed JWT, or with an API token ([provider configuration](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/index.md#L48-L63)). It reads the values from environment variables such as `AUTH0_DOMAIN`, `AUTH0_CLIENT_ID` and `AUTH0_CLIENT_SECRET`.

Every call uses the scopes that this application holds on the Management API. A missing read scope does not cause an error; the provider just reads an empty value. For example, `client_secret` is an empty string unless the application has the `read:client_keys` scope ([credentials doc](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/client_credentials.md#L182)). Connection options need the scopes `read:connections_options` and `update:connections_options` ([options scope](https://auth0.com/docs/troubleshoot/product-lifecycle/deprecations-and-migrations/migrate-connection-options-scope)).

Auth0 recommends one tenant for each environment, such as development, staging and production ([multiple environments](https://auth0.com/docs/get-started/auth0-overview/create-tenants/set-up-multiple-environments)). The IDs of objects differ from tenant to tenant, so configurations refer to objects through resources or data sources, never through copied IDs.

### Three Families of Resources {#three-families-of-resources}

> type: concept

| Family | Examples | Import ID | What a destroy does |
| --- | --- | --- | --- |
| Objects | `auth0_client`, `auth0_connection`, `auth0_action`, `auth0_log_stream` | The Auth0 ID, such as `con_…` or `lst_…` | Deletes the object |
| Links | `auth0_connection_client`, `auth0_trigger_actions`, `auth0_client_grant` | Two IDs joined by `::`, such as `con_…::<client ID>` | Removes the link |
| Singletons | `auth0_tenant`, `auth0_branding`, `auth0_prompt`, `auth0_guardian` | Any random UUID | Differs by resource |

The import formats come from the Import section of each [resource doc](https://github.com/auth0/terraform-provider-auth0/tree/v1.63.0/docs/resources). A singleton is a setting that exists once per tenant. Its destroy can do nothing, delete something, or turn a protection off; the pitfall Singletons on Destroy lists which.

### Plural Owns, Singular Adds {#plural-owns-singular-adds}

> type: concept

> edge.related: mixed-trigger-bindings, two-states-one-object

Many links exist in two versions:

| Link | Plural resource: owns the whole list | Singular resource: adds one item |
| --- | --- | --- |
| Apps that may use a connection | `auth0_connection_clients` | `auth0_connection_client` |
| Actions bound to a trigger | `auth0_trigger_actions` | `auth0_trigger_action` |
| Scopes of an API | `auth0_resource_server_scopes` | `auth0_resource_server_scope` |
| Permissions of a role | `auth0_role_permissions` | `auth0_role_permission` |

A plural resource writes exactly its own list and removes everything else. A singular resource appends one item and ignores the rest. The provider warns not to use both for the same parent ([connection clients](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/connection_client.md#L11-L14); [trigger actions](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/trigger_actions.md#L11-L13)). Choose one style for each parent, across all repos and states.

### Connection Options Are One Document {#connection-options-are-one-document}

> type: concept

> edge.related: connection-options-replaced

The `options` block of an `auth0_connection` holds the details of a sign-in method, such as the directory of an identity provider and its client secret. The provider sends `options` as one whole document. The docs warn that every nested field must be set, or existing settings can be lost ([connection doc](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/connection.md#L14)). So a small change to one option rewrites them all, with the values that Terraform knows.

### Actions Have Three States {#actions-have-three-states}

> type: concept

> edge.related: deploy-and-bind

An action is code that runs in a flow, such as `post-login`. It has three states ([write your first action](https://auth0.com/docs/customize/actions/write-your-first-action)):

1. A draft. `auth0_action` saves the code, and with the default `deploy = false` it stays a draft.
2. A deployed version. With `deploy = true`, the code becomes the live version.
3. Bound to a flow. Only a binding, through `auth0_trigger_actions` or `auth0_trigger_action`, makes the action run.

So a code change in Terraform can stay a draft, and a deployed action can stay unused. A bound action with `deploy = true`, however, changes live sign-ins at the next apply.

### State Is a Secrets Store {#state-is-a-secrets-store}

> type: concept

> edge.related: data-sources-copy-secrets

The state of an Auth0 configuration holds several kinds of secret:

- The client secrets of applications, in `auth0_client_credentials`.
- The `secrets` of actions, in plain text ([action doc](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/action.md#L16-L18)).
- The client secret of an identity provider, inside connection `options`.

Newer provider versions add write-only arguments, such as `secrets_wo` and `client_secret_wo`, which keep a value out of the state. They need Terraform 1.11 or later.

## Reading an Auth0 Configuration {#reading-auth0-configuration}

> type: category

These nodes say what you will find in the files, in which order to read them, and what each kind of plan line means.

### What You Will See {#auth0-what-you-will-see}

> type: concept

| Resource | Auth0 object | What to check |
| --- | --- | --- |
| `auth0_client` | application | `app_type`, callback and origin lists |
| `auth0_client_credentials` | sign-in method and secret of an application | Where the secret goes next |
| `auth0_resource_server` | API | Its identifier, which tokens use as the audience |
| `auth0_client_grant` | access of an application to an API | Which scopes, especially on the Management API |
| `auth0_connection` | sign-in method | `strategy`, a complete `options` block, secret sources |
| `auth0_connection_client(s)` | apps enabled on a connection | Singular or plural, and who else manages them |
| `auth0_action`, `auth0_trigger_action(s)` | code in a flow, and its binding | `deploy`, runtime, secrets, where the binding lives |
| `auth0_role`, `auth0_organization` | access groups | Who assigns users to them |
| `auth0_log_stream` | export of tenant logs | `type`, `filters`, the target |
| `auth0_tenant`, `auth0_branding`, `auth0_prompt` | tenant-wide settings | Exactly one owner per tenant |

An application of type `non_interactive` is a machine-to-machine application. A client grant to the Management API lets such an application change the tenant itself, so these grants deserve a careful look.

### Where to Look First {#auth0-where-to-look-first}

> type: concept

Read an Auth0 configuration in this order:

1. `required_providers`, the lock file and the provider block. Note the version gap to the latest release, and where the credentials and the tenant domain come from.
2. The backend and the environment selector. Map each state to a tenant, and flag any tenant that two states change.
3. The singletons. Each one needs exactly one owner per tenant.
4. The link resources. Look for singular and plural styles on the same parent, in this repo and in others.
5. `auth0_action`: `deploy`, the runtime, the secrets, and where its binding lives.
6. `auth0_connection`: the strategy, the full `options` block and its secrets.
7. `auth0_client`, its credentials and grants: the app type, the callback and origin lists, and grants to the Management API.
8. The data sources and outputs: lookups by name, and secrets that leave for other systems.
9. The `lifecycle` blocks: each `ignore_changes` marks a field that a person or another state owns.
10. `auth0_log_stream`: the filters and the target system.

### Plan Signals {#auth0-plan-signals}

> type: concept

| Line in the plan | Meaning |
| --- | --- |
| `+` on an object that exists in the tenant | Someone deleted it in the dashboard, or another state deleted its link; Terraform recreates it. |
| `~` on `options` of a connection | The whole options document will be rewritten. |
| `~ cross_origin_auth = true -> null` in every plan | A field with a server default that the code does not set ([issue 1029](https://github.com/auth0/terraform-provider-auth0/issues/1029)). |
| `~` on `auth0_trigger_actions` | The plural binding will write its exact list; bindings made elsewhere disappear. |
| `-/+ auth0_log_stream` | The stream is replaced; a stream to AWS EventBridge gets a new partner event source. |

## Pitfalls {#auth0-pitfalls}

> type: category

The pitfalls fall into three groups: ownership, secrets, and change management.

### Ownership Pitfalls {#auth0-ownership-pitfalls}

> type: concept

These pitfalls appear when two owners act on one tenant.

#### Mixed Trigger Bindings {#mixed-trigger-bindings}

> type: warning

`auth0_trigger_actions` writes its exact list of actions on every create and update, and writes an empty list on destroy. So its destroy unbinds every action of the flow, including actions that another configuration bound with `auth0_trigger_action` ([code](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/internal/auth0/action/resource_trigger_actions.go#L76-L139)). `auth0_trigger_action` also cannot set the order: it appends its action at the end of the flow ([trigger action doc](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/trigger_action.md#L10-L12)). Bind all actions of one flow in one place.

#### Two States, One Object {#two-states-one-object}

> type: warning

Two configurations can manage the same object without knowing it. For example, the create of `auth0_client_grant` adopts an existing grant for the same application and API ([grant code](https://github.com/auth0/terraform-provider-auth0/blob/v1.27.0/internal/auth0/client/resource_grant.go#L70-L96)). When either configuration destroys it, the other loses it. A `data "auth0_client"` lookup by name returns the first match, so a duplicate name silently picks the wrong application ([data source code](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/internal/auth0/client/data_source.go#L314-L340)).

#### Singletons on Destroy {#singletons-on-destroy}

> type: warning

A destroy of a tenant-wide setting does different things:

| Resource | What a destroy does |
| --- | --- |
| `auth0_tenant`, `auth0_prompt` | Nothing in Auth0; it only leaves the state ([tenant code](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/internal/auth0/tenant/resource.go#L642-L644)). |
| `auth0_branding` | Deletes the Universal Login template ([branding code](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/internal/auth0/branding/resource.go#L177-L188)). |
| `auth0_attack_protection` | Turns off breached-password detection, brute-force protection and IP throttling ([code](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/internal/auth0/attackprotection/resource.go#L837-L879)). |
| `auth0_guardian` | Turns off all multi-factor authentication factors ([code](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/internal/auth0/guardian/resource.go#L693-L709)). |

So removing a singleton from the code can weaken the security of a production tenant.

### Secret Pitfalls {#auth0-secret-pitfalls}

> type: concept

These pitfalls put secrets where they do not belong, or lose them.

#### Data Sources Copy Secrets {#data-sources-copy-secrets}

> type: warning

`data "auth0_client"` stores the client secret of the application that it reads. `data "auth0_connection"` stores the connection options, including an identity provider's secret. So every configuration that only reads these objects also holds their secrets in its state. Newer versions add `hide_client_secret` to both data sources ([client data source](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/data-sources/client.md#L31); [connection data source](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/data-sources/connection.md#L31)).

#### Empty Secret Without Scope {#empty-secret-without-scope}

> type: warning

If Terraform's application lacks `read:client_keys`, `client_secret` reads as an empty string, with no error ([credentials doc](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/client_credentials.md#L182)). A configuration that copies the secret into AWS Secrets Manager or into an action secret then stores an empty value. The plan looks clean, and the app fails to sign in later. A `postcondition` that checks the length of the secret catches this at plan time.

#### Connection Options Replaced Whole {#connection-options-replaced}

> type: warning

Because `options` is one document, an update sends all of it. Fields that the code leaves out are wiped, secrets included, and `ignore_changes` inside `options` does not prevent this ([issue 860](https://github.com/auth0/terraform-provider-auth0/issues/860)). Without the options scopes, the API leaves `options` out of its answers, and updates that contain `options` fail ([options scope](https://auth0.com/docs/troubleshoot/product-lifecycle/deprecations-and-migrations/migrate-connection-options-scope)). Write the full `options` block, and add a precondition that the secret inputs are not empty.

#### Secret Rotation {#secret-rotation}

> type: warning

Auth0 rotates a client secret in the dashboard or through the API, and a new secret can take up to 30 seconds to work ([rotate a client secret](https://auth0.com/docs/get-started/applications/rotate-client-secret)). Replacing `auth0_client_credentials` in Terraform does not rotate the secret ([issue 1785](https://github.com/auth0/terraform-provider-auth0/issues/1785)). A rotated secret can also have copies, for example in Secrets Manager, in an action secret or in a pipeline variable. Each copy changes only at the next apply of the configuration that writes it.

### Change Pitfalls {#auth0-change-pitfalls}

> type: concept

These pitfalls appear during normal changes and upgrades.

#### Deploy and Bind {#deploy-and-bind}

> type: warning

With the default `deploy = false`, a code change in `auth0_action` stays a draft, and nothing changes for users. With `deploy = true`, the change goes live at once in every flow that binds the action. Deleting an action that is still bound fails with `409 Conflict`. Unbind it in one apply, and delete it in the next ([trigger actions doc](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/trigger_actions.md#L17-L19)).

#### Lists Stay Unless Emptied {#lists-stay-unless-emptied}

> type: warning

Removing a list argument, such as `callbacks` or `allowed_origins`, from the code changes nothing in Auth0. To clear a list, set it to `[]` ([issue 1108](https://github.com/auth0/terraform-provider-auth0/issues/1108)). `allowed_origins` also defaults to the callback URLs ([client doc](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/resources/client.md#L213)). So a localhost callback left in a development list also becomes an allowed origin.

#### Rate Limits {#rate-limits}

> type: warning

The Management API has rate limits per tenant ([rate limit policy](https://auth0.com/docs/troubleshoot/customer-support/operational-policies/rate-limit-policy)). A large configuration reads many objects in every plan, even with no changes, and it can hit the limit ([issue 1488](https://github.com/auth0/terraform-provider-auth0/issues/1488)). The provider then waits and retries, so plans get slow. Calls that Terraform makes share the limit with the apps that use the tenant.

#### Version Gaps and Deprecations {#auth0-version-gaps}

> type: warning

Auth0 changes its API on a schedule, and old provider versions fall behind it. For example, the old way of managing the enabled clients of a connection reached its end of life on 2026-07-13 ([deprecations](https://auth0.com/docs/troubleshoot/product-lifecycle/deprecations-and-migrations)). Auth0 names provider 1.29.0 or later as compatible with the new endpoints ([dedicated connection endpoints](https://auth0.com/docs/troubleshoot/product-lifecycle/deprecations-and-migrations/migrate-dedicated-connection-endpoints)). The same page lists Rules and Hooks as deprecated in favor of actions. Read the changelog before each upgrade, and upgrade in steps.

## Good and Bad Decisions {#auth0-decisions}

> type: category

Each node compares a choice with its alternative.

### One Tenant per Environment {#one-tenant-per-environment}

> type: concept

Auth0 recommends separate tenants for development, staging and production ([multiple environments](https://auth0.com/docs/get-started/auth0-overview/create-tenants/set-up-multiple-environments)). A shared tenant saves a tenant and lets staging test the real company directory. But then the tenant-wide settings need guards in code, two states share objects, a staging log stream also exports production events, and a staging apply can change production actions. A tenant used by production should also carry the production tag, because the production rate limits apply only to such tenants (same page).

### Least Privilege for Terraform's Client {#least-privilege-for-terraform}

> type: concept

The provider's quickstart authorizes all Management API scopes for Terraform's application ([quickstart](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/guides/quickstart.md#L27-L28)). That is easy, but it includes every delete scope. A smaller set must still include the read scopes that some attributes depend on, such as `read:client_keys` and `read:connections_options`; otherwise values read as empty. Keep the credentials in environment variables, not in code, and prefer a signed JWT over a shared secret ([provider configuration](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/index.md#L29-L41)).

### One Owner per Parent {#one-owner-per-parent}

> type: concept

Use a plural link resource when one configuration owns the parent: it removes strays and keeps the order of actions. Use singular resources when several configurations each add their own item. Never mix the two on one parent (see Plural Owns, Singular Adds).

### Start from the Generated Config {#start-from-generated-config}

> type: concept

The Auth0 CLI command `auth0 tf generate` writes Terraform files and import blocks for an existing tenant ([generate guide](https://github.com/auth0/terraform-provider-auth0/blob/v1.63.0/docs/guides/generate_terraform_config.md)). It is experimental, and it does not export secrets. It writes plural resources such as `auth0_trigger_actions`, so check for singular resources on the same parents before you merge its output.

## About the Sources {#auth0-about-the-sources}

> type: comment

The provider facts come from the `auth0/auth0` repository at tag v1.63.0, the latest release on 2026-11-23, and at tag v1.27.0 for one older code path. The Auth0 facts come from auth0.com/docs on the same day.
