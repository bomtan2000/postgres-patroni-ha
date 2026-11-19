# Install

Order matters: **etcd → Patroni → HAProxy → Keepalived**. Patroni cannot start without a DCS,
and HAProxy health checks need Patroni's REST API.

| Node | IP | etcd | Patroni `failover_priority` | Keepalived |
|---|---|---|---|---|
| node1 | 192.168.63.11 | creates the cluster | 3 | MASTER 100 |
| node2 | 192.168.63.12 | joins | 2 | BACKUP 90 |
| node3 | 192.168.63.13 | joins | 1 | BACKUP 80 |
| VIP | 192.168.63.20 | | | |

OS: Oracle Linux / RHEL 9. Run as root unless the step says otherwise.

## 1. Hosts and firewall (all 3 nodes)

```bash
cat config/hosts >> /etc/hosts

# etcd client/peer, Postgres, Patroni API, HAProxy RW/RO/stats, VRRP
firewall-cmd --permanent --add-port={2379,2380,5432,8008,5000,5001,7000}/tcp
firewall-cmd --permanent --add-rich-rule='rule protocol value="vrrp" accept'
firewall-cmd --reload
```

## 2. Packages (all 3 nodes)

```bash
dnf install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
percona-release setup ppg18

dnf install -y epel-release
dnf install -y python3-click                       # Patroni dependency, from EPEL

dnf install -y percona-postgresql18-server percona-patroni
dnf install -y etcd python3-python-etcd
dnf install -y percona-haproxy keepalived
```

Do **not** run `initdb` or enable `postgresql-18.service`. Patroni initialises the data
directory and starts Postgres itself. A second manager fighting Patroni over the same data dir
is how you get two primaries.

## 3. etcd

The cluster grows one member at a time: node1 starts alone, then node2 and node3 are announced
with `member add` **before** they start.

```bash
# node1
cp config/etcd/node1.conf.yaml /etc/etcd/etcd.conf.yaml
systemctl enable --now etcd
etcdctl --endpoints=http://node1:2379 member list -w table     # node1 started
```

```bash
# node1: announce node2
etcdctl --endpoints=http://node1:2379 member add node2 --peer-urls=http://192.168.63.12:2380
# node2
cp config/etcd/node2.conf.yaml /etc/etcd/etcd.conf.yaml
systemctl enable --now etcd
```

```bash
# node1: announce node3
etcdctl --endpoints=http://node1:2379 member add node3 --peer-urls=http://192.168.63.13:2380
# node3
cp config/etcd/node3.conf.yaml /etc/etcd/etcd.conf.yaml
systemctl enable --now etcd
```

```bash
# any node: 3 members started, 3 endpoints healthy
etcdctl --endpoints=http://node1:2379,http://node2:2379,http://node3:2379 member list -w table
etcdctl --endpoints=http://node1:2379,http://node2:2379,http://node3:2379 endpoint health
```

Port **2379** is for clients (`etcdctl`, Patroni). 2380 is peer-to-peer and does not answer
client commands.

## 4. Patroni + PostgreSQL

All 3 nodes:

```bash
mkdir -p /etc/patroni /home/postgres/archived
chown -R postgres:postgres /etc/patroni /home/postgres/archived

cp config/patroni/node1.yml /etc/patroni/patroni.yml     # node2.yml / node3.yml on the others
chown postgres:postgres /etc/patroni/patroni.yml
chmod 600 /etc/patroni/patroni.yml

grep PATRONI_CONFIG_LOCATION /usr/lib/systemd/system/patroni.service
# must point to /etc/patroni/patroni.yml
```

Start **node1 first** and wait until it is Leader, then node2, then node3. The first Patroni to
take the leader key runs `initdb`. The others clone from it with `pg_basebackup`.

```bash
systemctl daemon-reload
systemctl enable --now patroni      # enable: a rebooted node rejoins the cluster by itself
patronictl -c /etc/patroni/patroni.yml list
```

Expected:

```
+--------+---------------+---------+-----------+----+-----------+
| Member | Host          | Role    | State     | TL | Lag in MB |
+--------+---------------+---------+-----------+----+-----------+
| node1  | 192.168.63.11 | Leader  | running   |  1 |           |
| node2  | 192.168.63.12 | Replica | streaming |  1 |         0 |
| node3  | 192.168.63.13 | Replica | streaming |  1 |         0 |
+--------+---------------+---------+-----------+----+-----------+
```

### Application users

Patroni 4 no longer creates users from `bootstrap.users`. Create them once on the leader:

```bash
sudo -u postgres psql -c "CREATE ROLE admin WITH LOGIN CREATEROLE CREATEDB PASSWORD 'AdminLab@123';"
```

## 5. HAProxy (all 3 nodes)

```bash
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
cp config/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg
haproxy -c -f /etc/haproxy/haproxy.cfg

setsebool -P haproxy_connect_any on      # SELinux: let HAProxy bind 5000/5001/7000 and reach 8008
systemctl enable --now haproxy
```

Stats page: `http://<node>:7000/`. In `primary`, exactly one server is UP. In `standbys`, two are UP.

## 6. Keepalived (all 3 nodes)

```bash
# enable_script_security runs track scripts as this user; most distros do not create it
id keepalived_script || useradd -r -M -s /sbin/nologin keepalived_script

mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
cp config/keepalived/node1.conf /etc/keepalived/keepalived.conf   # node2.conf / node3.conf on the others
chmod 600 /etc/keepalived/keepalived.conf
systemctl enable --now keepalived

ip -4 addr show | grep 192.168.63.20      # on node1 only
```

## 7. End to end

```bash
psql -h 192.168.63.20 -p 5000 -U postgres -c "select pg_is_in_recovery(), inet_server_addr();"   # f, leader IP
psql -h 192.168.63.20 -p 5001 -U postgres -c "select pg_is_in_recovery(), inet_server_addr();"   # t, a replica IP
```

Next: [failover-test.md](failover-test.md).
