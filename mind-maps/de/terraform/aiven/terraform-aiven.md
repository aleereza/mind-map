# Terraform for Aiven {#terraform-aiven}

> type: category

The Aiven provider lets Terraform manage Aiven projects, services and the objects inside services. Aiven maintains it under the source address `aiven/aiven` ([provider docs](https://registry.terraform.io/providers/aiven/aiven/latest/docs)). The provider calls the Aiven API with an API token ([provider configuration](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/internal/plugin/provider.go#L71-L83)). It does not connect to Kafka as a client. The [Kafka map](../../kafka/kafka.md#aiven-aiven) explains what Aiven runs, and the [Terraform map](../terraform.md) explains Terraform itself.

The unit is one Aiven project and the services in it. Aiven groups everything in three levels: an organization holds projects, and a project holds services ([Projects and Services](../../kafka/kafka.md#projects-and-services-aiven-projects-services)). A service is one running product, such as a Kafka cluster or a Kafka Connect cluster. Inside a Kafka service live the objects that a data engineer touches most: topics, access rules, schemas and connectors.

Each object has its own resource, and each resource has one key idea:

- A service is a machine with data. Its name is its identity, so a new name means a new, empty service ([A Service Holds Data](#a-service-holds-data-a-service-holds-data)).
- The advanced settings of a service only grow. Removing a setting from the code leaves the live value in place ([Service Settings Only Grow](#service-settings-only-grow-service-settings-only-grow)).
- A topic is known by its name. A new name means a new, empty topic ([The Topic Name Is the Key](#the-topic-name-is-the-key-the-topic-name-is-the-key)).
- A connector is a flat map of strings that the provider hardly checks ([A Connector Is a Map of Strings](#a-connector-is-a-map-of-strings-a-connector-is-a-map-of-strings)).
- A schema subject is a list of versions that only grows ([Schema Subjects Only Grow](#schema-subjects-only-grow-schema-subjects-only-grow)).
- Aiven has two kinds of access rule, and both can apply to one user ([Two ACL Models](#two-acl-models-two-acl-models)).

Two more ideas explain surprising plans. The provider is built on two Terraform libraries, and each resource behaves like its library ([Two Libraries in One Provider](#two-libraries-in-one-provider-two-libraries-in-one-provider)). Aiven also changes some things on its own, outside any plan ([Aiven Changes Things Too](#aiven-changes-things-too-aiven-changes-things-too)).

Now compare the Snowflake provider. Both manage a service that someone else runs. Snowflake's provider speaks SQL, and Aiven's provider speaks REST. In both, the most dangerous line in a plan is a replacement of an object that holds data.

The map has four parts. Mental Models explains the ideas above. Reading an Aiven Configuration says what you will see and where to look first. Pitfalls lists the failures that experienced users plan around. Good and Bad Decisions compares the main design choices.

## Mental Models {#aiven-mental-models}

> type: category

The models come in dependency order: each model uses only the models before it.

### A Service Holds Data {#a-service-holds-data}

> type: concept

> edge.related: service-replacement

An `aiven_kafka` or `aiven_kafka_connect` resource stands for a service: machines, disks and the software on them. A change of `service_name` or `project` replaces the service. The provider's own docs warn that the service and all its data are deleted before Aiven creates the new one ([provider warning](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/index.md#L43-L47)).

The same page asks you to set `termination_protection = true` on every production service. Aiven itself enforces this flag, so it also blocks a delete from the console or the CLI. It does not protect topics or other objects inside the service ([termination_protection](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka.md#L69)).

The children of a service, such as topics, ACLs and connectors, name the service in `project` and `service_name`. Those two fields force a replacement on each child too. So a replaced service also replaces everything inside it.

### Service Settings Only Grow {#service-settings-only-grow}

> type: concept

A Kafka service has a block of advanced settings, `kafka_user_config`. It holds broker defaults, the Kafka version, and switches for Kafka Connect, the Schema Registry and the REST proxy. The provider warns: "There's no way to reset advanced configuration options to default. Options that you add cannot be removed later" ([kafka_user_config](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka.md#L60)).

So the code shows only the settings that someone added and kept. A setting that someone removed from the code still runs on the service. To change it, set a new value; to learn the live value, read the service in the Aiven Console.

Broker defaults in this block also apply to topics. For example, `kafka.log_retention_bytes` sets the broker default for the topic setting `retention.bytes`, which every topic without its own value uses. That limit counts per partition ([Kafka topic settings](https://kafka.apache.org/39/generated/topic_config.html)).

### The Topic Name Is the Key {#the-topic-name-is-the-key}

> type: concept

> edge.related: topic-identity, partition-changes

An `aiven_kafka_topic` is known by three fields: `project`, `service_name` and `topic_name`. A change to any of them forces a new topic ([topic fields](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka_topic.md#L84-L87)). Kafka cannot rename a topic, so the old topic and its messages are deleted.

The provider also refuses two changes at plan time ([plan checks](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/internal/plugin/service/kafkatopic/topic.go#L161-L186)):

- A smaller `partitions` value fails with "number of partitions cannot be decreased".
- A new topic resource whose name already exists in the service fails with "topic conflict, already exists".

The `config` block holds only the topic settings that you set. Removing the block does not reset the topic: it keeps its last values ([config](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka_topic.md#L91)).

### A Connector Is a Map of Strings {#a-connector-is-a-map-of-strings}

> type: concept

> edge.related: connector-drift

An `aiven_kafka_connector` runs one Kafka Connect connector on a Connect service. Its `config` is one flat map of strings, the same keys that the Connect REST API takes. The provider checks only one thing: `config.name` must equal `connector_name` ([name check](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/internal/sdkprovider/service/kafka/kafka_connector.go#L115-L129)). It accepts any other key, including keys that Kafka Connect ignores.

A refresh copies the live configuration into the state, and an update sends the whole map. The resource has no field for task health, so a plan cannot show a connector whose tasks have failed. The [Kafka map](../../kafka/kafka.md#kafka-connect-service-aiven-kafka-connect) explains where Connect runs on Aiven.

### Schema Subjects Only Grow {#schema-subjects-only-grow}

> type: concept

> edge.related: schema-registry-edits

An `aiven_kafka_schema` stands for one subject in the Schema Registry, which Aiven runs with Karapace ([Karapace](../../kafka/kafka.md#karapace-karapace)). A subject is a list of schema versions. An edit to the schema in Terraform registers a new version; it never rewrites an old one. The plan checks the new schema against the version in the state, and it fails with "schema is not compatible with previous version" when the check fails ([schema resource](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/internal/sdkprovider/service/kafkaschema/kafka_schema.go#L444-L501)).

The compatibility rule comes from `compatibility_level` on the subject, or from the registry-wide `aiven_kafka_schema_configuration`. Destroying that resource sets the global level back to BACKWARD ([schema configuration](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/internal/sdkprovider/service/kafkaschema/kafka_schema_configuration.go#L129-L150)).

### Two ACL Models {#two-acl-models}

> type: concept

> edge.related: acl-matching

An ACL is an access control list: a rule that lets a service user do something. Aiven has two kinds ([ACLs](https://aiven.io/docs/products/kafka/concepts/acl)):

- Aiven ACLs, with `aiven_kafka_acl`. Each grants `read`, `write`, `readwrite` or `admin` on topics. Users and topics can use wildcards.
- Kafka-native ACLs, with `aiven_kafka_native_acl`. They follow Apache Kafka's own model, with `ALLOW` and `DENY` rules on more kinds of resources.

Both apply at the same time, and a `DENY` rule wins over any grant. In Aiven ACLs, `*` matches zero or more characters and `?` matches exactly one. The Schema Registry has its own ACLs, in `aiven_kafka_schema_registry_acl`.

### Two Libraries in One Provider {#two-libraries-in-one-provider}

> type: concept

> edge.related: floating-provider-version

The provider combines two Terraform libraries: the older SDKv2 and the newer Plugin Framework (PF). Aiven moves resources to PF one at a time. After a move, a resource stores only the values that you set, and it drops blocks that your configuration does not have ([upgrade guide](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/guides/upgrade-guide-v3-to-v4.md#L104-L191)). The first plan after such an upgrade can therefore show changes that remove blocks. Aiven says that applying them is safe.

| Resource | Moved to PF in |
| --- | --- |
| `aiven_kafka_schema_registry_acl` | 4.57.0 |
| `aiven_kafka_topic` | 4.59.0 |
| `aiven_kafka_acl`, `aiven_kafka_native_acl`, `aiven_kafka_user` | 4.61.0 |
| `aiven_service_integration`, `aiven_service_integration_endpoint` | 4.66.0 |

The table comes from the [changelog](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/CHANGELOG.md). `aiven_kafka`, `aiven_kafka_connect`, `aiven_kafka_connector`, the schema resources and `aiven_project` were still on SDKv2 in 4.66.0. Each release marks such changes with **IMPORTANT** in the changelog.

### Aiven Changes Things Too {#aiven-changes-things-too}

> type: concept

> edge.related: unpinned-plugin-versions

Aiven operates the services, so some changes happen outside Terraform:

- Maintenance updates install the latest connector plugin when no version is pinned ([connector versions](https://aiven.io/docs/products/kafka/kafka-connect/howto/manage-connector-versions)).
- Aiven issues new service-user certificates about three months before they expire. The certificates are valid for 820 days ([renew certificates](https://aiven.io/docs/products/kafka/howto/renew-ssl-certs)).
- The power state of a service is outside Terraform. You turn a service on or off in the Aiven Console or the CLI ([state](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka.md#L87)).
- After a power cycle, Aiven restores topics, users, ACLs, schemas and Connect settings from a configuration backup. It does not restore the messages in classic topics ([configuration backups](https://aiven.io/docs/products/kafka/concepts/configuration-backup)).

None of these changes shows in a plan before it happens.

## Reading an Aiven Configuration {#reading-aiven-configuration}

> type: category

These nodes say what you will find in the files, in which order to read them, and what each kind of plan line means.

### What You Will See {#aiven-what-you-will-see}

> type: concept

| Resource | Aiven object | What to check | Import ID |
| --- | --- | --- | --- |
| `aiven_project` | project | `aiven_organization_project` will replace it ([project doc](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/project.md#L11)) | `PROJECT` |
| `aiven_kafka` | Kafka service | `service_name`, `plan`, `termination_protection`, `kafka_user_config` | `PROJECT/SERVICE_NAME` |
| `aiven_kafka_connect` | Kafka Connect service | Its integration with the Kafka service | `PROJECT/SERVICE_NAME` |
| `aiven_service_integration`, `aiven_service_integration_endpoint` | link between services, and an outside target such as Prometheus | The type, and secrets in the endpoint | `PROJECT/INTEGRATION_ID` |
| `aiven_kafka_topic` | topic | Name source, `partitions`, `replication`, `config` | `PROJECT/SERVICE_NAME/TOPIC_NAME` |
| `aiven_kafka_user` | service user | Password and key in state | `PROJECT/SERVICE_NAME/USERNAME` |
| `aiven_kafka_acl`, `aiven_kafka_native_acl` | access rule | Wildcard reach; every field forces a new rule | `PROJECT/SERVICE_NAME/ACL_ID` |
| `aiven_kafka_schema`, `aiven_kafka_schema_configuration` | subject, registry-wide rule | Compatibility level | `PROJECT/SERVICE_NAME/SUBJECT` |
| `aiven_kafka_connector` | connector | Target service, secrets in `config`, stable `connector_name` | `PROJECT/SERVICE_NAME/CONNECTOR_NAME` |

The import IDs come from the Import section of each [resource doc](https://github.com/aiven/terraform-provider-aiven/tree/v4.66.0/docs/resources). Aiven generates the ID of an ACL, so you find it in the console or the API before an import.

### Where to Look First {#aiven-where-to-look-first}

> type: concept

Read an Aiven configuration in this order:

1. `required_providers` and the lock file. The version decides which resources use which library, and which IMPORTANT changelog notes apply.
2. The `provider "aiven"` block and the token behind it. The token decides which projects Terraform can reach and whether it can read secrets.
3. The `aiven_kafka` and `aiven_kafka_connect` blocks. Find where `project` and `service_name` come from, because they are the identity. Then read `plan`, `termination_protection`, `lifecycle` and `kafka_user_config`.
4. The `for_each` keys of topics, ACLs and connectors. A changed key without a `moved` block plans a destroy and a create.
5. The topics. Check `partitions`, `replication` and the `config` keys. Note topics that other tools also create, such as dead-letter topics of connectors.
6. The ACLs. Check how far each wildcard reaches, and which users exist outside Terraform.
7. The schemas and the registry-wide compatibility level.
8. The connectors. Check the target service, secrets in `config`, `depends_on` on ACLs, and plugin version pins.
9. The integrations and endpoints, and the outputs that other code reads.

### Plan Signals {#aiven-plan-signals}

> type: concept

| Line in the plan | Meaning |
| --- | --- |
| `-/+ aiven_kafka ... # forces replacement` | The service and all its data will be deleted, and every child object replaced. |
| `-/+ aiven_kafka_topic ... # forces replacement` | The topic and its messages will be deleted. |
| `number of partitions cannot be decreased` | The plan stops; Kafka cannot remove partitions. |
| `topic conflict, already exists` | A topic with this name exists outside this resource: import it, or add a `moved` block. |
| `schema is not compatible with previous version` | The schema edit breaks the compatibility rule. |
| `~` on a connector `config` without a code change | Someone changed the connector outside Terraform; the apply will revert it. |
| `~` on integrations right after a provider upgrade | A resource moved to the Plugin Framework; see the changelog's IMPORTANT notes. |

## Pitfalls {#aiven-pitfalls}

> type: category

The pitfalls fall into three groups: data loss, connectors, and operations.

### Data Loss Pitfalls {#aiven-data-loss-pitfalls}

> type: concept

These pitfalls delete messages. A plan shows each of them, so they are caught when someone reads the plan.

#### Service Replacement {#service-replacement}

> type: warning

A new `service_name` or `project`, for example from a typo in an input file, replaces the Kafka service. Every topic, ACL and connector inside it is replaced too. `termination_protection` makes the delete fail in Aiven, and `lifecycle { prevent_destroy = true }` makes the plan fail before anything runs. The old topic-level `termination_protection` is checked only by the provider, and only at apply ([issue 1596](https://github.com/aiven/terraform-provider-aiven/issues/1596)). Use `prevent_destroy` instead, as the topic docs advise ([topic docs](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka_topic.md#L94)).

#### Topic Identity {#topic-identity}

> type: warning

A rename in the code deletes the topic with its messages and creates an empty one. A changed `for_each` key without a `moved` block plans the same. When the new topic has the same name, the plan fails with "topic conflict, already exists". The same error appears when a tool such as Kafka Connect already created the topic, for example a dead-letter topic. In that case, import the topic.

#### Partition Changes {#partition-changes}

> type: warning

Raising `partitions` is an in-place change, but it changes where keys go. The producer picks the partition as `hash(key) % partitions` ([Kafka partitioner](https://github.com/apache/kafka/blob/4.1.0/clients/src/main/java/org/apache/kafka/clients/producer/internals/BuiltInPartitioner.java#L329-L330)), and Kafka does not move existing records ([Kafka operations](https://github.com/apache/kafka/blob/b87eb4f673075dc7bb410443689241fcb0302560/docs/operations/basic-kafka-operations.md#L59)). This table uses Kafka's own hash function, from About the Sources:

| Key | Partition with 3 partitions | Partition with 6 partitions |
| --- | --- | --- |
| `truck-17` | 2 | 2 |
| `truck-42` | 2 | 5 |
| `truck-88` | 1 | 4 |

After the change, new records for `truck-42` go to partition 5, while its older records stay in partition 2. A consumer that needs all events of one key in order can then read a newer event before an older one.

### Connector Pitfalls {#aiven-connector-pitfalls}

> type: concept

These pitfalls do not show as errors. The connector runs, but not as the code says.

#### Secrets in Connector Config {#secrets-in-connector-config}

> type: warning

The provider stopped marking connector `config` as sensitive in 4.2.0 ([changelog](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/CHANGELOG.md#L1399)). A plan prints each value that does not come from a sensitive variable, and the state stores every value in plain text. In Aiven, any user with `service:data:write` can read connector configurations, which "can contain secrets in plain text" ([permissions](https://aiven.io/docs/platform/concepts/permissions)). Kafka Connect secret providers keep the value out: the config holds a reference to AWS Secrets Manager instead ([secret providers](https://aiven.io/docs/products/kafka/kafka-connect/howto/configure-secret-providers)).

#### Connector Drift {#connector-drift}

> type: warning

An edit in the Aiven Console shows in the next plan, and the apply reverts it. For example, the console option for automatic restart is the key `_aiven.restart.on.failure` in the connector config ([provider example](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/examples/postgres/postgres_debezium_kafka/debezium_connector.tf#L24)); put it in the code to keep it. A change of `connector_name` or `service_name` creates a new connector. A sink connector reads with the consumer group `connect-<connector name>` ([Kafka source](https://github.com/apache/kafka/blob/4.1.0/connect/runtime/src/main/java/org/apache/kafka/connect/util/SinkUtils.java#L39-L41)). So a new name starts a new group with no saved offsets, and it reads the topic again from its configured start point.

#### Keys That Kafka Connect Ignores {#keys-that-connect-ignores}

> type: warning

The provider accepts any key, and Kafka's config classes only log unused keys at info level ([AbstractConfig](https://github.com/apache/kafka/blob/4.1.0/clients/src/main/java/org/apache/kafka/common/config/AbstractConfig.java#L386-L391)). So a misplaced setting does nothing, and no error says so. Two examples:

- A consumer setting for one connector needs the prefix `consumer.override.`, for example `consumer.override.max.poll.records` ([Kafka Connect user guide](https://github.com/apache/kafka/blob/b87eb4f673075dc7bb410443689241fcb0302560/docs/kafka-connect/user-guide.md#L57)).
- `offset.flush.interval.ms` is a worker setting. It is not a connector setting ([worker settings](https://kafka.apache.org/39/generated/connect_config.html)).

Aiven also says that `plugin_version` inside a connector config has no effect ([connector versions](https://aiven.io/docs/products/kafka/kafka-connect/howto/manage-connector-versions)).

#### Unpinned Plugin Versions {#unpinned-plugin-versions}

> type: warning

Without a pin, Aiven installs the latest connector plugin version during maintenance, and it can remove deprecated versions ([connector versions](https://aiven.io/docs/products/kafka/kafka-connect/howto/manage-connector-versions)). A connector can then change behavior with no code change. The pin lives on the service, not on the connector. A Connect service takes `plugin_versions` in `kafka_connect_user_config`. When Connect runs on the Kafka nodes, the Kafka service takes `kafka_connect_plugin_versions` in `kafka_user_config` ([Connect service doc](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka_connect.md#L125)). A new pin restarts Kafka Connect.

#### Connectors Before Their Integration {#connectors-before-integration}

> type: warning

A dedicated Connect service reaches Kafka through an `aiven_service_integration` of type `kafka_connect`. The provider's docs give the order: the Kafka service, the topics, the Connect service, the integration, and then the connectors ([setup order](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/internal/sdkprovider/service/kafka/kafka_connect.go#L16-L26)). A connector that refers only to the Connect service name has no dependency on the integration. Add `depends_on` on the integration, so Terraform keeps this order.

### Operations Pitfalls {#aiven-operations-pitfalls}

> type: concept

These pitfalls come from provider versions, schemas, access rules and service state.

#### Floating Provider Version {#floating-provider-version}

> type: warning

A constraint such as `>= 4.46.0` with no committed lock file installs each new release on the next run. Release 4.66.0, for example, moved integrations to the Plugin Framework, so their first plan can show removed blocks. It also fails the plan when one side of an integration sets both an endpoint ID and a service name ([4.66.0 notes](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/CHANGELOG.md#L15-L33)). Commit `.terraform.lock.hcl`, and raise the version on purpose.

#### Schema Registry Edits {#schema-registry-edits}

> type: warning

The plan checks a schema edit only against the version in the state. With a `*_TRANSITIVE` level, or with versions that producers registered themselves, the plan can pass and the apply can fail. Removing `compatibility_level` from a subject keeps its last value. Renaming the subject or changing `schema_type` deletes the subject softly and registers it again ([schema doc](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka_schema.md#L6)).

#### ACL Matching {#acl-matching}

> type: warning

In an Aiven ACL, `*` matches zero or more characters, dots included. So the topic pattern `orders.*` also matches `orders.dlq.retry`. Every field of an ACL forces a new resource, so an edit deletes the rule and then creates the new one, with a short gap in between ([ACL fields](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka_acl.md#L40-L44)). A new service also has a default ACL that gives `avnadmin` the `admin` right on all topics.

#### Powered-Off Services {#powered-off-services}

> type: warning

While a service is off, plans fail, because the provider first checks that the service is running ([service check](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/internal/schemautil/service.go#L1365-L1410)). A topic missing after a power cycle drops out of the state and is planned again ([topic doc](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka_topic.md#L10)). Aiven deletes a service that stays off for more than 180 days ([power off](https://aiven.io/docs/platform/concepts/service-power-cycle)).

#### Secrets the Token Cannot Read {#secrets-token-cannot-read}

> type: warning

The provider asks the Aiven API for secrets when it reads some resources, such as service users and integration endpoints. From 1 March 2027 the API redacts secrets unless the token's role may read them ([4.66.0 notes](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/CHANGELOG.md#L34-L39); [secret redaction](https://aiven.io/docs/tools/api/secret-redaction)). A token without such a role then makes these plans fail. The environment variable `AIVEN_INCLUDE_SECRETS=false` reads the resources without secrets.

## Good and Bad Decisions {#aiven-decisions}

> type: category

Each node compares a choice with its alternative.

### Dedicated or Shared Connect {#dedicated-or-shared-connect}

> type: concept

Kafka Connect can run on the nodes of the Kafka service or in a dedicated Connect service ([enable Kafka Connect](https://aiven.io/docs/products/kafka/kafka-connect/howto/enable-connect)). On the Kafka nodes it costs nothing extra, but it shares CPU and memory with the brokers and needs a business or premium plan. A dedicated service costs more and scales on its own. The trade-off is cost against the stability of the brokers. Moving a connector between the two recreates it, so choose before the first apply.

### Service Users in Code or by Hand {#service-users-in-code-or-by-hand}

> type: concept

`aiven_kafka_user` puts each service user in code, with drift detection. Its password and access key then sit in the state in plain text, and write-only arguments need Terraform 1.11 or later ([storing passwords](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/guides/storing-passwords.md)). Users made by hand keep secrets out of the state, but nothing checks them, and the ACLs refer to names that Terraform cannot see. Either way, certificate renewal is a manual step ([renew certificates](https://aiven.io/docs/products/kafka/howto/renew-ssl-certs)).

### Secret References over Secret Values {#secret-references}

> type: concept

A connector config can hold a secret value, or a reference that Kafka Connect resolves at run time from AWS Secrets Manager or from environment variables ([secret providers](https://aiven.io/docs/products/kafka/kafka-connect/howto/configure-secret-providers)). The reference keeps the secret out of the state, the plan and the connector API. It costs one more setup step on the Connect service.

### Pin Everything That Moves {#pin-everything-that-moves}

> type: concept

Three versions change behavior without a code change: the provider, the connector plugins and Karapace. Commit the lock file for the provider. Pin plugins with `plugin_versions` and Karapace with `karapace_version` ([Kafka resource](https://github.com/aiven/terraform-provider-aiven/blob/v4.66.0/docs/resources/kafka.md#L134)). Then raise each pin on purpose, and read the release notes first.

### One Project per Environment {#one-project-per-environment}

> type: concept

Aiven recommends separate projects for environments such as development and production ([organizations and projects](https://aiven.io/docs/platform/concepts/orgs-units-projects)). It also recommends one application user, with its own token, for each tool ([application users](https://aiven.io/docs/platform/concepts/application-users)). Then a token for development cannot touch production, and each token can be rotated alone. One shared token for all environments is simpler, but it gives every pipeline the rights of all of them.

## About the Sources {#aiven-about-the-sources}

> type: comment

The provider facts come from the `aiven/aiven` repository at tag v4.66.0, the latest release on 2026-11-23. The Aiven facts come from aiven.io/docs on the same day.

The partition table comes from this Python script. It copies Kafka's `murmur2` function and its default partitioner, and it checks itself against three test values from Kafka's own tests ([UtilsTest](https://github.com/apache/kafka/blob/4.1.0/clients/src/test/java/org/apache/kafka/common/utils/UtilsTest.java#L106-L118)):

```python
def murmur2(data: bytes) -> int:
    length = len(data)
    m, r = 0x5BD1E995, 24
    h = (0x9747B28C ^ length) & 0xFFFFFFFF
    for i in range(length // 4):
        i4 = i * 4
        k = data[i4] | data[i4 + 1] << 8 | data[i4 + 2] << 16 | data[i4 + 3] << 24
        k = (k * m) & 0xFFFFFFFF
        k ^= k >> r
        k = (k * m) & 0xFFFFFFFF
        h = (h * m) & 0xFFFFFFFF
        h ^= k
    rest = length % 4
    if rest == 3:
        h ^= data[(length & ~3) + 2] << 16
    if rest >= 2:
        h ^= data[(length & ~3) + 1] << 8
    if rest >= 1:
        h ^= data[length & ~3]
        h = (h * m) & 0xFFFFFFFF
    h ^= h >> 13
    h = (h * m) & 0xFFFFFFFF
    h ^= h >> 15
    return h - (1 << 32) if h & 0x80000000 else h


def partition(key: str, partitions: int) -> int:
    return (murmur2(key.encode()) & 0x7FFFFFFF) % partitions


assert murmur2(b"21") == -973932308
assert murmur2(b"foobar") == -790332482
assert murmur2(b"a-little-bit-long-string") == -985981536
for key in ["truck-17", "truck-42", "truck-88"]:
    print(key, partition(key, 3), partition(key, 6))
```
