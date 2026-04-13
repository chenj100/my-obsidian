---
title: YACE CloudWatch Exporter Gotchas
created: 2026-04-13
tags:
  - type/permanent
  - status/evergreen
  - domain/engineering
---

YACE (Yet Another CloudWatch Exporter) has several non-obvious behaviours that cause incorrect or missing metrics if not accounted for.

## 1. `length` must equal `scrape_interval`

YACE does **not** add timestamps to its exported `/metrics` endpoint. Prometheus therefore stamps each scrape with the current time. If `length` ≠ `scrape_interval`, you will get duplicate or missing data points.

```yaml
# scrape_interval: 5m → length must be 300
length: 300   # correct
length: 600   # wrong — duplicates
```

Alternative: set `addCloudwatchTimestamp: true` on the metric. This may relax the equality requirement, but `scrape_interval` should still be ≤ `length`.

Reference: [YACE issue #865](https://github.com/prometheus-community/yet-another-cloudwatch-exporter/issues/865#issuecomment-1517656590) has a thorough explanation of `scrape_interval`, `length`, `delay`, and `period`.

## 2. CloudWatch counters are deltas, not cumulative

YACE exports CloudWatch counters as **delta values** (change per period), not as ever-increasing Prometheus counters. This means `rate()` and `increase()` are wrong.

| Intent | Wrong | Correct |
|---|---|---|
| Total invocations over 1h | `increase(aws_lambda_invocations_sum[1h])` | `sum_over_time(aws_lambda_invocations_sum[1h])` |
| Invocation rate | `rate(aws_lambda_invocations_sum[1h])` | `sum_over_time(aws_lambda_invocations_sum[1h]) / 3600` |

## 3. Metrics are pre-aggregated — don't re-aggregate naively

CloudWatch metrics arrive already aggregated per period (e.g. average over 5 min). You cannot simply wrap them in `avg()` to get a 1h average — that gives an average of averages.

For a **weighted average** over a longer window, use `SampleCount`:

```promql
# Average Lambda duration over 1h
sum_over_time(aws_lambda_duration_sum[1h])
  / sum_over_time(aws_lambda_duration_sample_count[1h])
```

`SampleCount` is not included in default YACE statistics — add it explicitly to your metric config.

For **percentiles**: exporting p50/p95 per 5 min cannot be re-aggregated over longer windows. To get true percentiles you need histogram bucket counts. This may be achievable via CloudWatch [Trimmed Sum](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Statistics-definitions.html) statistics, but it's non-standard and increases cost (one series per bucket).

## 4. Untagged AWS resources are invisible to YACE

YACE discovers resources via `resourcegroupstaggingapi GetResources`, which silently excludes untagged resources. There is no workaround via `searchTags: none` — it doesn't work despite AWS docs implying otherwise.

**Workaround:** use `static_config` to hardcode ARNs/resource IDs for untagged resources until tags can be added.

## 5. S3 storage metrics have daily resolution and large delay

`BucketSizeBytes` and `NumberOfObjects` are only published once per day and can lag by hours or even days.

```yaml
- name: BucketSizeBytes
  period: 86400
  length: 300
  delay: 86400     # default delay: 120 will return nothing
  nilToZero: true
  statistics: [Average]
```

## 6. S3 request metrics must be explicitly enabled in AWS

Metrics like `GetRequests` and `PutRequests` are S3 request metrics and are **off by default**. Configuring them in YACE without first enabling them in AWS returns no data.

Enable via: [AWS docs — Creating a CloudWatch metrics configuration for a bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/configure-request-metrics-bucket.html)

## 7. Don't mix different `length` values within the same metric type

Having metrics of the same `type` (e.g. `s3`) with different `length` values can cause one metric's length to bleed into another's, producing duplicate data. Keep `length` consistent within a type block, or split into separate type blocks.
