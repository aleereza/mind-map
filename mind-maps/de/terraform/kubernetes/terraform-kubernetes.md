# Terraform for Kubernetes {#terraform-kubernetes}

> type: category

Kubernetes is a system that runs containers on a group of machines, called a cluster. You describe objects, such as a Deployment that runs copies of a container or a Service that gives them one address. The cluster then keeps the real state equal to the description. On AWS, Amazon EKS (Elastic Kubernetes Service) runs the cluster's control plane for you. The [Terraform map](../terraform.md) explains Terraform itself.

Terraform reaches a cluster on EKS through four providers:

- `hashicorp/aws` builds the cluster itself: the control plane, the node groups, add-ons and access rules.
- `hashicorp/kubernetes` creates Kubernetes objects inside the cluster, one typed resource per kind, plus `kubernetes_manifest` for any kind ([provider docs](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs)).
- `hashicorp/helm` installs Helm charts. A chart is a package of Kubernetes objects with settings, called values; one installed chart is a release ([provider docs](https://registry.terraform.io/providers/hashicorp/helm/latest/docs)).
- A community `kubectl` provider applies raw YAML files, as the `kubectl apply` command does.

The unit is one cluster. The last three providers need the cluster's address, its certificate and a short-lived token. They get them from the cluster that the first provider built, which creates the central problem of this setup ([Two APIs, One Depends on the Other](#two-apis-one-depends-on-the-other-two-apis-one-depends-on-the-other)).

Five more ideas explain most plans:

- Three records describe each release: the Terraform state, Helm's own record, and the live objects ([Three Records of Truth](#three-records-of-truth-three-records-of-truth)).
- Each field of a live object has one owner: Terraform, Helm, a controller or EKS ([Every Field Has One Owner](#every-field-has-one-owner-every-field-has-one-owner)).
- Custom resource definitions (CRDs) add new kinds to a cluster, and they have their own lifecycle ([CRDs Have Their Own Lifecycle](#crds-have-their-own-lifecycle-crds-have-their-own-lifecycle)).
- A finished apply does not mean a healthy cluster ([Apply Complete Is Not Healthy](#apply-complete-is-not-healthy-apply-complete-is-not-healthy)).
- Identity crosses between AWS and Kubernetes in two directions ([Two Identity Bridges](#two-identity-bridges-two-identity-bridges)).

Now compare Snowflake. Snowflake runs a statement once, and the object stays as created. Kubernetes runs control loops that keep changing objects towards their description, and some controllers add fields of their own. So in Kubernetes, more of what a plan shows depends on who else acted after the last apply.

The map has four parts. Mental Models explains the ideas above. Reading a Kubernetes Configuration says what you will see and where to look first. Pitfalls lists the failures that experienced users plan around. Good and Bad Decisions compares the main design choices.

## Mental Models {#k8s-mental-models}

> type: category

The models come in dependency order: each model uses only the models before it.

### Two APIs, One Depends on the Other {#two-apis-one-depends-on-the-other}

> type: concept

> edge.related: providers-from-same-apply

Terraform builds the cluster through the AWS API. Then it talks to the Kubernetes API with credentials that come from that same cluster. The Kubernetes provider warns that the cluster and the Kubernetes resources should not be created in the same module. Its docs call separate applies "the most reliable way" ([provider docs](https://github.com/hashicorp/terraform-provider-kubernetes/blob/v3.5.0/docs/index.md#L42-L44)). Its EKS example therefore uses two folders: one for the cluster and one for what runs inside ([EKS example](https://github.com/hashicorp/terraform-provider-kubernetes/blob/v3.5.0/_examples/eks/README.md)).

An EKS token is short-lived: AWS signs it for 15 minutes, and the AWS command-line tool marks it as expired after 14 ([get-token code](https://github.com/aws/aws-cli/blob/2.42.1/awscli/customizations/eks/get_token.py#L132-L136)). An `exec` block in the provider settings runs `aws eks get-token` for a fresh token on every run ([exec plugins](https://github.com/hashicorp/terraform-provider-kubernetes/blob/v3.5.0/docs/index.md#L108-L124)). So the machine that runs Terraform needs the AWS command-line tool.

### Three Records of Truth {#three-records-of-truth}

> type: concept

> edge.related: invisible-drift

A `helm_release` leaves three records:

1. The Terraform state holds the inputs: the chart, its version and its values.
2. Helm keeps its own record of each release, with the rendered objects, in a Secret in the cluster.
3. The live objects run in the cluster.

A `helm upgrade` or a `kubectl edit` outside Terraform changes the second or the third record, but not the first. So the plan can say "No changes" while the cluster differs from the code ([issue 372](https://github.com/hashicorp/terraform-provider-helm/issues/372)). The helm provider has an option that stores the rendered objects in the state, so a plan can compare them ([manifest experiment](https://github.com/hashicorp/terraform-provider-helm/blob/v3.3.0/docs/index.md#L201-L213)).

### Every Field Has One Owner {#every-field-has-one-owner}

> type: concept

> edge.related: controller-fields-force-replacement

Kubernetes records which client set each field of an object. With server-side apply, it reports a conflict when a second client tries to set the same field ([server-side apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/)). Controllers also set fields after Terraform creates an object. For example, the AWS Load Balancer Controller sets the class of a Service that asks for a load balancer. A diff that comes back after every apply means two owners write one field.

### CRDs Have Their Own Lifecycle {#crds-have-their-own-lifecycle}

> type: concept

> edge.related: helm-does-not-upgrade-crds

A custom resource definition adds a new kind to the cluster's API, such as `PrometheusRule` or `ExternalSecret`. A chart can bring its CRDs in a `crds` folder. Helm installs that folder once and never upgrades or deletes it: "There is no support at this time for upgrading or deleting CRDs using Helm" ([Helm CRD guide](https://github.com/helm/helm-www/blob/d869ccd3db838b63cee08bffb8400e1a9221c3ee/versioned_docs/version-3/chart_best_practices/custom_resource_definitions.md#L30-L45)). A custom resource, an object of such a kind, can only exist after its CRD exists. That creates an order between charts, and between charts and the resources that use their kinds.

### Apply Complete Is Not Healthy {#apply-complete-is-not-healthy}

> type: concept

Terraform reports success when the API accepts the objects. Whether the pods then run depends on images, resources and configuration. `helm_release` waits for its objects by default, for up to 300 seconds per operation ([release doc](https://github.com/hashicorp/terraform-provider-helm/blob/v3.3.0/docs/resources/release.md#L59-L64)). Setting `wait = false` makes applies faster, but then a broken release also reports success. The `kubectl_manifest` resource waits only for a Deployment to roll out ([kubectl doc](https://github.com/gavinbunney/terraform-provider-kubectl/blob/v1.19.0/docs/resources/kubectl_manifest.md#L37)).

### Two Identity Bridges {#two-identity-bridges}

> type: concept

Identity crosses between AWS and Kubernetes in two directions:

- From AWS into Kubernetes: an IAM principal becomes a Kubernetes user. EKS does this with access entries, managed through the AWS API, or with the older `aws-auth` ConfigMap inside the cluster ([access to clusters](https://docs.aws.amazon.com/eks/latest/userguide/grant-k8s-access.html)).
- From a pod out to AWS: a pod gets an IAM role. EKS does this with IAM roles for service accounts (IRSA), or with EKS Pod Identity ([pod identities](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)).

Access entries are managed only through the AWS provider, so granting access needs no working connection to the cluster.

## Reading a Kubernetes Configuration {#reading-k8s-configuration}

> type: category

These nodes say what you will find in the files, in which order to read them, and what each kind of plan line means.

### What You Will See {#k8s-what-you-will-see}

> type: concept

| Resource | What it is | What to check |
| --- | --- | --- |
| `aws_eks_cluster` | the control plane | Kubernetes version, access mode, endpoint access |
| `aws_eks_node_group` | a group of worker machines | Machine image source, sizes, ignored fields |
| `aws_eks_addon` | an EKS add-on, such as the network plugin | Pinned version per Kubernetes version |
| `aws_eks_access_entry` | IAM principal mapped to Kubernetes access | Who gets cluster admin |
| `helm_release` | one installed chart | `version`, values source, `wait`, `atomic`, `timeout` |
| `kubernetes_deployment`, `kubernetes_config_map`, and other typed resources | one object of a known kind | Namespace, labels, restarts on config change |
| `kubernetes_manifest` | any object, including custom resources | The CRD it needs at plan time |
| `kubectl_manifest` | raw YAML, from a community provider | Which fork, and server-side apply |
| `kubernetes_secret` | a Secret | Its data sits in the state in plain text |

A chart can also be configured from a template file with `templatefile`, which renders the values from variables. The rendered values then appear in the plan as one block of text.

### Where to Look First {#k8s-where-to-look-first}

> type: concept

Read a Kubernetes configuration in this order:

1. `required_providers`, the lock file and the Terraform version pin. The helm provider changed its syntax in version 3, so the version tells you which docs apply ([v3 upgrade guide](https://github.com/hashicorp/terraform-provider-helm/blob/v3.3.0/docs/guides/v3-upgrade-guide.md)).
2. The `provider "kubernetes"` and `provider "helm"` blocks. Find where the host, certificate and token come from. If the cluster is built in the same configuration, look for a documented bootstrap order.
3. The backend and how environments switch: workspaces, variable files or backend settings.
4. The pipeline: plan and apply, saved plan or not, approvals, `-target` steps and secret injection.
5. The cluster definition: version, access mode, access entries, add-on pins, endpoint access and node groups.
6. Each `helm_release`: the version pin, `wait`, `atomic`, `timeout`, CRD handling and secrets in values.
7. Each custom resource: which CRD it needs, which chart installs that CRD, and which resource type applies it.
8. The secret paths: `kubernetes_secret`, Helm values, and an operator that reads a secret store.
9. The `lifecycle` blocks, `ignore_changes`, and checksum annotations. Each one marks a problem someone already met.

### Plan Signals {#k8s-plan-signals}

> type: concept

| Line in the plan | Meaning |
| --- | --- |
| Errors such as `Failed to construct REST client` on every Kubernetes resource | The provider settings depend on a cluster that the plan will create or replace. |
| `no matches for kind "PrometheusRule"` | A `kubernetes_manifest` needs a CRD that is not installed yet. |
| `metadata` of a `helm_release` as `(known after apply)` | Normal for any change to the chart or its values ([release code](https://github.com/hashicorp/terraform-provider-helm/blob/v2.13.0/helm/resource_release.go#L826-L843)). |
| `-/+` on a Service of type LoadBalancer in every apply | A controller set a field that cannot change; the load balancer gets a new address each time. |
| `(sensitive value)` in place of all Helm values | One sensitive input made the whole values text sensitive. |

## Pitfalls {#k8s-pitfalls}

> type: category

The pitfalls fall into three groups: bootstrap, Helm, and the cluster.

### Bootstrap Pitfalls {#k8s-bootstrap-pitfalls}

> type: concept

These pitfalls appear when a cluster is new or replaced.

#### Providers from the Same Apply {#providers-from-same-apply}

> type: warning

When the provider settings read the cluster's address from resources in the same configuration, a plan for a new or replaced cluster has no address yet. Every Kubernetes resource then fails, for example with "Failed to construct REST client" ([issue 1775](https://github.com/hashicorp/terraform-provider-kubernetes/issues/1775)), or calls `localhost`. Separate states for the cluster and for what runs inside it fix this. A documented bootstrap with `-target` works too, but Terraform reserves `-target` for exceptional cases ([plan command](https://developer.hashicorp.com/terraform/cli/commands/plan)).

#### CRDs Needed at Plan Time {#crds-needed-at-plan-time}

> type: warning

`kubernetes_manifest` reads the API schema of its kind during the plan, so the cluster must be reachable and the CRD must exist ([manifest doc](https://github.com/hashicorp/terraform-provider-kubernetes/blob/v3.5.0/docs/resources/manifest.md#L82)). A custom resource planned in the same run as the chart that installs its CRD fails with `no matches for kind`. There are three ways out:

- Install the CRDs in an earlier apply.
- Use `kubectl_manifest`, which looks up the kind only at apply time ([kubectl code](https://github.com/gavinbunney/terraform-provider-kubectl/blob/v1.19.0/kubernetes/resource_kubectl_manifest.go#L218-L245)).
- Put the custom resources into a small chart of their own.

### Helm Pitfalls {#k8s-helm-pitfalls}

> type: concept

These pitfalls come from the gap between Terraform and Helm.

#### Helm Does Not Upgrade CRDs {#helm-does-not-upgrade-crds}

> type: warning

A chart upgrade that brings new CRD versions in its `crds` folder installs the new chart, but the old CRDs stay. New fields of the custom resources then fail. The plan shows only a new `version`. Each chart documents its own way to upgrade CRDs, for example with `kubectl apply` of the chart's CRD files. Charts that render their CRDs as normal templates do upgrade them, but they can also delete them when the release is removed, which deletes every custom resource of that kind.

#### Failed and Stuck Releases {#failed-and-stuck-releases}

> type: warning

When an install fails, the release can stay in Helm's record. The next apply then fails with "cannot re-use a name that is still in use" ([issue 1886](https://github.com/hashicorp/terraform-provider-helm/issues/1886)). An apply that is killed mid-operation can leave the release in a `pending-*` state, and every later run fails with "another operation (install/upgrade/rollback) is in progress". Recovery needs `helm rollback` or `helm uninstall` by hand. `atomic = true` removes a failed install automatically; it defaults to `false` ([release doc](https://github.com/hashicorp/terraform-provider-helm/blob/v3.3.0/docs/resources/release.md#L25)).

#### Secrets in Helm Values {#secrets-in-helm-values}

> type: warning

Helm values are part of the release resource. A secret written into `values`, for example a webhook URL, can print in clear text in a plan when the release changes or is destroyed. Only values passed through `set_sensitive` stay hidden ([issue 1287](https://github.com/hashicorp/terraform-provider-helm/issues/1287)). The opposite problem also happens: when a values template reads one sensitive variable, the whole values text becomes sensitive, and reviewers see no diff at all. Pass secrets through `set_sensitive` or the write-only `set_wo`, or let an operator read them from a secret store at run time.

#### Invisible Drift {#invisible-drift}

> type: warning

Some changes never show in a plan: a chart that someone upgraded by hand, an object that someone edited, or an image tag `latest` that moved. Terraform compares only its inputs (see Three Records of Truth). Pin chart versions and image tags, and do not change Terraform-managed releases by hand.

### Cluster Pitfalls {#k8s-cluster-pitfalls}

> type: concept

These pitfalls concern objects, identity and versions of the cluster.

#### Controller Fields Force Replacement {#controller-fields-force-replacement}

> type: warning

The AWS Load Balancer Controller sets `spec.loadBalancerClass` when a Service of type LoadBalancer is created ([controller docs](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/v3.6.3/docs/deploy/installation.md#L12-L13)). That field cannot change, and the Kubernetes provider replaces the Service when it differs from the code. So every apply deletes and recreates the Service, and the load balancer gets a new address. Write the field into the configuration with the value that the controller sets.

#### Secrets in State {#k8s-secrets-in-state}

> type: warning

The data of a `kubernetes_secret` sits in the state in plain text ([secret doc](https://github.com/hashicorp/terraform-provider-kubernetes/blob/v3.5.0/docs/resources/secret_v1.md#L14)). The write-only argument `data_wo` keeps it out, with Terraform 1.11 or later. An operator that copies secrets from AWS Secrets Manager into the cluster keeps them out of Terraform entirely.

#### Access Through aws-auth {#access-through-aws-auth}

> type: warning

The `aws-auth` ConfigMap holds all IAM-to-Kubernetes mappings in one object, including the entries that let nodes join the cluster. A configuration that writes the whole object must keep those entries, or nodes lose access. EKS access entries replace the ConfigMap. The access mode moves one way only: from `CONFIG_MAP` to `API_AND_CONFIG_MAP` to `API` ([EKS module upgrade notes](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/v21.30.2/docs/UPGRADE-20.0.md#L221-L223)). Keep the node entries that EKS created until matching access entries exist ([migrating to access entries](https://docs.aws.amazon.com/eks/latest/userguide/migrating-access-entries.html)).

#### Version Upgrades {#k8s-version-upgrades}

> type: warning

EKS raises the control plane one minor version per update, and the nodes must catch up before the next step ([update a cluster](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)). Add-on versions belong to a Kubernetes version, so each version change also changes add-on pins. A node group that reads its machine image version from AWS's "recommended" parameter replaces its nodes whenever AWS publishes a new image, even during an unrelated change ([node group doc](https://github.com/hashicorp/terraform-provider-aws/blob/v6.73.0/website/docs/r/eks_node_group.html.markdown#L69-L77)). A node replacement drains pods, and it fails after 15 minutes if pods cannot be evicted ([node updates](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html)).

#### NetworkPolicies Need an Enforcer {#networkpolicies-need-an-enforcer}

> type: warning

A NetworkPolicy object has no effect unless the cluster's network plugin enforces it ([network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)). On EKS, the VPC CNI plugin enforces policies only when its `enableNetworkPolicy` setting is on ([configure network policy](https://docs.aws.amazon.com/eks/latest/userguide/cni-network-policy-configure.html)). Without it, Terraform creates the policies, the plan is clean, and all traffic still flows.

## Good and Bad Decisions {#k8s-decisions}

> type: category

Each node compares a choice with its alternative.

### Split the Cluster from Its Contents {#split-the-cluster}

> type: concept

The Kubernetes provider recommends separate applies for the cluster and for what runs inside it. Two states give reliable plans even when the cluster is new or replaced, at the cost of two pipelines and a data source that connects them. One state gives one plan per change, at the cost of a manual bootstrap with `-target` steps.

### Fresh Tokens with exec {#fresh-tokens-with-exec}

> type: concept

An `exec` block fetches a new token for each run, so a long apply does not fail when a token expires, and no token lands in the state. The provider docs forbid mixing `exec` with other credential settings ([exec plugins](https://github.com/hashicorp/terraform-provider-kubernetes/blob/v3.5.0/docs/index.md#L112)). The cost is the AWS command-line tool on every runner.

### Pin Every Version Axis {#pin-every-version-axis}

> type: concept

A cluster has many versions that move on their own: the control plane, the node image, the add-ons, the charts, the CRDs, the container images and the providers. Without `version`, `helm_release` installs the latest chart ([release doc](https://github.com/hashicorp/terraform-provider-helm/blob/v3.3.0/docs/resources/release.md#L63)). Pin each axis, commit the lock file, and raise each pin in its own reviewed change.

### Choose the Custom Resource Tool {#choose-the-custom-resource-tool}

> type: concept

| Tool | For | Against |
| --- | --- | --- |
| `kubernetes_manifest` | Maintained by HashiCorp, server-side apply, typed diff | Needs the CRD at plan time |
| `kubectl_manifest` | Plan does not need the CRD | Drift shows only as an opaque fingerprint; the original provider has stalled |
| A small chart of raw objects | Helm handles the order and the lifecycle | One more chart to maintain |

The original `gavinbunney/kubectl` provider is "functionally stalled", according to the fork `alekc/kubectl`, which supports a `moved` block to migrate ([fork ADR](https://github.com/alekc/terraform-provider-kubectl/blob/v3.0.2/docs/adr/0001-framework-migration-and-moved-blocks.md#L14-L22)).

### Atomic Releases with Real Timeouts {#atomic-releases}

> type: concept

`atomic = true` with a realistic `timeout` rolls a failed install back and reports the failure. `wait = false` makes applies shorter, but it reports broken releases as successes. Prefer the first for anything that other systems depend on.

## About the Sources {#k8s-about-the-sources}

> type: comment

The provider facts come from `hashicorp/kubernetes` at tag v3.5.0, `hashicorp/helm` at tags v3.3.0 and v2.13.0, `gavinbunney/kubectl` at tag v1.19.0 and `alekc/kubectl` at tag v3.0.2. The Kubernetes, Helm and EKS facts come from their official docs. All were read on 2026-11-23.
