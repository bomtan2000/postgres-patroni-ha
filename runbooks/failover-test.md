# Failover test

Prove three things: data replicates, a dead leader is replaced automatically, and the old
leader comes back as a replica by itself.

## 1. Replication

Write through the VIP on the **read/write** port:

```bash
psql -h 192.168.63.20 -p 5000 -U postgres
```

```sql
CREATE TABLE ha_test (team text, id integer);
INSERT INTO ha_test VALUES ('devops', 1), ('platform', 2);
```

Read it on **node3 directly** (port 5432, not through HAProxy). Going through `node3:5000`
would just be routed back to the leader and prove nothing about node3:

```bash
psql -h node3 -p 5432 -U postgres -c "select pg_is_in_recovery(), * from ha_test;"
# t | devops | 1
# t | platform | 2
```

Or through the read-only port, which only routes to replicas:

```bash
psql -h 192.168.63.20 -p 5001 -U postgres -c "select inet_server_addr(), count(*) from ha_test group by 1;"
```

## 2. Kill the leader

```bash
patronictl -c /etc/patroni/patroni.yml list     # node1 = Leader
# on node1:
shutdown -h now
```

In another terminal, keep writing through the VIP to see the outage window:

```bash
while true; do
  psql -h 192.168.63.20 -p 5000 -U postgres -Atc "insert into ha_test values ('probe', 0) returning now();" 2>&1 | tail -1
  sleep 1
done
```

## 3. Watch the new leader (node2)

```bash
patronictl -c /etc/patroni/patroni.yml list     # node2 = Leader, TL+1 (new timeline)
ip -4 addr show | grep 192.168.63.20           # VIP moved to node2 (node1 is down)
```

What happens, in order:

1. node1 stops renewing the leader key in etcd. It expires after at most `ttl` = 30 s.
2. node2 and node3 compare WAL positions. Only members within `maximum_lag_on_failover` can
   run. The one with the most WAL wins. On a tie, `failover_priority` decides: node2 (2) beats node3 (1).
3. node2 promotes. node3 re-points its replication to node2.
4. HAProxy on every node sees `GET /primary` → 200 on node2 within `inter 3s × rise 2`.
5. Keepalived moves the VIP because node1 is gone. That is the only reason it moves here:
   any live HAProxy would route correctly.

Expected write outage: up to ~30-40 s (TTL + promote + health checks).

## 4. Bring node1 back

Power node1 on. `patroni` is enabled, so it starts, sees node2 holds the leader key, runs
`pg_rewind` against node2 and follows it:

```bash
patronictl -c /etc/patroni/patroni.yml list     # node1 = Replica, streaming
```

node1 stays a replica. Patroni does not fail back on its own. If you want node1 as leader
again, do a planned [switchover](operations.md#planned-switchover).

## Cleanup

```sql
DROP TABLE ha_test;
```
