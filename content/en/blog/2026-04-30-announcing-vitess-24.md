---
author: 'Vitess Maintainer Team'
date: 2026-04-30
slug: '2026-04-30-announcing-vitess-24'
tags: [ 'release', 'Vitess', 'v24', 'MySQL', 'kubernetes', 'operator', 'vreplication', 'query-serving', 'observability' ]
title: 'Announcing Vitess 24'
description: "Vitess 24 is now Generally Available"
---

# **Announcing Vitess 24**

The Vitess maintainers are happy to announce the release of [version 24.0.0](https://github.com/vitessio/vitess/releases/tag/v24.0.0), along with [version 2.17.0](https://github.com/planetscale/vitess-operator/releases/tag/v2.17.0) of the Vitess Kubernetes Operator.

Version 24.0.0 delivers major advances in query serving, introduces binlog streaming through VTGate, and makes structured logging the default across Vitess. This release also improves operational tooling with better VTOrc recovery ordering and MySQL CLONE support for faster replica provisioning.

This blog post highlights some of the major changes in this release. For a more detailed description, please refer to the [release notes](https://github.com/vitessio/vitess/releases/tag/v24.0.0).

---

## Summary

* **Query Serving**: Window function pushdown, View Routing Rules, and tablet targeting via USE statements
* **VTGate**: Binlog streaming support via MySQL protocol and gRPC
* **Logging**: Structured JSON logging is now the default
* **Cluster Management and VTOrc**: Ordered recovery execution, semi-sync rollout improvements, new `--cell` flag
* **Backup & Restore**: MySQL CLONE support for faster replica provisioning
* **Observability**: QueryThrottler metrics and OpenTelemetry tracing support
* **Kubernetes Operator**: Vitess Operator v2.17.0

---

### Query Serving

Vitess 24.0 brings significant query serving enhancements that expand what queries can be pushed down to shards and give operators more control over query routing.

#### Window Function Pushdown

Vitess can now push window functions down to individual shards in sharded keyspaces when the `PARTITION BY` clause aligns with the sharding key. This avoids pulling all rows to VTGate for processing, significantly improving performance for analytical queries that use `ROW_NUMBER()`, `RANK()`, `SUM() OVER()`, and similar window functions.

#### View Routing Rules

A new View Routing Rules feature allows you to route queries against a view to a different view or table. This is useful for migrations and for exposing different views of the same underlying data to different applications.

#### Tablet Targeting via USE Statement

You can now target queries to specific tablet types using the `USE` statement with a tablet type suffix:

```sql
USE `keyspace:shard@replica`;
```

This enables session-level control over read routing without changing application code. The targeting persists for the session until changed or cleared.

#### JSON_EXTRACT Dynamic Arguments

The `JSON_EXTRACT` function now supports dynamic path arguments like bind variables or results from other function calls. Previously, only static string literals were supported for path arguments. NULL handling now matches MySQL behavior as well.

### VTGate Binlog Streaming

VTGate now supports GTID-based binlog streaming through two protocols:

- **MySQL protocol**: Clients can connect using the standard MySQL `COM_BINLOG_DUMP_GTID` replication protocol command
- **gRPC**: The new `BinlogDumpGTID` streaming RPC provides native gRPC access for custom clients

This allows binlog clients to connect to Vitess without special VStream-aware adapters or direct MySQL access. The feature is disabled by default; enable it with `--enable-binlog-dump`. Use `--binlog-dump-authorized-users` to control which users can stream binlogs.

For workflows that need automatic resharding, multi-shard aggregation, or event filtering, use VStream instead.

### Structured Logging

Vitess now uses structured JSON logging by default. Log output is emitted as JSON objects with consistent field names across all components, making it easier to parse and query logs with standard tools.

The previous `glog`-based logging is deprecated as of v24 and will be removed in v25. Use `--log-format=text` to revert to the previous format during the transition period.

### Cluster Management and VTOrc

#### New `--cell` Flag

VTOrc now requires a `--cell` flag to specify which cell it operates in. This enables future cross-cell problem validation and helps VTOrc make smarter decisions about tablet availability and recovery actions. In v25, this flag will become mandatory.

#### Ordered Recovery Execution and Semi-Sync Rollout

VTOrc now executes recoveries per-shard with a defined ordering, rather than per-tablet in isolation. Problems with ordering dependencies (like semi-sync configuration) are executed serially first, while independent problems run concurrently. This ensures dependent recoveries happen in the correct sequence.

The main user-facing improvement is for semi-sync rollouts: VTOrc now ensures replicas have semi-sync enabled before updating the primary, preventing write stalls that could occur when the primary waits for acknowledgements that no replica is prepared to send.

### Backup & Restore

#### MySQL CLONE Support

VTTablet and VTBackup now support using MySQL's native CLONE plugin to provision new replicas by copying data directly from a donor tablet over the network. Physical-level data copying is significantly faster than logical backup and restore, especially for large datasets.

New flags:
- `--clone-from`: Tablet alias to clone from (vttablet)
- `--clone-from-keyspace`, `--clone-from-shard`, `--clone-from-tablet-type`: Clone source specification (vtbackup)

Requires MySQL 8.0.17+ and InnoDB-only tables.

#### Breaking Change: External Decompressor

The external decompressor command stored in a backup's `MANIFEST` file is no longer used at restore time by default. This change addresses a security risk where an attacker with write access to backup storage could modify the `MANIFEST` to execute arbitrary commands on the tablet.

If you rely on the previous behavior, add `--external-decompressor-use-manifest` to your VTTablet configuration, but be aware of the security implications.

### Observability

#### QueryThrottler Metrics

Four new metrics provide visibility into query throttling behavior:

- **QueryThrottlerRequests**: Total requests evaluated by the query throttler
- **QueryThrottlerThrottled**: Requests that were throttled
- **QueryThrottlerTotalLatencyNs**: Total time spent in query throttling
- **QueryThrottlerEvaluateLatencyNs**: Time taken to make throttling decisions

All metrics include labels for `Strategy`, `Workload`, and `Priority`. The throttled metric includes additional labels for `MetricName`, `MetricValue`, and `DryRun`.

#### Connection Pool Waiter Cap

VTTablet now allows setting limits on the number of requests waiting to get a connection from the connection pool:

- `--queryserver-config-query-pool-waiter-cap`
- `--queryserver-config-stream-pool-waiter-cap`
- `--queryserver-config-txpool-waiter-cap`

A new metric `ConnPoolWaitersExhausted` tracks how often connection attempts fail due to hitting the waiter cap.

#### OpenTelemetry Tracing

Vitess now supports OpenTelemetry for distributed tracing. OpenTracing-based backends (Jaeger, DataDog, ZipKin) are deprecated and will be removed in a future release.

### Kubernetes Operator

Vitess v24.0.0 comes with a companion release of the [vitess-operator v2.17.0](https://github.com/planetscale/vitess-operator/releases/tag/v2.17.0).

Please refer to the [operator release notes](https://github.com/planetscale/vitess-operator/releases/tag/v2.17.0) to learn more about the new features and improvements.

---

### Migrate and Learn More

To ease migration from a previous version to v24.0.0, we highly recommend reading the release notes for both [Vitess](https://github.com/vitessio/vitess/releases/tag/v24.0.0) and the [Kubernetes Operator](https://github.com/planetscale/vitess-operator/releases/tag/v2.17.0). The entire [changelog](https://github.com/vitessio/vitess/blob/main/changelog/24.0/24.0.0/changelog.md) for this version is available too.

We also recommend exploring our [documentation for v24.0.0](https://vitess.io/docs/24.0/), where you can find step-by-step user guides, best practices, and tips for running Vitess.

### Community

As an open-source project, we truly appreciate feedback, insights, and contributions from our community.
Whether you want to share a story, ask a question, or anything else, you can reach out to us on [GitHub](https://github.com/vitessio/vitess) or in our [Slack](http://vitess.io/slack).

---

*The Vitess Maintainer Team*
