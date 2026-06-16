> **This page is auto-published to Confluence on each tag push.** Do not edit the Confluence page directly — changes will be overwritten. Make changes in the GitHub repository.

# terraform-aws-elasticsearch-cloudwatch-sns-alarms

**GitHub:** [elocalorg/terraform-aws-elasticsearch-cloudwatch-sns-alarms](https://github.com/elocalorg/terraform-aws-elasticsearch-cloudwatch-sns-alarms)

## Status

> ⚠️ **Deprecated** — superseded by [terraform-aws-opensearch-alarms](https://github.com/elocalorg/terraform-aws-opensearch-alarms). Amazon Elasticsearch Service is end-of-life; AWS migrated all remaining clusters to OpenSearch. No new features will be added; critical fixes only. Existing callers should migrate to the opensearch-alarms module.

## Overview

Creates CloudWatch metric alarms for an Amazon Elasticsearch Service (AWS/ES) domain and routes all alert and recovery notifications to an SNS topic. By default the module creates the SNS topic for you; pass an existing topic ARN if one already exists. The module monitors cluster health, storage, CPU, JVM memory, snapshot failures, shard count, and optionally KMS key health and dedicated-master node metrics.

## Use this module instead of

Do not use the following native resources directly. Use this module instead:

* `aws_cloudwatch_metric_alarm` — all CloudWatch alarms for an Elasticsearch domain are managed by this module; defining them separately will conflict with the alarm names this module creates
* `aws_sns_topic` — managed internally when `create_sns_topic = true` (the default); pass an existing ARN via `sns_topic` and set `create_sns_topic = false` to reuse an existing topic
* `aws_sns_topic_policy` — managed internally alongside the SNS topic; do not define a separate policy for the topic this module creates

## What this module enforces

* **Both alarm and OK actions route to SNS** — every alarm sends notifications on threshold breach and on recovery; this cannot be disabled per alarm
* **Alarm names are scoped to the domain** — all alarm names include the `alarm_name_prefix` and `alarm_name_postfix` variables so multiple domains in the same account do not collide
* **Per-node storage uses `Minimum` statistic** — `FreeStorageSpace` uses `Minimum` across all nodes, matching AWS recommendations so the alarm fires when any single node is low, not just the average
* **Most alarms enabled by default** — cluster status red/yellow, free storage, writes blocked, node count, snapshot failure, CPU, and JVM memory are all on by default; KMS and dedicated-master alarms are off by default and must be explicitly enabled
* **Shard alarm uses `GreaterThanOrEqualToThreshold` on allocated shard count** — triggers when the domain approaches the per-cluster shard limit

## Usage

```hcl
module "es_alarms" {
  source  = "app.terraform.io/elocal/elasticsearch-cloudwatch-sns-alarms/aws"
  version = "~> 1.0"

  domain_name = "my-es-domain"
}
```

To reuse an existing SNS topic instead of creating one:

```hcl
module "es_alarms" {
  source  = "app.terraform.io/elocal/elasticsearch-cloudwatch-sns-alarms/aws"
  version = "~> 1.0"

  domain_name      = "my-es-domain"
  sns_topic        = "arn:aws:sns:us-east-1:123456789012:my-existing-topic"
  create_sns_topic = false
}
```

## Configuration guide

### SNS topic

By default the module creates a new SNS topic named with a timestamp suffix. Set `sns_topic_prefix` and `sns_topic_postfix` to control naming. To reuse an existing topic, set `create_sns_topic = false` and pass the full ARN as `sns_topic`.

### Alarm sensitivity (periods and evaluation counts)

Each alarm has a `_period` variable (seconds per datapoint) and a `_periods` variable (number of consecutive datapoints that must breach the threshold before alerting). Increase `_periods` to reduce noise for alarms that fire transiently. CPU and master CPU default to 3 evaluation periods; most others default to 1.

### Shard monitoring

`monitor_allocated_shards_too_high` is enabled by default and alerts when `AllocatedShards` exceeds `available_shards_threshold` (default 5400, tuned for a 6-node cluster at 90% of the 6000-shard limit). Set `available_shards_threshold` to `0.9 * max_shards_per_cluster * node_count` for your cluster size. Also set `max_available_shards` to a non-zero value to enable the upper-bound check.

### Dedicated master node alarms

`monitor_master_cpu_utilization_too_high` and `monitor_master_jvm_memory_pressure_too_high` are disabled by default. Enable them only when the domain has dedicated master nodes configured, otherwise CloudWatch will emit no data for these metrics and the alarm will not behave as expected.

### KMS alarms

`monitor_kms` is disabled by default. Enable it only when the Elasticsearch domain is configured with a KMS customer-managed key for encryption at rest. When enabled, alarms fire if the key is disabled or its grants are revoked.

### Storage thresholds

`free_storage_space_threshold` defaults to 20480 MiB (20 GiB) per node. Set it based on the node's EBS volume size; AWS recommends keeping at least 20% free. `monitor_free_storage_space_total_too_low` is disabled by default; enable it for multi-node clusters and set `free_storage_space_total_threshold` to `free_storage_space_threshold * node_count`.

## What this module does not support

* **OpenSearch domains** — this module targets the `AWS/ES` CloudWatch namespace; for OpenSearch Service domains use [terraform-aws-opensearch-alarms](https://github.com/elocalorg/terraform-aws-opensearch-alarms) instead
* **Alerting integrations beyond SNS** — routing to PagerDuty, Slack, or other destinations must be wired up at the SNS subscription level outside this module
* **CloudWatch dashboard creation** — metrics are only alarmed, not visualised; create dashboards separately

## Version history

### 1.0.4 — Fix shard monitor alarm math

_June 2023_

* Corrects the `AllocatedShards` alarm threshold calculation and alarm description

### 1.0.3 — Typo fix

_June 2023_

* Minor description typo fix

### 1.0.2 — Typo fix

_June 2023_

* Minor description typo fix

### 1.0.1 — Add shard monitoring

_June 2023_

* Initial elocal release; extends upstream with `AllocatedShards` CloudWatch alarm (PEV-2723)

---

## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 0.12 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_domain_name"></a> [domain\_name](#input\_domain\_name) | The Elasticsearch domain name you want to monitor | `string` | n/a | yes |
| <a name="input_alarm_name_prefix"></a> [alarm\_name\_prefix](#input\_alarm\_name\_prefix) | Alarm name prefix, used in the naming of alarms created | `string` | `""` | no |
| <a name="input_alarm_name_postfix"></a> [alarm\_name\_postfix](#input\_alarm\_name\_postfix) | Alarm name suffix, used in the naming of alarms created | `string` | `""` | no |
| <a name="input_tags"></a> [tags](#input\_tags) | A map of tags to add to all resources | `map(string)` | `{}` | no |
| <a name="input_create_sns_topic"></a> [create\_sns\_topic](#input\_create\_sns\_topic) | If false, uses `sns_topic` directly instead of creating a new topic | `bool` | `true` | no |
| <a name="input_sns_topic"></a> [sns\_topic](#input\_sns\_topic) | SNS topic ARN. Required when `create_sns_topic = false`. Ignored when creating a new topic. | `string` | `""` | no |
| <a name="input_sns_topic_prefix"></a> [sns\_topic\_prefix](#input\_sns\_topic\_prefix) | SNS topic name prefix; only used when creating a new topic | `string` | `""` | no |
| <a name="input_sns_topic_postfix"></a> [sns\_topic\_postfix](#input\_sns\_topic\_postfix) | SNS topic name suffix; only used when creating a new topic | `string` | `""` | no |
| <a name="input_monitor_cluster_status_is_red"></a> [monitor\_cluster\_status\_is\_red](#input\_monitor\_cluster\_status\_is\_red) | Enable monitoring of cluster status is in red | `bool` | `true` | no |
| <a name="input_monitor_cluster_status_is_yellow"></a> [monitor\_cluster\_status\_is\_yellow](#input\_monitor\_cluster\_status\_is\_yellow) | Enable monitoring of cluster status is in yellow | `bool` | `true` | no |
| <a name="input_monitor_free_storage_space_too_low"></a> [monitor\_free\_storage\_space\_too\_low](#input\_monitor\_free\_storage\_space\_too\_low) | Enable monitoring of per-node free storage space | `bool` | `true` | no |
| <a name="input_monitor_free_storage_space_total_too_low"></a> [monitor\_free\_storage\_space\_total\_too\_low](#input\_monitor\_free\_storage\_space\_total\_too\_low) | Enable monitoring of cluster total free storage space | `bool` | `false` | no |
| <a name="input_monitor_cluster_index_writes_blocked"></a> [monitor\_cluster\_index\_writes\_blocked](#input\_monitor\_cluster\_index\_writes\_blocked) | Enable monitoring of cluster index writes being blocked | `bool` | `true` | no |
| <a name="input_monitor_min_available_nodes"></a> [monitor\_min\_available\_nodes](#input\_monitor\_min\_available\_nodes) | Enable monitoring of minimum available nodes | `bool` | `true` | no |
| <a name="input_monitor_automated_snapshot_failure"></a> [monitor\_automated\_snapshot\_failure](#input\_monitor\_automated\_snapshot\_failure) | Enable monitoring of automated snapshot failure | `bool` | `true` | no |
| <a name="input_monitor_cpu_utilization_too_high"></a> [monitor\_cpu\_utilization\_too\_high](#input\_monitor\_cpu\_utilization\_too\_high) | Enable monitoring of CPU utilization | `bool` | `true` | no |
| <a name="input_monitor_jvm_memory_pressure_too_high"></a> [monitor\_jvm\_memory\_pressure\_too\_high](#input\_monitor\_jvm\_memory\_pressure\_too\_high) | Enable monitoring of JVM memory pressure | `bool` | `true` | no |
| <a name="input_monitor_kms"></a> [monitor\_kms](#input\_monitor\_kms) | Enable monitoring of KMS-related metrics; only enable when using KMS with Elasticsearch | `bool` | `false` | no |
| <a name="input_monitor_master_cpu_utilization_too_high"></a> [monitor\_master\_cpu\_utilization\_too\_high](#input\_monitor\_master\_cpu\_utilization\_too\_high) | Enable monitoring of dedicated master node CPU utilization | `bool` | `false` | no |
| <a name="input_monitor_master_jvm_memory_pressure_too_high"></a> [monitor\_master\_jvm\_memory\_pressure\_too\_high](#input\_monitor\_master\_jvm\_memory\_pressure\_too\_high) | Enable monitoring of dedicated master node JVM memory pressure | `bool` | `false` | no |
| <a name="input_monitor_allocated_shards_too_high"></a> [monitor\_allocated\_shards\_too\_high](#input\_monitor\_allocated\_shards\_too\_high) | Enable monitoring of per-cluster allocated shard count | `bool` | `true` | no |
| <a name="input_cpu_utilization_threshold"></a> [cpu\_utilization\_threshold](#input\_cpu\_utilization\_threshold) | Maximum percentage of CPU utilization before alarming | `number` | `80` | no |
| <a name="input_jvm_memory_pressure_threshold"></a> [jvm\_memory\_pressure\_threshold](#input\_jvm\_memory\_pressure\_threshold) | Maximum percentage of Java heap used for data nodes before alarming | `number` | `80` | no |
| <a name="input_master_cpu_utilization_threshold"></a> [master\_cpu\_utilization\_threshold](#input\_master\_cpu\_utilization\_threshold) | Maximum percentage of CPU utilization for master nodes before alarming | `number` | `80` | no |
| <a name="input_master_jvm_memory_pressure_threshold"></a> [master\_jvm\_memory\_pressure\_threshold](#input\_master\_jvm\_memory\_pressure\_threshold) | Maximum percentage of Java heap used for master nodes before alarming | `number` | `80` | no |
| <a name="input_free_storage_space_threshold"></a> [free\_storage\_space\_threshold](#input\_free\_storage\_space\_threshold) | Minimum free storage space per node in MiB before alarming | `number` | `20480` | no |
| <a name="input_free_storage_space_total_threshold"></a> [free\_storage\_space\_total\_threshold](#input\_free\_storage\_space\_total\_threshold) | Minimum total free storage space across the cluster in MiB before alarming | `number` | `20480` | no |
| <a name="input_min_available_nodes"></a> [min\_available\_nodes](#input\_min\_available\_nodes) | Minimum reachable nodes; set to non-zero to enable the node count alarm | `number` | `0` | no |
| <a name="input_available_shards_threshold"></a> [available\_shards\_threshold](#input\_available\_shards\_threshold) | Maximum allocated shards before alarming; default tuned for a 6-node cluster at 90% of the 6000-shard limit | `number` | `5400` | no |
| <a name="input_max_available_shards"></a> [max\_available\_shards](#input\_max\_available\_shards) | Maximum available shards per cluster; set to non-zero to enable the upper-bound check | `number` | `0` | no |
| <a name="input_alarm_cluster_status_is_red_period"></a> [alarm\_cluster\_status\_is\_red\_period](#input\_alarm\_cluster\_status\_is\_red\_period) | Period in seconds for cluster status red alarm | `number` | `60` | no |
| <a name="input_alarm_cluster_status_is_red_periods"></a> [alarm\_cluster\_status\_is\_red\_periods](#input\_alarm\_cluster\_status\_is\_red\_periods) | Number of consecutive periods before triggering cluster status red alarm | `number` | `1` | no |
| <a name="input_alarm_cluster_status_is_yellow_period"></a> [alarm\_cluster\_status\_is\_yellow\_period](#input\_alarm\_cluster\_status\_is\_yellow\_period) | Period in seconds for cluster status yellow alarm | `number` | `60` | no |
| <a name="input_alarm_cluster_status_is_yellow_periods"></a> [alarm\_cluster\_status\_is\_yellow\_periods](#input\_alarm\_cluster\_status\_is\_yellow\_periods) | Number of consecutive periods before triggering cluster status yellow alarm | `number` | `1` | no |
| <a name="input_alarm_free_storage_space_too_low_period"></a> [alarm\_free\_storage\_space\_too\_low\_period](#input\_alarm\_free\_storage\_space\_too\_low\_period) | Period in seconds for per-node free storage alarm | `number` | `60` | no |
| <a name="input_alarm_free_storage_space_too_low_periods"></a> [alarm\_free\_storage\_space\_too\_low\_periods](#input\_alarm\_free\_storage\_space\_too\_low\_periods) | Number of consecutive periods before triggering per-node free storage alarm | `number` | `1` | no |
| <a name="input_alarm_free_storage_space_total_too_low_period"></a> [alarm\_free\_storage\_space\_total\_too\_low\_period](#input\_alarm\_free\_storage\_space\_total\_too\_low\_period) | Period in seconds for total cluster free storage alarm | `number` | `60` | no |
| <a name="input_alarm_free_storage_space_total_too_low_periods"></a> [alarm\_free\_storage\_space\_total\_too\_low\_periods](#input\_alarm\_free\_storage\_space\_total\_too\_low\_periods) | Number of consecutive periods before triggering total cluster free storage alarm | `number` | `1` | no |
| <a name="input_alarm_cluster_index_writes_blocked_period"></a> [alarm\_cluster\_index\_writes\_blocked\_period](#input\_alarm\_cluster\_index\_writes\_blocked\_period) | Period in seconds for writes blocked alarm | `number` | `300` | no |
| <a name="input_alarm_cluster_index_writes_blocked_periods"></a> [alarm\_cluster\_index\_writes\_blocked\_periods](#input\_alarm\_cluster\_index\_writes\_blocked\_periods) | Number of consecutive periods before triggering writes blocked alarm | `number` | `1` | no |
| <a name="input_alarm_min_available_nodes_period"></a> [alarm\_min\_available\_nodes\_period](#input\_alarm\_min\_available\_nodes\_period) | Period in seconds for minimum available nodes alarm | `number` | `86400` | no |
| <a name="input_alarm_min_available_nodes_periods"></a> [alarm\_min\_available\_nodes\_periods](#input\_alarm\_min\_available\_nodes\_periods) | Number of consecutive periods before triggering minimum available nodes alarm | `number` | `1` | no |
| <a name="input_alarm_automated_snapshot_failure_period"></a> [alarm\_automated\_snapshot\_failure\_period](#input\_alarm\_automated\_snapshot\_failure\_period) | Period in seconds for automated snapshot failure alarm | `number` | `60` | no |
| <a name="input_alarm_automated_snapshot_failure_periods"></a> [alarm\_automated\_snapshot\_failure\_periods](#input\_alarm\_automated\_snapshot\_failure\_periods) | Number of consecutive periods before triggering snapshot failure alarm | `number` | `1` | no |
| <a name="input_alarm_cpu_utilization_too_high_period"></a> [alarm\_cpu\_utilization\_too\_high\_period](#input\_alarm\_cpu\_utilization\_too\_high\_period) | Period in seconds for CPU utilization alarm | `number` | `900` | no |
| <a name="input_alarm_cpu_utilization_too_high_periods"></a> [alarm\_cpu\_utilization\_too\_high\_periods](#input\_alarm\_cpu\_utilization\_too\_high\_periods) | Number of consecutive periods before triggering CPU utilization alarm | `number` | `3` | no |
| <a name="input_alarm_jvm_memory_pressure_too_high_period"></a> [alarm\_jvm\_memory\_pressure\_too\_high\_period](#input\_alarm\_jvm\_memory\_pressure\_too\_high\_period) | Period in seconds for JVM memory pressure alarm | `number` | `900` | no |
| <a name="input_alarm_jvm_memory_pressure_too_high_periods"></a> [alarm\_jvm\_memory\_pressure\_too\_high\_periods](#input\_alarm\_jvm\_memory\_pressure\_too\_high\_periods) | Number of consecutive periods before triggering JVM memory pressure alarm | `number` | `1` | no |
| <a name="input_alarm_kms_period"></a> [alarm\_kms\_period](#input\_alarm\_kms\_period) | Period in seconds for KMS alarms | `number` | `60` | no |
| <a name="input_alarm_kms_periods"></a> [alarm\_kms\_periods](#input\_alarm\_kms\_periods) | Number of consecutive periods before triggering KMS alarms | `number` | `1` | no |
| <a name="input_alarm_master_cpu_utilization_too_high_period"></a> [alarm\_master\_cpu\_utilization\_too\_high\_period](#input\_alarm\_master\_cpu\_utilization\_too\_high\_period) | Period in seconds for master node CPU utilization alarm | `number` | `900` | no |
| <a name="input_alarm_master_cpu_utilization_too_high_periods"></a> [alarm\_master\_cpu\_utilization\_too\_high\_periods](#input\_alarm\_master\_cpu\_utilization\_too\_high\_periods) | Number of consecutive periods before triggering master node CPU utilization alarm | `number` | `3` | no |
| <a name="input_alarm_master_jvm_memory_pressure_too_high_period"></a> [alarm\_master\_jvm\_memory\_pressure\_too\_high\_period](#input\_alarm\_master\_jvm\_memory\_pressure\_too\_high\_period) | Period in seconds for master node JVM memory pressure alarm | `number` | `900` | no |
| <a name="input_alarm_master_jvm_memory_pressure_too_high_periods"></a> [alarm\_master\_jvm\_memory\_pressure\_too\_high\_periods](#input\_alarm\_master\_jvm\_memory\_pressure\_too\_high\_periods) | Number of consecutive periods before triggering master node JVM memory pressure alarm | `number` | `1` | no |
| <a name="input_alarm_allocated_shards_too_high_period"></a> [alarm\_allocated\_shards\_too\_high\_period](#input\_alarm\_allocated\_shards\_too\_high\_period) | Period in seconds for allocated shards alarm | `number` | `900` | no |
| <a name="input_alarm_allocated_shards_too_high_periods"></a> [alarm\_allocated\_shards\_too\_high\_periods](#input\_alarm\_allocated\_shards\_too\_high\_periods) | Number of consecutive periods before triggering allocated shards alarm | `number` | `1` | no |

## Outputs

| Name | Description |
|------|-------------|
| <a name="output_sns_topic_arn"></a> [sns\_topic\_arn](#output\_sns\_topic\_arn) | The ARN of the SNS topic |
| <a name="output_sns_topic_name"></a> [sns\_topic\_name](#output\_sns\_topic\_name) | The SNS topic name |

## Resources

| Name | Type |
|------|------|
| [aws_cloudwatch_metric_alarm.allocated_shards_too_high](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.automated_snapshot_failure](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.cluster_index_writes_blocked](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.cluster_status_is_red](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.cluster_status_is_yellow](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.cpu_utilization_too_high](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.free_storage_space_too_low](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.free_storage_space_total_too_low](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.jvm_memory_pressure_too_high](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.kms_key_error](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.kms_key_inaccessible](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.master_cpu_utilization_too_high](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.master_jvm_memory_pressure_too_high](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_cloudwatch_metric_alarm.min_available_nodes](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) | resource |
| [aws_iam_policy_document.sns_topic_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document) | data source |
| [aws_sns_topic.default](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sns_topic) | resource |
| [aws_sns_topic.default_prefix](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sns_topic) | resource |
| [aws_sns_topic_policy.default](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/sns_topic_policy) | resource |
| [aws_caller_identity.default](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/caller_identity) | data source |
