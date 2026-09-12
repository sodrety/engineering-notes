---
title: 'Kubernetes Native Histograms: Stop Guessing Your Latency Buckets'
description: 'Kubernetes v1.37 makes native histograms easier to adopt. Here is the observability trade-off, the safe migration path, and the PromQL that changes.'
pubDate: 2026-09-12
---

Latency dashboards are only as useful as the measurements behind them.

For years, Prometheus users have represented latency with classic histograms: choose bucket boundaries in advance, count observations in each bucket, and estimate percentiles from those counts. It is simple and widely supported, but it creates an uncomfortable design problem:

> You must guess the useful resolution before you know the shape of the traffic.

Kubernetes v1.37 makes a different model easier to use. Native histogram support has graduated to Beta and is enabled by default in Kubernetes components. The change is not just a new metric format. It is a lesson in how to evolve observability without breaking every existing dashboard at once.

## Why fixed buckets become a production problem

Suppose an API server exposes request latency with these buckets:

```text
5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s
```

That list may work during initial testing. Later, the service changes:

- a fast cache path moves most requests into the microsecond range;
- a dependency introduces a long tail beyond one second;
- a new region has very different network latency;
- an SLO moves from p95 to p99.9.

The old buckets do not adapt. Fine-grained buckets improve accuracy but create more time series. Coarse buckets reduce storage but make percentile estimates less precise. `histogram_quantile()` must interpolate between the boundaries, so a wide bucket can hide a meaningful change in the tail.

Classic histograms also represent each bucket as a separate labelled series, commonly using a label such as `le`. Ten buckets across several label combinations can multiply the number of series that Prometheus scrapes and stores.

## What a native histogram changes

A Prometheus native histogram stores a distribution as one structured time series. Instead of requiring application authors to publish a fixed list of bucket boundaries, it uses dynamic exponential spans, plus metadata describing the schema and zero threshold.

The practical differences are:

- the resolution adapts across a wider value range;
- the distribution does not need one time series per bucket boundary;
- quantile calculations can use the histogram’s structure rather than only coarse fixed buckets;
- the metric name no longer needs a `_bucket` suffix for the native query path.

The Kubernetes announcement describes the default configuration as using a bucket factor of 1.1 and a maximum of 160 buckets. Under that configuration, it describes a worst-case relative quantile error of about 5%. It also reports that native histograms can reduce time-series overhead substantially, in some cases by up to 90%.

Those are useful design targets, not guarantees for every workload. Storage savings depend on labels, scrape settings, retention, aggregation, and the rest of the metrics pipeline.

## The compatibility design matters more than the format

The risky way to introduce a new metrics representation would be to stop emitting the old one immediately. Existing dashboards, alerts, recording rules, exporters, and remote storage integrations could silently lose data.

Kubernetes takes a safer route: dual exposition.

When native histograms are enabled, components can expose classic buckets for existing consumers while including native histogram data in the Prometheus protobuf payload for consumers that understand it. That lets an organization upgrade the collection path before it rewrites every query.

The migration therefore becomes a sequence of reversible changes instead of one large cutover:

```text
Kubernetes v1.37
       |
       +--> classic buckets ------> existing dashboards and alerts
       |
       +--> native spans ---------> new PromQL and storage path
```

For a monitoring team, this is the important architecture: **new data can be introduced without forcing every reader to change on the same day**.

## A safer Prometheus migration

For Prometheus 3.x, start with explicit scrape configuration for the jobs you are testing:

```yaml
scrape_configs:
  - job_name: kubernetes-apiservers
    scrape_native_histograms: true
    always_scrape_classic_histograms: true
```

Keep `always_scrape_classic_histograms: true` during the transition. It allows existing queries based on `_bucket`, `_count`, and `_sum` to continue working while you migrate dashboards and alerts.

Then create a small test matrix:

1. Confirm that the Prometheus version accepts native histogram scraping.
2. Confirm that the Kubernetes endpoint negotiates the protobuf exposition.
3. Compare classic and native p95/p99 results for the same service.
4. Check dashboards, recording rules, alerts, remote write, and long-term storage.
5. Roll back collection for the test job if any consumer behaves unexpectedly.

Do not begin by changing every query in the organization. Choose one latency metric with a known tail problem and migrate that metric first.

## The PromQL changes are small, but important

A classic histogram query often looks like this:

```text
histogram_quantile(
  0.99,
  sum by (le) (
    rate(apiserver_request_duration_seconds_bucket[5m])
  )
)
```

The native histogram query operates directly on the metric:

```text
histogram_quantile(
  0.99,
  sum(rate(apiserver_request_duration_seconds[5m]))
)
```

The missing `le` grouping is not cosmetic. It reflects the fact that the native histogram is no longer a collection of independent bucket series that must be aligned manually.

During migration, keep both queries in a temporary comparison dashboard. Differences do not automatically mean that one query is broken; they may expose interpolation error in the classic representation or a change in the data path. Investigate the distribution before changing an alert threshold.

## When native histograms are not automatically better

Native histograms add capability, but they also add compatibility requirements.

Before enabling them broadly, check:

- Prometheus and collector versions;
- remote-write and remote-read support;
- dashboard and alert query behavior;
- recording rules that refer to `_bucket`, `_count`, or `_sum`;
- storage and retention costs for both formats during the overlap period;
- operational tooling that parses the text exposition format directly.

If a downstream system does not understand native histograms, keep classic exposition enabled for that path. The point of the Beta migration is not to remove every classic metric immediately. It is to make the higher-resolution representation available while the ecosystem catches up.

## A practical rollout checklist

- [ ] Pick one latency metric with a visible tail or bucket-resolution problem.
- [ ] Upgrade or verify a Prometheus collector that supports native histograms.
- [ ] Enable native scraping for one Kubernetes job.
- [ ] Keep classic histograms enabled during comparison.
- [ ] Compare p95 and p99 results over the same traffic window.
- [ ] Migrate one dashboard and one alert to native PromQL.
- [ ] Test remote storage and recording rules.
- [ ] Measure actual series count and storage before claiming savings.
- [ ] Keep a rollback switch for the collector and Kubernetes feature gate.
- [ ] Disable classic ingestion only after consumers are verified.

## Final take

The most useful part of Kubernetes native histograms is not that they promise fewer time series. It is the migration pattern behind them:

> Introduce a better internal representation, expose it beside the old one, migrate readers gradually, and measure the real operational result before removing compatibility.

That is a good way to evolve observability systems in general. Metrics are APIs. Changing their representation without a compatibility plan turns a performance improvement into an incident.

## References

- [Kubernetes v1.37: Native Histograms Graduates to Beta](https://kubernetes.io/blog/2026/09/11/kubernetes-v1-37-native-histograms-beta/) — official Kubernetes blog, September 11, 2026.
- [Prometheus Native Histograms specification](https://prometheus.io/docs/specs/native_histograms/) — official Prometheus documentation.
