# etcd Learning PoC on Ubuntu 24.04 (WSL2)

## Overview

This repository contains hands-on learning notes and Proof of Concept (PoC) for **etcd** on Ubuntu 24.04 running under WSL2 on Windows 11.

### What does "etcd" mean?

The name **etcd** comes from two Unix conventions:

- **`/etc`** — the Unix/Linux directory that stores system-wide configuration files
- **`d`** — stands for **distributed**

> **etcd = distributed `/etc`**
> A distributed store for configuration and coordination data — just like `/etc` is where a single Linux machine stores its config.

This naming was intentional by the etcd authors: etcd does for a cluster what `/etc` does for a single machine.

As a **PostgreSQL Developer, DBA, and Architect**, understanding etcd is essential because modern PostgreSQL HA stacks — Patroni, Kubernetes operators like CloudNativePG — rely on etcd as the distributed coordination backbone.

### Professional Context

This PoC is part of a deliberate skill development journey across three PostgreSQL roles:

| Role | Focus Area |
| ---- | ---------- |
| **PostgreSQL Developer** | Query design, schema modeling, performance tuning |
| **PostgreSQL DBA** | Backup, recovery, monitoring, maintenance |
| **PostgreSQL Architect** | HA design, replication topology, cloud-native stacks |

A PostgreSQL architect who understands etcd can design, troubleshoot, and reason about HA systems at a deeper level than someone who treats it as a black box. This repository is a **living PoC portfolio** — documented hands-on explorations that go beyond theoretical knowledge.

---

## Environment

| Component | Version            |
| --------- | ------------------ |
| OS        | Ubuntu 24.04.4 LTS |
| Platform  | WSL2 (Windows 11)  |
| etcd      | 3.6.4              |
| etcdctl   | 3.6.4              |

---

## Why Learn etcd?

Traditional databases store business data:

- Customers, Orders, Payments

etcd stores **cluster state and coordination metadata**:

- Who is the leader?
- Which nodes are healthy?
- Service discovery information
- Configuration values

> **PostgreSQL stores application data. etcd stores cluster coordination data.**

---

## Learning Path

```
Phase 1 (Basics)          Phase 2 (Cluster)          Phase 3 (Production)
──────────────────         ──────────────────          ──────────────────
Single node                3-node cluster              Real Patroni
KV ops                     Raft / elections            PostgreSQL HA
Watches / leases           Compaction / revisions      Failover testing
Snapshots                  Transactions (CAS)          PgBouncer + Patroni
Patroni simulation         TLS + auth
```

---

## Installation

Download and extract:

```bash
cd /tmp

wget https://github.com/etcd-io/etcd/releases/download/v3.6.4/etcd-v3.6.4-linux-amd64.tar.gz

tar -xvf etcd-v3.6.4-linux-amd64.tar.gz

cd etcd-v3.6.4-linux-amd64
```

Start etcd (single node):

```bash
./etcd
```

---

## Verify Installation

```bash
./etcd --version
./etcdctl version
```

Check cluster health:

```bash
./etcdctl endpoint health
```

Check cluster status:

```bash
./etcdctl endpoint status -w table
```

---

## Key-Value Operations

Insert values:

```bash
./etcdctl put postgres/leader node1
./etcdctl put postgres/node1 primary
./etcdctl put postgres/node2 replica
```

Retrieve a value:

```bash
./etcdctl get postgres/leader
```

List all keys:

```bash
./etcdctl get "" --prefix
```

List keys only:

```bash
./etcdctl get "" --prefix --keys-only
```

---

## Prefix Queries

Retrieve all PostgreSQL-related entries:

```bash
./etcdctl get postgres/ --prefix
```

Example output:

```
postgres/leader
node1

postgres/node1
primary

postgres/node2
replica
```

---

## Watch Operations

Terminal 1 — start watching:

```bash
./etcdctl watch postgres/leader
```

Terminal 2 — trigger a change:

```bash
./etcdctl put postgres/leader node2
```

The watcher immediately detects the change. This is the mechanism Patroni and Kubernetes use to monitor cluster state changes in real time.

---

## Revision History and Compaction

Every `put` increments a global revision counter. You can read past state — a concept familiar to PostgreSQL DBAs via MVCC:

```bash
# Check current revision
./etcdctl endpoint status -w table

# Read a key at a specific past revision
./etcdctl get postgres/leader --rev=3

# Compact old revisions (discard history before revision 5)
./etcdctl compact 5

# Defragment after compaction
./etcdctl defrag
```

