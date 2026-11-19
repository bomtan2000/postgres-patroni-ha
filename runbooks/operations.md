# Operations

Every command runs on any node: `patronictl` talks to etcd, not to the local Postgres.

```bash
alias pctl='patronictl -c /etc/patroni/patroni.yml'
```

## Cluster state

```bash
pctl list                       # roles, timeline, lag
pctl history                    # every failover / switchover with its LSN
curl -s http://node1:8008/patroni | python3 -m json.tool
```

## Planned switchover

Moves the leader with no data loss: Patroni waits for the candidate to catch up first.

```bash
pctl switchover --leader node2 --candidate node1 --force
pctl list
```

Clients on port 5000 are disconnected from the old leader (`on-marked-down shutdown-sessions`)
and reconnect to the new one within a few seconds.

## Change a PostgreSQL parameter

`bootstrap.dcs` in `patroni.yml` is read only once, when the cluster is created. After that the
source of truth is etcd:

```bash
pctl edit-config                                   # opens the DCS config in $EDITOR
pctl edit-config -p 'max_connections=300' --force  # or non-interactive
pctl list                                          # "Pending restart" column if a restart is needed
pctl restart pg-cluster --pending                  # rolling restart, replicas first
```

Editing `postgresql.conf` by hand is pointless: Patroni rewrites it.

## Maintenance mode

Stop Patroni from acting (no failover) while you work on Postgres yourself:

```bash
pctl pause
# ... maintenance ...
pctl resume
```

While paused, a dead leader is **not** replaced. Do not leave it paused.

## Rebuild a broken replica

```bash
pctl reinit pg-cluster node3 --force      # wipes node3's data dir, pg_basebackup from the leader
```

## Take a node out of leader elections

Set the tag in that node's `/etc/patroni/patroni.yml`, then reload:

```yaml
tags:
  nofailover: true      # never promoted (e.g. weaker hardware, DR copy)
  noloadbalance: true   # /replica returns 503: HAProxy stops sending reads to it
```

```bash
pctl reload pg-cluster node3
```

## etcd health

```bash
etcdctl --endpoints=http://node1:2379,http://node2:2379,http://node3:2379 endpoint status -w table
etcdctl --endpoints=http://node1:2379,http://node2:2379,http://node3:2379 endpoint health
```

Losing **one** etcd member is fine (2/3 quorum). Losing **two** means no quorum: Patroni can no
longer renew the leader key, and the leader **demotes itself to read-only** after `ttl`. That is
deliberate: no DCS, no proof you are still the only leader.
