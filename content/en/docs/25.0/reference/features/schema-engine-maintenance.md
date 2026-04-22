---
title: Schema Engine Maintenance
weight: 17
aliases: ['/docs/reference/schema-engine-maintenance/']
---

VTTablet includes automated maintenance features for monitoring and managing InnoDB table bloat. These help operators identify tables with excessive free space and prevent production outages caused by `mysql.gtid_executed` table bloat.

## User Table Free Space Alerting

VTTablet monitors user tables for InnoDB bloat and surfaces tables where reclaimable free space (`DATA_FREE`) exceeds a configured threshold. This is a visibility-only feature and does not modify user tables.

### Configuration

Use the following vttablet flag to enable free space alerting:

```
--schema-user-tables-free-space-percent-threshold int
```

- **Default**: 50
- **Range**: 0-100
- **Behavior**: When set to 0, the feature is disabled. When set to 1-100, VTTablet reports user tables where `DATA_FREE` exceeds the specified percentage of total allocated space.

This flag is dynamically reloadable via a viper config file, so you can adjust the threshold without restarting vttablet.

### Metric

When a user table exceeds the configured free space threshold, VTTablet exports the `SchemaTableDataFreeBytes` metric:

```
SchemaTableDataFreeBytes{Table="<table_name>"}
```

- **Type**: Gauge
- **Value**: Raw `DATA_FREE` bytes for the table
- **Labels**: `Table` label identifies the specific table

The metric label is removed when:
- The table drops below the threshold
- The table is dropped
- The feature is disabled

### How It Works

During each periodic schema reload, VTTablet queries `information_schema.TABLES` to check the free space ratio for each user table:

```sql
data_free * 100 > (data_length + index_length + data_free) * <percent_threshold>
```

Tables exceeding the threshold are logged at INFO level and exported via the `SchemaTableDataFreeBytes` metric. VTTablet does not automatically run `OPTIMIZE TABLE` on user tables.

### Alerting Recommendations

Set up alerts on the `SchemaTableDataFreeBytes` metric to proactively identify tables that may benefit from optimization. You can then schedule `OPTIMIZE TABLE` operations during maintenance windows.

## Automatic mysql.gtid_executed Optimization

VTTablet automatically optimizes the `mysql.gtid_executed` table on non-primary tablets to prevent production outages caused by table bloat.

### Background

The `mysql.gtid_executed` table stores GTID execution history and can grow significantly over time. When this table accumulates multiple GiB of bloat, binlog rotation can trigger compaction scans that cause InnoDB assertion failures and database outages.

### How It Works

VTTablet runs `OPTIMIZE NO_WRITE_TO_BINLOG TABLE mysql.gtid_executed` on non-primary tablets when the following conditions are met:

1. **Tablet is not primary**: Only runs on replica, rdonly, or other non-primary tablet types
2. **5-minute cooldown**: Waits 5 minutes after any primary-to-non-primary or non-primary-to-primary transition
3. **128 MiB threshold**: Only triggers when `DATA_FREE` exceeds 128 MiB
4. **24-hour throttle**: Each tablet runs OPTIMIZE at most once every 24 hours
5. **60-second timeout**: Operations that exceed 60 seconds are terminated

### Key Behaviors

- **No configuration required**: The 128 MiB threshold is intentionally not configurable. This is an opinionated safety measure to prevent outages without requiring operator tuning.
- **Non-blocking**: Runs in a background goroutine without affecting query serving
- **Super read-only handling**: Temporarily disables `@@global.super_read_only` on the replica during optimization
- **Failover-safe**: Aborts if the tablet is promoted to primary during optimization

### Monitoring

There is no dedicated metric for `mysql.gtid_executed` optimization. Monitor vttablet logs at INFO level for optimization events and at WARN level for failures.

Log messages to alert on:

- **WARN** messages containing `CRITICAL:` indicate `super_read_only` restore failures. If this occurs, the replica may temporarily accept writes until MySQL is restarted.

### Operational Notes

- External `OPTIMIZE TABLE` operations on `mysql.gtid_executed` do not reset the 24-hour throttle. The internal timer is based on vttablet's last successful optimization.
- The optimization runs on all non-primary tablet types, including `primary-not-serving` during transitions.