> **Insight:** etcd's revision history is analogous to PostgreSQL's MVCC — multiple versions of a row exist until vacuumed. `compact` in etcd is the equivalent of `VACUUM`.

---

## Lease (TTL)

Create a lease:

```bash
./etcdctl lease grant 120
```

Attach a key to the lease:

```bash
./etcdctl put temp/status running --lease=<LEASE_ID>
```

After the TTL expires, the key is automatically removed.

### Lease Keep-Alive (Heartbeat Pattern)

This is how Patroni heartbeats work internally:

```bash
# Terminal 1 — simulate a node heartbeat
./etcdctl lease keep-alive <LEASE_ID>

# Kill Terminal 1 (simulate node crash)
# The key disappears automatically → triggers failover detection
```

> **Insight:** Patroni's leader key is held with a lease and renewed every `ttl/2` seconds. If the primary crashes, the lease expires, the key disappears, and replicas detect the absence via a watch — then a new leader is elected.

---

## Transactions (Compare-And-Swap)

This is the actual primitive behind safe leader election. Only one node can win:

```bash
./etcdctl txn <<EOF
compares:
value("patroni/cluster/leader") = "pg1"

success requests:
put patroni/cluster/leader pg2

failure requests:
get patroni/cluster/leader
EOF
```

- If the current leader is `pg1`, it is updated to `pg2` (success path).
- If not, the current value is returned (failure path).

> **Insight:** This is equivalent to PostgreSQL's `SELECT ... FOR UPDATE` + conditional `UPDATE`. It prevents split-brain during leader transitions.

---

## Snapshot Backup

Create a snapshot:

```bash
./etcdctl snapshot save backup.db
```

Verify snapshot:

```bash
./etcdutl snapshot status backup.db -w table
```

---

## Maintenance

Check alarms:

```bash
./etcdctl alarm list
```

Defragment database:

```bash
./etcdctl defrag
```

Check endpoint hash:

```bash
./etcdctl endpoint hashkv
```

---

## Patroni-Style Simulation

Create cluster metadata:

```bash
./etcdctl put patroni/cluster/leader pg1

./etcdctl put patroni/cluster/member/pg1 running
./etcdctl put patroni/cluster/member/pg2 running
./etcdctl put patroni/cluster/member/pg3 running
```

View cluster state:

```bash
./etcdctl get patroni/ --prefix
```

Simulate failover (leader change):

```bash
./etcdctl put patroni/cluster/leader pg2
```

---

## Three-Node etcd Cluster on a Single WSL2 Machine

> **Yes — you can simulate a 3-node etcd cluster on a single Windows 11 WSL2 machine.**
> Each node runs as a separate process on different ports. This is not production-grade but is sufficient to observe Raft consensus, leader election, and quorum behavior.

### Port Layout

| Node  | Client Port | Peer Port |
| ----- | ----------- | --------- |
| etcd1 | 2379        | 2380      |
| etcd2 | 2381        | 2382      |
| etcd3 | 2383        | 2384      |

---

### Step 1 — Prepare Data Directories

```bash
mkdir -p /tmp/etcd-cluster/etcd1
mkdir -p /tmp/etcd-cluster/etcd2
mkdir -p /tmp/etcd-cluster/etcd3
```

---

### Step 2 — Start Node 1

Open Terminal 1:

```bash
cd /tmp/etcd-v3.6.4-linux-amd64

./etcd \
  --name etcd1 \
  --data-dir /tmp/etcd-cluster/etcd1 \
  --listen-client-urls http://127.0.0.1:2379 \
  --advertise-client-urls http://127.0.0.1:2379 \
  --listen-peer-urls http://127.0.0.1:2380 \
  --initial-advertise-peer-urls http://127.0.0.1:2380 \
  --initial-cluster etcd1=http://127.0.0.1:2380,etcd2=http://127.0.0.1:2382,etcd3=http://127.0.0.1:2384 \
  --initial-cluster-token etcd-poc-cluster \
  --initial-cluster-state new
```

---

### Step 3 — Start Node 2

Open Terminal 2:

```bash
cd /tmp/etcd-v3.6.4-linux-amd64

./etcd \
  --name etcd2 \
  --data-dir /tmp/etcd-cluster/etcd2 \
  --listen-client-urls http://127.0.0.1:2381 \
  --advertise-client-urls http://127.0.0.1:2381 \
  --listen-peer-urls http://127.0.0.1:2382 \
  --initial-advertise-peer-urls http://127.0.0.1:2382 \
  --initial-cluster etcd1=http://127.0.0.1:2380,etcd2=http://127.0.0.1:2382,etcd3=http://127.0.0.1:2384 \
  --initial-cluster-token etcd-poc-cluster \
  --initial-cluster-state new
```

