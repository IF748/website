---
title: VTOrc
weight: 8
---

VTOrc is the automated fault detection and repair tool of Vitess. It started off as a fork of the [Orchestrator](https://github.com/openark/orchestrator), which was then custom-fitted to the Vitess use-case running as a Vitess component.
An overview of the architecture of VTOrc can be found on this [page](../../../reference/vtorc/architecture).

Setting up VTOrc lets you avoid performing the `InitShardPrimary` step. It automatically detects that the new shard doesn't have a primary and elects one for you.
It detects any configuration problems in the cluster and fixes them. Here is the list of things VTOrc can do for you:

| Recovery Name                                                                                                                                            | Description                                                                                                                                                                                                      | Fix that VTOrc does                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| `ClusterHasNoPrimary`                                                                                                                                    | VTOrc detects when a shard doesn't have any primary tablet elected                                                                                                                                               | VTOrc runs PlannedReparentShard to elect a new primary         |
| `DeadPrimary`                                                                                                                                            | VTOrc detects when the primary tablet is dead                                                                                                                                                                    | VTOrc runs EmergencyReparentShard to elect a different primary |
| `PrimaryTabletUnreachableByQuorum`                                                                                                                       | VTOrc detects when the primary vttablet process is unreachable by a quorum of replica tablets (requires [gossip-based failure detection](#gossip-based-failure-detection))                                      | VTOrc runs EmergencyReparentShard to elect a different primary |
| `IncapacitatedPrimary`                                                                                                                                   | VTOrc detects when the primary tablet is consistently failing health checks but is still network-reachable                                                                                                       | VTOrc runs PlannedReparentShard, falling back to EmergencyReparentShard if that fails |
| `PrimaryIsReadOnly`, `PrimarySemiSyncMustBeSet`, `PrimarySemiSyncMustNotBeSet`                                                                           | VTOrc detects when the primary tablet has configuration issues like being read-only, semi-sync being set or not being set                                                                                        | VTOrc fixes the configurations on the primary.                 |
| `NotConnectedToPrimary`, `ConnectedToWrongPrimary`, `ReplicationStopped`, `ReplicaIsWritable`, `ReplicaSemiSyncMustBeSet`, `ReplicaSemiSyncMustNotBeSet` | VTOrc detects when a replica has configuration issues like not being connected to the primary, connected to the wrong primary, replication stopped, replica being writable, semi-sync being set or not being set | VTOrc fixes the configurations on the replica.                 |
| `StaleTopoPrimary`                                                                                                                                       | VTOrc detects when a tablet still has type PRIMARY in the topology but a newer primary has already been elected. This can happen if a topology update fails during an emergency reparent operation.              | VTOrc demotes the stale primary to a read-only replica, updates its type to REPLICA in the topology, and configures it to replicate from the current primary. |

### Flags

For a full list of supported flags, please look at [VTOrc reference page](../../../reference/programs/vtorc).

### UI, API and Metrics

For information about the UI, API and metrics that VTOrc exports, please consult this [page](../../../reference/vtorc/ui_api_metrics).

### Example invocation of VTOrc

You can bring VTOrc using the following invocation:

```sh
vtorc --topo-implementation etcd2 \
  --topo-global-server-address "localhost:2379" \
  --topo-global-root /vitess/global \
  --cell zone1 \
  --port 15000 \
  --log-dir=${VTDATAROOT}/tmp \
  --recovery-period-block-duration "10m" \
  --instance-poll-time "1s" \
  --topo-information-refresh-duration "30s" \
  --alsologtostderr
 ```

### Cell Awareness

Starting in v24, VTOrc supports the `--cell` flag to specify which cell the VTOrc instance is running in. This flag is optional in v24 but will become required in v25 and later versions.

The `--cell` flag enables VTOrc to be cell-aware, which will be used in future releases for cross-cell problem validation. When specified, VTOrc validates that the cell exists in the topology. If the cell doesn't exist, VTOrc will fail to start. If the flag is not provided in v24, VTOrc will log a warning but continue to operate normally.

### Filtering Tablets

By default, VTOrc monitors all tablets across all cells. You can restrict which tablets it watches using the `--clusters-to-watch` and `--cells-to-watch` flags.

#### Filtering by Keyspace/Shard

The `--clusters-to-watch` flag accepts a comma-separated list of keyspaces or keyspace/shard combinations:

```sh
# Watch all shards in keyspace1 and keyspace2
vtorc --clusters-to-watch "keyspace1,keyspace2" ...

# Watch specific shards
vtorc --clusters-to-watch "keyspace1,keyspace2/-80" ...
```

#### Filtering by Cell

The `--cells-to-watch` flag accepts a comma-separated list of cells. VTOrc will only monitor tablets in those cells:

```sh
# Only watch tablets in zone1 and zone2
vtorc --cells-to-watch "zone1,zone2" ...
```

VTOrc validates that each specified cell exists in the topology. If any cell doesn't exist, VTOrc will fail to start.

#### Combining Filters

When both flags are set, a tablet must match **both** filters to be monitored:

```sh
vtorc --clusters-to-watch "keyspace1" --cells-to-watch "zone1,zone2" ...
```

This configuration makes VTOrc monitor only tablets in `keyspace1` that are located in `zone1` or `zone2`.

When neither flag is set, VTOrc monitors all tablets in the topology.

### Durability Policies

All the failovers that VTOrc performs will be honoring the [durability policies](../../configuration-basic/durability_policy). Please be careful in setting the
desired durability policies for your keyspace because this will affect what situations VTOrc can recover from and what situations will require manual intervention.

### Gossip-Based Failure Detection

Traditional VTOrc failure detection relies on MySQL replication monitoring. If the MySQL process fails, VTOrc detects the replication break and triggers failover. However, when the vttablet process crashes while MySQL remains healthy, the replication connection stays intact and VTOrc does not detect the failure. This leaves the shard in a broken state where the primary vttablet is unreachable but no automatic recovery occurs.

Gossip-based failure detection addresses this limitation by enabling tablets to monitor each other directly. When a quorum of replica tablets agrees that the primary vttablet is unreachable, VTOrc triggers an Emergency Reparent Shard operation.

#### Enabling Gossip

Gossip is configured per-keyspace using the `vtctldclient UpdateGossipConfig` command:

```sh
vtctldclient UpdateGossipConfig --enable --ping-interval=1s --max-update-age=5s --phi-threshold=4 <keyspace>
```

Three components must be configured for gossip to work:

1. **vttablet**: Add `grpc-gossip` to the `--service-map` flag:
   ```sh
   vttablet --service-map 'grpc-queryservice,grpc-tabletmanager,grpc-updatestream,grpc-gossip' ...
   ```

2. **VTOrc**: Set the `--gossip-listen-addr` flag:
   ```sh
   vtorc --gossip-listen-addr ':15100' ...
   ```

3. **Keyspace**: Enable gossip using `UpdateGossipConfig` as shown above.

Configuration changes propagate through the topology service without requiring process restarts.

#### Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--enable` | N/A | Enables gossip for the keyspace. Use `--disable` to turn it off. |
| `--ping-interval` | 1s | How often tablets exchange gossip heartbeats. |
| `--max-update-age` | 5s | Maximum staleness before marking a peer as down. |
| `--phi-threshold` | 4 | Phi-accrual suspicion threshold. Higher values reduce false positives but increase detection latency. |

#### Quorum Requirements

For VTOrc to trigger `PrimaryTabletUnreachableByQuorum`:

- The primary must be marked as `Down` by the gossip protocol (not just `Suspect`)
- A strict majority of non-primary replicas must be `Alive`
- At least 2 alive observers are required
- For small shards (2 or fewer replicas), VTOrc's own health check must also corroborate the failure

#### Multi-Keyspace Considerations

When multiple keyspaces have gossip enabled, they must use consistent configuration values. If conflicting settings are detected, VTOrc refuses to start gossip and logs an error. Resolve this by ensuring all enabled keyspaces use the same `ping-interval`, `max-update-age`, and `phi-threshold` values.

#### Debug Endpoint

Both vttablet and VTOrc expose a `/debug/gossip` HTTP endpoint that returns the current gossip state as JSON. This is useful for debugging connectivity issues or verifying that gossip is functioning correctly.

### Running VTOrc using the Vitess Operator

To find information about deploying VTOrc using Vitess Operator please take a look at this [page](../../../reference/vtorc/running_with_vtop).
