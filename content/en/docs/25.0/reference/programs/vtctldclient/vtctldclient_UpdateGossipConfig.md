---
title: UpdateGossipConfig
series: vtctldclient
---
## vtctldclient UpdateGossipConfig

Update the gossip protocol configuration for all tablets in the given keyspace (across all cells)

```
vtctldclient UpdateGossipConfig [--enable|--disable] [--phi-threshold=<float64>] [--ping-interval=<duration>] [--max-update-age=<duration>] <keyspace>
```

### Description

Configures the gossip protocol for tablet-to-tablet failure detection within a keyspace. Gossip enables VTOrc to detect when the primary vttablet process becomes unreachable, even if MySQL replication remains healthy.

For gossip to work, vttablets must include `grpc-gossip` in their `--service-map` flag, and VTOrc must be started with `--gossip-listen-addr`.

Configuration changes are stored in the topology and propagated to tablets via SrvKeyspace watches. No process restarts are required after changing gossip settings.

### Options

```
      --disable              Disable gossip for this keyspace
      --enable               Enable gossip for this keyspace
  -h, --help                 help for UpdateGossipConfig
      --max-update-age string   Max staleness before marking peer down. Must be a positive duration when specified; omit to use the default (5s) on create or preserve the existing value on update
      --phi-threshold float  Phi-accrual suspicion threshold. Must be > 0 when specified; omit to use the default (4) on create or preserve the existing value on update
      --ping-interval string Gossip exchange interval. Must be a positive duration when specified; omit to use the default (1s) on create or preserve the existing value on update
```

### Options inherited from parent commands

```
      --action-timeout duration              timeout to use for the command (default 1h0m0s)
      --compact                              use compact format for otherwise verbose outputs
      --server string                        server to use for the connection (required)
      --topo-global-root string              the path of the global topology data in the global topology server (default "/vitess/global")
      --topo-global-server-address strings   the address of the global topology server(s) (default [localhost:2379])
      --topo-implementation string           the topology implementation to use (default "etcd2")
```

### Examples

Enable gossip with default settings:
```sh
vtctldclient UpdateGossipConfig --enable commerce
```

Enable gossip with custom tuning:
```sh
vtctldclient UpdateGossipConfig --enable --ping-interval=500ms --max-update-age=3s --phi-threshold=3.5 commerce
```

Disable gossip:
```sh
vtctldclient UpdateGossipConfig --disable commerce
```

Update only the phi threshold (preserves other settings):
```sh
vtctldclient UpdateGossipConfig --phi-threshold=5 commerce
```

### SEE ALSO

* [vtctldclient](../)	 - Executes a cluster management command on the remote vtctld server or alternatively as a standalone binary using --server=internal.