---

### Step 4 — Start Node 3

Open Terminal 3:

```bash
cd /tmp/etcd-v3.6.4-linux-amd64

./etcd \
  --name etcd3 \
  --data-dir /tmp/etcd-cluster/etcd3 \
  --listen-client-urls http://127.0.0.1:2383 \
  --advertise-client-urls http://127.0.0.1:2383 \
  --listen-peer-urls http://127.0.0.1:2384 \
  --initial-advertise-peer-urls http://127.0.0.1:2384 \
  --initial-cluster etcd1=http://127.0.0.1:2380,etcd2=http://127.0.0.1:2382,etcd3=http://127.0.0.1:2384 \
  --initial-cluster-token etcd-poc-cluster \
  --initial-cluster-state new
```

---

### Step 5 — Verify Cluster Health

Open Terminal 4:

```bash
cd /tmp/etcd-v3.6.4-linux-amd64

# Check all endpoints
./etcdctl \
  --endpoints=http://127.0.0.1:2379,http://127.0.0.1:2381,http://127.0.0.1:2383 \
  endpoint health

# Check status (shows which node is the leader)
./etcdctl \
  --endpoints=http://127.0.0.1:2379,http://127.0.0.1:2381,http://127.0.0.1:2383 \
  endpoint status -w table
```

Expected output:

```
+--------------------+------------------+---------+---------+-----------+...
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER |...
+--------------------+------------------+---------+---------+-----------+...
| http://...:2379    | xxxxxxxxxxxxxxxx |  3.6.4  |  20 kB  |   false   |...
| http://...:2381    | xxxxxxxxxxxxxxxx |  3.6.4  |  20 kB  |   true    |...
| http://...:2383    | xxxxxxxxxxxxxxxx |  3.6.4  |  20 kB  |   false   |...
+--------------------+------------------+---------+---------+-----------+...
```

---

### Step 6 — Write and Read Across the Cluster

Write via node 1, read via node 3 — data is replicated automatically:

```bash
# Write to etcd1
./etcdctl --endpoints=http://127.0.0.1:2379 put postgres/leader pg1

# Read from etcd3
./etcdctl --endpoints=http://127.0.0.1:2383 get postgres/leader
```

---

### Step 7 — Simulate Leader Failure (Raft Election)

Find which node is the current leader from the `endpoint status` output.
Kill that terminal process (`Ctrl+C`).

Then observe re-election:

```bash
# Use the two surviving nodes
./etcdctl \
  --endpoints=http://127.0.0.1:2379,http://127.0.0.1:2383 \
  endpoint status -w table
```

A new leader is elected automatically within a few seconds.

> **Insight:** With 3 nodes, the cluster tolerates **1 node failure** (quorum = 2 of 3). Kill 2 nodes and the cluster becomes read-only — it cannot accept writes because quorum is lost. This is Raft consensus in action.

---

### Step 8 — Watch Leader Election in Real Time

Terminal A — watch the leader key:

```bash
./etcdctl --endpoints=http://127.0.0.1:2379 watch patroni/cluster/leader
```

Terminal B — kill the leader node process, then write a new leader:

```bash
./etcdctl --endpoints=http://127.0.0.1:2381 put patroni/cluster/leader pg2
```

Terminal A immediately shows the PUT event.

---

### Step 9 — Cleanup

```bash
# Kill all etcd processes
pkill -f etcd

# Remove data directories
rm -rf /tmp/etcd-cluster
```

---

## Quorum Reference

| Cluster Size | Tolerated Failures | Quorum Required |
| ------------ | ------------------ | --------------- |
| 1 node       | 0                  | 1               |
| 3 nodes      | 1                  | 2               |
| 5 nodes      | 2                  | 3               |

> A 3-node cluster is the **minimum recommended** for production. It tolerates 1 failure without losing availability.

---


## References

- [etcd Official Documentation](https://etcd.io/docs/)
- [PostgreSQL High Availability Resources](https://www.postgresql.org/docs/current/high-availability.html)

---

## Disclaimer

This repository is intended for learning, experimentation, and Proof of Concept purposes only. Not suitable for production use without proper security hardening (TLS, authentication, firewall rules).
