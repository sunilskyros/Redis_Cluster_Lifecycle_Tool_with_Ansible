# Redis Cluster Lifecycle Tool

This submission builds a six-node Redis Cluster lifecycle tool for the DevOps Engineer assignment. The `redis-tool` CLI performs prerequisite checks, manages the local SSH container infrastructure, and orchestrates Ansible playbooks for provisioning, data seeding, status, rolling upgrade, and full verification.

## Project Layout

```text
submission/
├── redis-tool
├── ansible/
│   ├── ansible.cfg
│   ├── group_vars/all.yml
│   ├── inventory/hosts.ini
│   ├── playbooks/
│   └── roles/redis/
├── infra/
│   ├── Containerfile
│   └── compose.yml
├── logs/
├── output/
└── README.md
```

## Prerequisites

`redis-tool` checks these before running any command:

- Podman or Docker. If both are installed, Podman is preferred.
- Compose support: `podman-compose`, `podman compose`, `docker compose`, or `docker-compose`.
- Ansible 2.14+ through `ansible-playbook`.
- `ssh-keygen` for `./redis-tool infra up`.

Install references:

- Podman: https://podman.io/docs/installation
- Docker Engine: https://docs.docker.com/engine/install/
- Ansible: `pip install ansible` or your OS package manager

## Bring Up Infrastructure

From this `submission` directory:

```bash
chmod +x ./redis-tool
./redis-tool infra up
```

`infra up` creates a local SSH key under `infra/ssh/`, writes `authorized_keys`, builds an Ubuntu 22.04 SSH image, and starts six containers. The containers have static cluster IPs `10.10.0.11` through `10.10.0.16` on the compose network. Ansible connects through fixed localhost SSH ports `2221` through `2226`, which works on Linux and macOS container runtimes while Redis itself announces the static container IPs required by the assignment.

To stop the environment:

```bash
./redis-tool infra down
```

## Commands

Provision Redis 7.0.15 as a 3-master, 3-replica cluster:

```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

Seed deterministic data:

```bash
./redis-tool data seed --keys 1000
```

Verify deterministic data:

```bash
./redis-tool data verify --keys 1000
```

Print cluster status:

```bash
./redis-tool status
```

Perform the rolling upgrade:

```bash
./redis-tool upgrade --target-version 7.2.6 --strategy rolling --keys 1000
```

Run full verification:

```bash
./redis-tool verify --full --keys 1000
```

Each command writes a terminal transcript to `output/` and structured JSONL events to `logs/`.

## Rolling Upgrade Strategy

The upgrade playbook implements the required no-downtime strategy:

1. Pre-flight checks verify `cluster_state:ok`, node reachability, version drift from the target version, and deterministic data integrity.
2. Replicas are upgraded first with `serial: 1`. Each replica is stopped, upgraded to the target Redis source version, restarted with the same cluster configuration, checked for `master_link_status:up`, and followed by a cluster health check.
3. Masters are upgraded one at a time. For each original master, Ansible finds its connected upgraded replica and sends `CLUSTER FAILOVER` to that replica. Once the replica is promoted, the old master is stopped, upgraded, restarted, and verified as a synced replica.
4. Post-upgrade checks verify data integrity and require every node to report the target Redis version before printing `UPGRADE COMPLETE`.

If a step fails, Ansible stops at that task and the CLI exits non-zero. The playbook does not attempt automatic rollback, matching the base assignment requirement.

## Data Integrity

The seed and verify playbooks use deterministic key/value pairs:

- Key format: `key:0001`, `key:0002`, ...
- Value: SHA256 digest of the key string

Because the value can be recomputed independently, `data verify`, `upgrade`, and `verify --full` can prove whether keys are missing or mismatched before and after the upgrade.

## Assumptions And Trade-Offs

- Redis is built from official source tarballs to guarantee exact versions such as `7.0.15` and `7.2.6`.
- Redis is run as a daemon inside the SSH containers instead of using systemd, because the assignment containers are SSH-managed Ubuntu nodes and most compose containers do not run systemd as PID 1.
- Ansible connects through published localhost SSH ports for portability. The Redis cluster bus and client traffic use the fixed `10.10.0.0/24` container network and `cluster-announce-ip` values.
- The role uses only Ansible built-in modules and project-owned tasks/templates. No Ansible Galaxy roles or Redis collections are used.

## Known Limitations

- Initial Redis source downloads require internet access from the containers.
- Rootless Podman networking with static IPs depends on the installed Podman and podman-compose versions. Docker Compose is a fallback if Podman networking is unavailable.
- Rollback, scale-out, and scale-in are not implemented because they are stretch goals.
