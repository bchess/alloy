---
canonical: https://grafana.com/docs/alloy/latest/reference/components/prometheus/prometheus.exporter.cloudwatch/
aliases:
  - ../prometheus.exporter.cloudwatch/ # /docs/alloy/latest/reference/components/prometheus.exporter.cloudwatch/
description: Learn about prometheus.exporter.cloudwatch
title: prometheus.exporter.cloudwatch
---

# prometheus.exporter.cloudwatch

The `prometheus.exporter.cloudwatch` component
embeds [`yet-another-cloudwatch-exporter`](https://github.com/nerdswords/yet-another-cloudwatch-exporter), letting you
collect [Amazon CloudWatch metrics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html),
translate them to a prometheus-compatible format and remote write them.

This component lets you scrape CloudWatch metrics in a set of configurations we call _jobs_. There are
two kinds of jobs: [discovery][] and [static][].

[discovery]: #discovery-block
[static]: #static-block

## Authentication

{{< param "PRODUCT_NAME" >}} must be running in an environment with access to AWS. The exporter uses
the [AWS SDK for Go](https://aws.github.io/aws-sdk-go-v2/docs/getting-started/) and
provides authentication
via [AWS's default credential chain](https://aws.github.io/aws-sdk-go-v2/docs/configuring-sdk/#specifying-credentials).
Regardless of the method used to acquire the credentials,
some permissions are needed for the exporter to work.

```
"tag:GetResources",
"cloudwatch:GetMetricData",
"cloudwatch:GetMetricStatistics",
"cloudwatch:ListMetrics"
```

The following IAM permissions are required for the [Transit Gateway](https://aws.amazon.com/transit-gateway/)
attachment (tgwa) metrics to work.

```
"ec2:DescribeTags",
"ec2:DescribeInstances",
"ec2:DescribeRegions",
"ec2:DescribeTransitGateway*"
```

The following IAM permission is required to discover tagged [API Gateway](https://aws.amazon.com/es/api-gateway/) REST
APIs:

```
"apigateway:GET"
```

The following IAM permissions are required to discover
tagged [Database Migration Service](https://aws.amazon.com/dms/) (DMS) replication instances and tasks:

```
"dms:DescribeReplicationInstances",
"dms:DescribeReplicationTasks"
```

To use all of the integration features, use the following AWS IAM Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1674249227793",
      "Action": [
        "tag:GetResources",
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "ec2:DescribeTags",
        "ec2:DescribeInstances",
        "ec2:DescribeRegions",
        "ec2:DescribeTransitGateway*",
        "apigateway:GET",
        "dms:DescribeReplicationInstances",
        "dms:DescribeReplicationTasks"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

## Usage

```alloy
prometheus.exporter.cloudwatch "queues" {
    sts_region      = "us-east-2"
    aws_sdk_version_v2 = false
    discovery {
        type        = "AWS/SQS"
        regions     = ["us-east-2"]
        search_tags = {
            "scrape" = "true",
        }

        metric {
            name       = "NumberOfMessagesSent"
            statistics = ["Sum", "Average"]
            period     = "1m"
        }

        metric {
            name       = "NumberOfMessagesReceived"
            statistics = ["Sum", "Average"]
            period     = "1m"
        }
    }
}
```

## Arguments

You can use the following arguments to configure the exporter's behavior.
Omitted fields take their default values.

| Name                      | Type                | Description                                                                    | Default | Required |
|---------------------------|---------------------|--------------------------------------------------------------------------------|---------|----------|
| `sts_region`              | `string`            | AWS region to use when calling [STS][] for retrieving account information.     |         | yes      |
| `aws_sdk_version_v2`      | `bool`              | Use AWS SDK version 2.                                                                                                                                                                                                                  | `false` | no       |
| `fips_disabled`           | `bool`              | Disable use of FIPS endpoints. Set 'true' when running outside of USA regions. | `true`  | no       |
| `debug`                   | `bool`              | Enable debug logging on CloudWatch exporter internals.                         | `false` | no       |
| `discovery_exported_tags` | `map(list(string))` | List of tags (value) per service (key) to export in all metrics. For example, defining the `["name", "type"]` under `"AWS/EC2"` will export the name and type tags and its values as labels in all metrics. Affects all discovery jobs. | `{}` | no |

[STS]: https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html

## Blocks

You can use the following blocks in`prometheus.exporter.cloudwatch` to configure collector-specific options:

| Hierarchy                 | Name                   | Description                                                                                                                                                | Required |
|--------------------|------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|----------|
| discovery                 | [discovery][]          | Configures a discovery job. Multiple jobs can be configured.                                                                                               | no\*     |
| discovery > role          | [role][]               | Configures the IAM roles the job should assume to scrape metrics. Defaults to the role configured in the environment {{< param "PRODUCT_NAME" >}} runs on. | no       |
| discovery > metric        | [metric][]             | Configures the list of metrics the job should scrape. Multiple metrics can be defined inside one job.                                                      | yes      |
| static                    | [static][]             | Configures a static job. Multiple jobs can be configured.                                                                                                  | no\*     |
| static > role             | [role][]               | Configures the IAM roles the job should assume to scrape metrics. Defaults to the role configured in the environment {{< param "PRODUCT_NAME" >}} runs on. | no       |
| static > metric           | [metric][]             | Configures the list of metrics the job should scrape. Multiple metrics can be defined inside one job.                                                      | yes      |
| custom_namespace          | [custom_namespace][]   | Configures a custom namespace job. Multiple jobs can be configured.                                                                                               | no\*     |
| custom_namespace > role   | [role][]               | Configures the IAM roles the job should assume to scrape metrics. Defaults to the role configured in the environment {{< param "PRODUCT_NAME" >}} runs on. | no       |
| custom_namespace > metric | [metric][]             | Configures the list of metrics the job should scrape. Multiple metrics can be defined inside one job.                                                      | yes      |
| decoupled_scraping        | [decoupled_scraping][] | Configures the decoupled scraping feature to retrieve metrics on a schedule and return the cached metrics.                                                 | no       |

{{< admonition type="note" >}}
The `static`, `discovery`, and `custom_namespace` blocks are marked as not required, 
but you must configure at least one static, discovery, or custom namespace job.
{{< /admonition >}}

[discovery]: #discovery-block
[static]: #static-block
[custom_namespace]: #custom_namespace-block
[metric]: #metric-block
[role]: #role-block
[decoupled_scraping]: #decoupled_scraping-block

### discovery block

The `discovery` block allows the component to scrape CloudWatch metrics with only the AWS service and a list of metrics
under that service/namespace.
{{< param "PRODUCT_NAME" >}} will find AWS resources in the specified service for which to scrape these metrics, label them appropriately,
and export them to Prometheus. For example, if we wanted to scrape CPU utilization and network traffic metrics from all AWS EC2 instances:

```alloy
prometheus.exporter.cloudwatch "discover_instances" {
    sts_region = "us-east-2"

    discovery {
        type    = "AWS/EC2"
        regions = ["us-east-2"]

        metric {
            name       = "CPUUtilization"
            statistics = ["Average"]
            period     = "5m"
        }

        metric {
            name       = "NetworkPacketsIn"
            statistics = ["Average"]
            period     = "5m"
        }
    }
}
```

You can configure the `discovery` block one or multiple times to scrape metrics from different services or with
different `search_tags`.

| Name                          | Type           | Description                                                                                                                                                                                                                                                | Default | Required |
| ----------------------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | -------- |
| `regions`                     | `list(string)` | List of AWS regions.                                                                                                                                                                                                                                       |         | yes      |
| `type`                        | `string`       | CloudWatch service alias (`"alb"`, `"ec2"`, etc) or namespace name (`"AWS/EC2"`, `"AWS/S3"`, etc). Refer to [supported-services][] for a complete list.                                                                                                    |         | yes      |
| `custom_tags`                 | `map(string)`  | Custom tags to be added as a list of key / value pairs. When exported to Prometheus format, the label name follows the following format: `custom_tag_{key}`.                                                                                               | `{}`    | no       |
| `recently_active_only`        | `bool`         | Only return metrics that have been active in the last 3 hours.                                                                                                                                 | `false` | no       |
| `search_tags`                 | `map(string)`  | List of key / value pairs to use for tag filtering (all must match). Value can be a regex.                                                                                                                                                                 | `{}`    | no       |
| `dimension_name_requirements` | `list(string)` | List of metric dimensions to query. Before querying metric values, the total list of metrics will be filtered to only those that contain exactly this list of dimensions. An empty or undefined list results in all dimension combinations being included. | `{}`    | no       |
| `nil_to_zero`                 | `bool`         | When `true`, `NaN` metric values are converted to 0. Individual metrics can override this value in the [metric][] block.                                                                                                                                   | `true`  | no       |

[supported-services]: #supported-services-in-discovery-jobs

### static block

The `static` block configures the component to scrape a specific set of CloudWatch metrics. The metrics need to be fully
qualified with the following specifications:

1. `namespace`: For example, `AWS/EC2`, `AWS/EBS`, `CoolApp` if it were a custom metric, etc.
2. `dimensions`: CloudWatch identifies a metric by a set of dimensions, which are essentially label / value pairs. For
   example, all `AWS/EC2` metrics are identified by the `InstanceId` dimension and the identifier itself.
3. `metric`: Metric name and statistics.

For example, if you want to scrape the same metrics in the discovery example, but for a specific AWS EC2 instance:

```alloy
prometheus.exporter.cloudwatch "static_instances" {
    sts_region = "us-east-2"

    static "instances" {
        regions    = ["us-east-2"]
        namespace  = "AWS/EC2"
        dimensions = {
            "InstanceId" = "i01u29u12ue1u2c",
        }

        metric {
            name       = "CPUUsage"
            statistics = ["Sum", "Average"]
            period     = "1m"
        }
    }
}
```

As shown above, `static` blocks must be specified with a label, which will translate to the `name` label in the exported
metric.

```alloy
static "LABEL" {
    regions    = ["us-east-2"]
    namespace  = "AWS/EC2"
    // ...
}
```

You can configure the `static` block one or multiple times to scrape metrics with different sets of `dimensions`.

| Name          | Type           | Description                                                                                                                                                  | Default | Required |
| ------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------- | -------- |
| `regions`     | `list(string)` | List of AWS regions.                                                                                                                                         |         | yes      |
| `namespace`   | `string`       | CloudWatch metric namespace.                                                                                                                                 |         | yes      |
| `dimensions`  | `map(string)`  | CloudWatch metric dimensions as a list of name / value pairs. Must uniquely define all metrics in this job.                                                  |         | yes      |
| `custom_tags` | `map(string)`  | Custom tags to be added as a list of key / value pairs. When exported to Prometheus format, the label name follows the following format: `custom_tag_{key}`. | `{}`    | no       |
| `nil_to_zero` | `bool`         | When `true`, `NaN` metric values are converted to 0. Individual metrics can override this value in the [metric][] block.                                     | `true`  | no       |

All dimensions must be specified when scraping single metrics like the example above. For example, `AWS/Logs` metrics
require `Resource`, `Service`, `Class`, and `Type` dimensions to be specified. The same applies to CloudWatch custom
metrics,
all dimensions attached to a metric when saved in CloudWatch are required.

### custom_namespace block

The `custom_namespace` block allows the component to scrape CloudWatch metrics from custom namespaces using only the namespace name and a list of metrics under that namespace.
For example:

```alloy
prometheus.exporter.cloudwatch "discover_instances" {
    sts_region = "eu-west-1"

    custom_namespace "customEC2Metrics" {
        namespace = "CustomEC2Metrics"
        regions   = ["us-east-1"]

        metric {
            name       = "cpu_usage_idle"
            statistics = ["Average"]
            period     = "5m"
        }

        metric {
            name       = "disk_free"
            statistics = ["Average"]
            period     = "5m"
        }
    }
}
```

You can configure the `custom_namespace` block multiple times if you need to scrape metrics from different namespaces.

| Name                          | Type           | Description                                                                                                                                                                                                                                                | Default | Required |
| ----------------------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | -------- |
| `regions`                     | `list(string)` | List of AWS regions.                                                                                                                                                                                                                                       |         | yes      |
| `namespace`                   | `string`       | CloudWatch metric namespace.                                                                                                                                 |         | yes      |
| `recently_active_only`        | `bool`         | Only return metrics that have been active in the last 3 hours.                                                                                                                                 | `false` | no       |
| `custom_tags`                 | `map(string)`  | Custom tags to be added as a list of key / value pairs. When exported to Prometheus format, the label name follows the following format: `custom_tag_{key}`.                                                                                               | `{}`    | no       |
| `dimension_name_requirements` | `list(string)` | List of metric dimensions to query. Before querying metric values, the total list of metrics will be filtered to only those that contain exactly this list of dimensions. An empty or undefined list results in all dimension combinations being included. | `{}`    | no       |
| `nil_to_zero`                 | `bool`         | When `true`, `NaN` metric values are converted to 0. Individual metrics can override this value in the [metric][] block.                                                                                                                                   | `true`  | no       |


### metric block

Represents an AWS Metrics to scrape. To see available metrics, AWS does not keep a documentation page with all available
metrics.
Follow [this guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/viewing_metrics_with_cloudwatch.html)
on how to explore metrics, to easily pick the ones you need.

| Name          | Type           | Description                                                               | Default                                                                                                            | Required |
| ------------- | -------------- | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | -------- |
| `name`        | `string`       | Metric name.                                                              |                                                                                                                    | yes      |
| `statistics`  | `list(string)` | List of statistics to scrape. For example, `"Minimum"`, `"Maximum"`, etc. |                                                                                                                    | yes      |
| `period`      | `duration`     | Refer to the [period][] section below.                                    |                                                                                                                    | yes      |
| `length`      | `duration`     | Refer to the [period][] section below.                                    | Calculated based on `period`. Refer to [period][] for details.                                                     | no       |
| `nil_to_zero` | `bool`         | When `true`, `NaN` metric values are converted to 0.                      | The value of `nil_to_zero` in the parent [static][] or [discovery][] block. `true` if not set in the parent block. | no       |

[period]: #period-and-length

#### period and length

`period` controls primarily the width of the time bucket used for aggregating metrics collected from CloudWatch. `length`
controls how far back in time CloudWatch metrics are considered during each {{< param "PRODUCT_NAME" >}} scrape.
If both settings are configured, the time parameters when calling CloudWatch APIs works as follows:

{{< figure src="/media/docs/alloy/cloudwatch-period-and-length-time-model-2.png" alt="An example of a CloudWatch period and length time model" >}}

As noted above, if across multiple metrics under the same static or discovery job, there's different `period`
and/or `length`
the minimum of all periods, and maximum of all lengths is configured.

On the other hand, if `length` is not configured, both period and length settings will be calculated based on the
required
`period` configuration attribute.

If all metrics within a job (discovery or static) have the same `period` value configured, CloudWatch APIs will be
requested
for metrics from the scrape time, to `period`s seconds in the past. The values of these are exported to Prometheus.

{{< figure src="/media/docs/alloy/cloudwatch-single-period-time-model.png" alt="An example of a CloudWatch single period and time model" >}}

On the other hand, if metrics with different `period`s are configured under an individual job, this works differently.
First, two variables are calculated aggregating all periods: `length`, taking the maximum value of all periods, and
the new `period` value, taking the minimum of all periods. Then, CloudWatch APIs will be requested for metrics from
`now - length` to `now`, aggregating each in samples for `period` seconds. For each metric, the most recent sample
is exported to CloudWatch.

{{< figure src="/media/docs/alloy/cloudwatch-multiple-period-time-model.png" alt="An example of a CloudWatch multiple period and time model" >}}

### role block

Represents an [AWS IAM Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html). If omitted, the AWS role
that corresponds to the credentials configured in the environment will be used.

Multiple roles can be useful when scraping metrics from different AWS accounts with a single pair of credentials. In
this case, a different role
is configured for {{< param "PRODUCT_NAME" >}} to assume before calling AWS APIs. Therefore, the credentials configured in the system need
permission to assume the target role.
Refer to [Granting a user permissions to switch roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_permissions-to-switch.html)
in the AWS IAM documentation for more information about how to configure this.

| Name          | Type     | Description                                                             | Default | Required |
| ------------- | -------- | ----------------------------------------------------------------------- | ------- | -------- |
| `role_arn`    | `string` | AWS IAM Role ARN the exporter should assume to perform AWS API calls.   |         | yes      |
| `external_id` | `string` | External ID used when calling STS AssumeRole API. Refer to the [IAM User Guide][details] for more information. | `""`    | no       |

[details]: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html

### decoupled_scraping block

The `decoupled_scraping` block configures an optional feature that scrapes CloudWatch metrics in the background on a
scheduled interval. When this feature is enabled, CloudWatch metrics are gathered asynchronously at the scheduled interval instead
of synchronously when the CloudWatch component is scraped.

The decoupled scraping feature reduces the number of API requests sent to AWS.
This feature also prevents component scrape timeouts when you gather high volumes of CloudWatch metrics.

| Name              | Type     | Description                                                             | Default | Required |
| ----------------- | -------- | ----------------------------------------------------------------------- | ------- | -------- |
| `enabled`         | `bool`   | Controls whether the decoupled scraping featured is enabled             | false   | no       |
| `scrape_interval` | `string` | Controls how frequently to asynchronously gather new CloudWatch metrics | 5m      | no       |

## Exported fields

{{< docs/shared lookup="reference/components/exporter-component-exports.md" source="alloy" version="<ALLOY_VERSION>" >}}

## Component health

`prometheus.exporter.cloudwatch` is only reported as unhealthy if given
an invalid configuration. In those cases, exported fields retain their last
healthy values.

## Debug information

`prometheus.exporter.cloudwatch` does not expose any component-specific
debug information.

## Debug metrics

`prometheus.exporter.cloudwatch` does not expose any component-specific
debug metrics.

## Example

See the examples described under each [discovery][] and [static] sections.

[discovery]: #discovery-block
[static]: #static-block

## Supported services in discovery jobs

The following is a list of AWS services that are supported in `cloudwatch_exporter` discovery jobs. When configuring a
discovery job, the `type` field of each `discovery_job` must match the desired job namespace.

- Namespace: `CWAgent`
- Namespace: `AWS/Usage`
- Namespace: `AWS/CertificateManager`
- Namespace: `AWS/ACMPrivateCA`
- Namespace: `AmazonMWAA`
- Namespace: `AWS/MWAA`
- Namespace: `AWS/ApplicationELB`
- Namespace: `AWS/AppStream`
- Namespace: `AWS/Backup`
- Namespace: `AWS/ApiGateway`
- Namespace: `AWS/AmazonMQ`
- Namespace: `AWS/AppRunner`
- Namespace: `AWS/AppSync`
- Namespace: `AWS/Athena`
- Namespace: `AWS/AutoScaling`
- Namespace: `AWS/ElasticBeanstalk`
- Namespace: `AWS/Billing`
- Namespace: `AWS/Cassandra`
- Namespace: `AWS/CloudFront`
- Namespace: `AWS/Cognito`
- Namespace: `AWS/DataSync`
- Namespace: `AWS/DMS`
- Namespace: `AWS/DDoSProtection`
- Namespace: `AWS/DocDB`
- Namespace: `AWS/DX`
- Namespace: `AWS/DynamoDB`
- Namespace: `AWS/EBS`
- Namespace: `AWS/ElastiCache`
- Namespace: `AWS/MemoryDB`
- Namespace: `AWS/EC2`
- Namespace: `AWS/EC2Spot`
- Namespace: `AWS/EC2CapacityReservations`
- Namespace: `AWS/ECS`
- Namespace: `ECS/ContainerInsights`
- Namespace: `AWS/EFS`
- Namespace: `AWS/ELB`
- Namespace: `AWS/ElasticMapReduce`
- Namespace: `AWS/EMRServerless`
- Namespace: `AWS/ES`
- Namespace: `AWS/Firehose`
- Namespace: `AWS/FSx`
- Namespace: `AWS/GameLift`
- Namespace: `AWS/GatewayELB`
- Namespace: `AWS/GlobalAccelerator`
- Namespace: `Glue`
- Namespace: `AWS/IoT`
- Namespace: `AWS/Kafka`
- Namespace: `AWS/KafkaConnect`
- Namespace: `AWS/Kinesis`
- Namespace: `AWS/KinesisAnalytics`
- Namespace: `AWS/KMS`
- Namespace: `AWS/Lambda`
- Namespace: `AWS/Logs`
- Namespace: `AWS/MediaConnect`
- Namespace: `AWS/MediaConvert`
- Namespace: `AWS/MediaPackage`
- Namespace: `AWS/MediaLive`
- Namespace: `AWS/MediaTailor`
- Namespace: `AWS/Neptune`
- Namespace: `AWS/NetworkFirewall`
- Namespace: `AWS/NATGateway`
- Namespace: `AWS/NetworkELB`
- Namespace: `AWS/PrivateLinkEndpoints`
- Namespace: `AWS/PrivateLinkServices`
- Namespace: `AWS/Prometheus`
- Namespace: `AWS/QLDB`
- Namespace: `AWS/RDS`
- Namespace: `AWS/Redshift`
- Namespace: `AWS/Route53Resolver`
- Namespace: `AWS/Route53`
- Namespace: `AWS/RUM`
- Namespace: `AWS/S3`
- Namespace: `AWS/SecretsManager`
- Namespace: `AWS/SES`
- Namespace: `AWS/States`
- Namespace: `AWS/SNS`
- Namespace: `AWS/SQS`
- Namespace: `AWS/StorageGateway`
- Namespace: `AWS/TransitGateway`
- Namespace: `AWS/TrustedAdvisor`
- Namespace: `AWS/VPN`
- Namespace: `AWS/ClientVPN`
- Namespace: `AWS/WAFV2`
- Namespace: `AWS/WorkSpaces`
- Namespace: `AWS/AOSS`
- Namespace: `AWS/SageMaker`
- Namespace: `/aws/sagemaker/Endpoints`
- Namespace: `/aws/sagemaker/TrainingJobs`
- Namespace: `/aws/sagemaker/ProcessingJobs`
- Namespace: `/aws/sagemaker/TransformJobs`
- Namespace: `/aws/sagemaker/InferenceRecommendationsJobs`
- Namespace: `AWS/Sagemaker/ModelBuildingPipeline`
- Namespace: `AWS/IPAM`
- Namespace: `AWS/Bedrock`
- Namespace: `AWS/Events`

<!-- START GENERATED COMPATIBLE COMPONENTS -->

## Compatible components

`prometheus.exporter.cloudwatch` has exports that can be consumed by the following components:

- Components that consume [Targets](../../../compatibility/#targets-consumers)

{{< admonition type="note" >}}
Connecting some components may not be sensible or components may require further configuration to make the connection work correctly.
Refer to the linked documentation for more details.
{{< /admonition >}}

<!-- END GENERATED COMPATIBLE COMPONENTS -->
