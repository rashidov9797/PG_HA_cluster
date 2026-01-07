# 🐘 PostgreSQL 18 HA on Rocky Linux 8 (Patroni + ETCD + HAProxy + pgBackRest)


<img width="966" height="843" alt="image" src="https://github.com/user-attachments/assets/52e47d13-c291-4002-88ad-c1a41aae9200" />


This repository documents my portfolio project: a **production-style PostgreSQL 18 High Availability (HA)** architecture using **Patroni + ETCD** for leader election/failover, **HAProxy** for RW/RO traffic routing, and **pgBackRest** for backups and continuous WAL archiving.

All configuration examples in this repo are **sanitized** (IPs/passwords replaced with placeholders). The goal is to provide a clear, reproducible blueprint and an operations-focused manual.

---


### 0) Patroni Cluster Topology (Proof)

Command:
patronictl -c /etc/patroni/patroni.yml list

<img width="1239" height="280" alt="image" src="https://github.com/user-attachments/assets/0e584e75-4cdd-4240-a1f1-78246eab0955" />


---

## 🏗️ Architecture & Roles

### 1. Database Layer (`node1`, `node2`, `node3`)
Each DB node runs:
- **PostgreSQL 18** (Database Engine)
- **Patroni** (HA Controller + REST API on `8008`)
- **ETCD** (Distributed Consensus Store on `2379`)

**Logic:**
- One node is elected **Leader** (Read/Write).
- Other nodes become **Replicas** (Read-Only, Streaming Replication).
- ETCD ensures quorum to prevent "Split-Brain" scenarios.

### 2. Traffic Management (`haproxy`)
HAProxy exposes two stable endpoints for applications:
- **Port `5000` (RW)** → Routes strictly to the **Leader**.
- **Port `5001` (RO)** → Load balances across **Replicas**.

**Logic:**
- HAProxy checks Patroni's API (`:8008/master` and `:8008/replica`) to determine routing dynamically.

### 3. Disaster Recovery (`pgbackrest`)
A dedicated repository server storing:
- **Full / Differential Backups**
- **Continuous WAL Archiving** (Point-in-Time Recovery ready)

---

## 🌍 Topology (Sanitized)

| Hostname | IP Address | Role | OS |
| :--- | :--- | :--- | :--- |
| **node1** | `{{NODE1_IP}}` | Primary DB | Rocky Linux 8 |
| **node2** | `{{NODE2_IP}}` | Standby DB | Rocky Linux 8 |
| **node3** | `{{NODE3_IP}}` | Standby DB | Rocky Linux 8 |
| **haproxy** | `{{HAPROXY_IP}}` | Load Balancer | Rocky Linux 8 |
| **pgbackrest** | `{{PGBACKREST_IP}}` | Backup Repo | Rocky Linux 8 |

### 🔌 Ports Overview
- **5432:** PostgreSQL
- **8008:** Patroni REST API
- **2379:** ETCD Client
- **2380:** ETCD Peer
- **5000:** HAProxy RW (Write)
- **5001:** HAProxy RO (Read)

---

## 🛠️ Installation Guide (Step-by-Step)


### 2. Install Dependencies
Run on **all DB nodes** (node1, node2, node3).

    sudo dnf install -y python3-pip gcc git
    sudo dnf install -y postgresql18-server postgresql18-devel   # package name may vary by repo
    sudo dnf install -y etcd

    sudo pip3 install --upgrade pip
    sudo pip3 install "patroni[etcd]" psycopg2-binary


### 3. Configure Firewall
Run on each server as appropriate.

DB nodes (node1/node2/node3):

    sudo firewall-cmd --permanent --add-port=5432/tcp
    sudo firewall-cmd --permanent --add-port=8008/tcp
    sudo firewall-cmd --permanent --add-port=2379/tcp
    sudo firewall-cmd --permanent --add-port=2380/tcp
    sudo firewall-cmd --reload

HAProxy node:

    sudo firewall-cmd --permanent --add-port=5000/tcp
    sudo firewall-cmd --permanent --add-port=5001/tcp
    sudo firewall-cmd --reload

PgBackRest repo node (SSH):

    sudo firewall-cmd --permanent --add-service=ssh
    sudo firewall-cmd --reload


---

## Phase 2: ETCD Cluster Setup
Run on node1, node2, node3.

ETCD config file example: /etc/etcd/etcd.conf
(Example for node1 — change ETCD_NAME and IPs for node2/node3)

    ETCD_NAME="node1"
    ETCD_DATA_DIR="/var/lib/etcd/default.etcd"

    ETCD_LISTEN_PEER_URLS="http://{{NODE1_IP}}:2380"
    ETCD_LISTEN_CLIENT_URLS="http://{{NODE1_IP}}:2379,http://127.0.0.1:2379"

    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://{{NODE1_IP}}:2380"
    ETCD_ADVERTISE_CLIENT_URLS="http://{{NODE1_IP}}:2379"

    ETCD_INITIAL_CLUSTER="node1=http://{{NODE1_IP}}:2380,node2=http://{{NODE2_IP}}:2380,node3=http://{{NODE3_IP}}:2380"
    ETCD_INITIAL_CLUSTER_STATE="new"
    ETCD_INITIAL_CLUSTER_TOKEN="pg-ha-token"

Start ETCD + verify quorum:

    sudo systemctl enable --now etcd
    etcdctl member list
    etcdctl endpoint status --write-out=table


---

## Phase 3: Patroni Configuration
Run on node1, node2, node3.

Patroni config file: /etc/patroni/postgres.yml
(Example for node1 — update name/restapi/connect_address values per node)

    scope: postgres-ha
    namespace: /db/
    name: node1

    restapi:
      listen: {{NODE1_IP}}:8008
      connect_address: {{NODE1_IP}}:8008

    etcd:
      host: 127.0.0.1:2379

    bootstrap:
      dcs:
        ttl: 30
        loop_wait: 10
        retry_timeout: 10
        maximum_lag_on_failover: 1048576
        postgresql:
          use_pg_rewind: true
          parameters:
            wal_level: replica
            hot_standby: "on"
            max_wal_senders: 10
            max_replication_slots: 10
            archive_mode: "on"
            archive_command: "pgbackrest --stanza=main archive-push %p"
      initdb:
        - encoding: UTF8
        - data-checksums

    postgresql:
      listen: 0.0.0.0:5432
      connect_address: {{NODE1_IP}}:5432
      data_dir: /data/pgdata
      authentication:
        superuser:
          username: postgres
          password: "{{CHANGE_ME_PASSWORD}}"
        replication:
          username: replicator
          password: "{{CHANGE_ME_PASSWORD}}"

Start Patroni + verify cluster:

    sudo systemctl enable --now patroni
    patronictl -c /etc/patroni/postgres.yml list


---

## Phase 4: HAProxy Configuration
Run on HAProxy node.

HAProxy config file: /etc/haproxy/haproxy.cfg

    global
        log         127.0.0.1 local2
        maxconn     4000
        daemon

    defaults
        mode                    tcp
        log                     global
        option                  tcplog
        timeout connect         10s
        timeout client          1m
        timeout server          1m

    # --- FRONTEND: WRITE (Leader) ---
    frontend postgres_rw
        bind *:5000
        default_backend pg_primary

    backend pg_primary
        option httpchk GET /master
        http-check expect status 200
        default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
        server node1 {{NODE1_IP}}:5432 maxconn 100 check port 8008
        server node2 {{NODE2_IP}}:5432 maxconn 100 check port 8008
        server node3 {{NODE3_IP}}:5432 maxconn 100 check port 8008

    # --- FRONTEND: READ (Replicas) ---
    frontend postgres_ro
        bind *:5001
        default_backend pg_replicas

    backend pg_replicas
        option httpchk GET /replica
        http-check expect status 200
        balance roundrobin
        default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
        server node1 {{NODE1_IP}}:5432 maxconn 100 check port 8008
        server node2 {{NODE2_IP}}:5432 maxconn 100 check port 8008
        server node3 {{NODE3_IP}}:5432 maxconn 100 check port 8008

Start HAProxy:

    sudo systemctl enable --now haproxy


## ✅ Proof: RW vs RO Routing (pgAdmin Test)

This section proves that HAProxy correctly routes:
- **Port `5000` → Leader (Read/Write)**
- **Port `5001` → Replicas (Read-Only, load balanced)**



SELECT
  now() AS time,
  inet_server_addr() AS server_ip,
  inet_server_port() AS server_port,
  pg_is_in_recovery() AS is_replica;


<img width="1338" height="646" alt="image" src="https://github.com/user-attachments/assets/5728e511-1927-4ca0-b2bd-c9714ad04bae" />


<img width="1238" height="561" alt="image" src="https://github.com/user-attachments/assets/3ff13207-304a-4ce7-a5c8-5ff5e17c9f95" />




---


## Phase 5: pgBackRest Configuration (Repository Server + DB Nodes)

Goal:
- A dedicated repo server stores backups and WAL archives at: /data/pgbackrest
- DB nodes push WAL to the repo via pgBackRest (usually over SSH)
- Repo server can run full/diff backups by connecting to DB nodes

Placeholders:
- {{PGBACKREST_IP}}  = pgBackRest repo server IP
- {{NODE1_IP}}, {{NODE2_IP}}, {{NODE3_IP}} = DB node IPs
- PGDATA on DB nodes = /data/pgdata
- Repo path on repo server = /data/pgbackrest


----------------------------------------------------------------------
A) Repo Server (pgbackrest host) Setup
----------------------------------------------------------------------


1) Install packages (on repo server)

    sudo dnf install -y pgbackrest openssh-server
    sudo systemctl enable --now sshd

2) Create repository directory (on repo server)

    sudo mkdir -p /data/pgbackrest
    sudo chown -R postgres:postgres /data/pgbackrest
    sudo chmod 750 /data/pgbackrest

3) Configure pgBackRest on repo server

File: /etc/pgbackrest/pgbackrest.conf   (some installs use /etc/pgbackrest.conf)

    [global]
    repo1-path=/data/pgbackrest
    repo1-type=posix
    repo1-retention-full=2
    repo1-retention-diff=7
    repo1-retention-archive=14
    start-fast=y
    process-max=4
    log-level-console=info
    log-level-file=detail

    [pg-ha-cluster]
    pg1-host={{NODE1_IP}}
    pg1-host-user=postgres
    pg1-path=/data/pgdata

    pg2-host={{NODE2_IP}}
    pg2-host-user=postgres
    pg2-path=/data/pgdata

    pg3-host={{NODE3_IP}}
    pg3-host-user=postgres
    pg3-path=/data/pgdata

Note:
- stanza name here is: pg-ha-cluster
- pg1/pg2/pg3 represent DB nodes (Patroni cluster members)

4) SSH trust (repo server -> DB nodes)
pgBackRest will SSH from repo server to DB nodes. Ensure the repo server can SSH to each DB node as user "postgres"
WITHOUT password prompts.

On repo server (as postgres), generate key:

    ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

Copy key to each DB node (repeat for node1/node2/node3):

    ssh-copy-id postgres@{{NODE1_IP}}
    ssh-copy-id postgres@{{NODE2_IP}}
    ssh-copy-id postgres@{{NODE3_IP}}

Test SSH (must not ask password / host verification):

    ssh -o BatchMode=yes postgres@{{NODE1_IP}} "hostname"
    ssh -o BatchMode=yes postgres@{{NODE2_IP}} "hostname"
    ssh -o BatchMode=yes postgres@{{NODE3_IP}} "hostname"

If you see "Host key verification failed", fix known_hosts:
- run a normal ssh once and accept host key
- or remove the old key from ~/.ssh/known_hosts for that IP

5) Create the stanza (run on repo server)
This registers the cluster and validates connectivity.

    pgbackrest --stanza=pg-ha-cluster stanza-create

6) Validate repository and cluster (run on repo server)

    pgbackrest --stanza=pg-ha-cluster check
    pgbackrest --stanza=pg-ha-cluster info





<img width="1193" height="649" alt="image" src="https://github.com/user-attachments/assets/f449b91c-d588-4622-b329-5efb41651e0c" />


----------------------------------------------------------------------
B) DB Nodes Setup (node1/node2/node3)
----------------------------------------------------------------------

1) Install pgBackRest on each DB node

    sudo dnf install -y pgbackrest

2) Ensure DB nodes can SSH to repo server (for WAL archive-push)
Archive-push typically runs on DB nodes and connects to repo server via SSH.
So each DB node should SSH to pgbackrest server as postgres user without password prompts.

On EACH DB node (as postgres), generate key:

    ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

Copy DB node key to repo server (repeat on each DB node):

    ssh-copy-id postgres@{{PGBACKREST_IP}}

Test from each DB node:

    ssh -o BatchMode=yes postgres@{{PGBACKREST_IP}} "hostname"

3) Configure archive_command (Patroni/PostgreSQL parameter)
In Patroni config, ensure archive is enabled and points to the stanza name:

    archive_mode: "on"
    archive_command: "pgbackrest --stanza=pg-ha-cluster archive-push %p"

After reload/restart (Patroni-managed), verify that PostgreSQL is archiving:
- on Leader node, check logs for successful archive-push
- confirm on repo server that archive directory is filling over time

4) Quick WAL archive validation (on Leader DB node)
Force a WAL switch:

    psql -U postgres -d postgres -c "select pg_switch_wal();"

Then on repo server check:

    pgbackrest --stanza=pg-ha-cluster info





NOTE:
- If DIFF/INCR fails, first ensure a FULL backup exists:
    pgbackrest --stanza=pg-ha-cluster info

