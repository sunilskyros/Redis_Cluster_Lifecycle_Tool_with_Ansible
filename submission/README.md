# Redis Cluster Lifecycle Tool

This project provides a robust, production-grade CLI tool and Ansible orchestration suite to manage the lifecycle of a Redis Cluster. It automates container-based infrastructure setup, cluster provisioning, deterministic data seeding/verification, zero-downtime rolling upgrades, clean rollbacks, and horizontal scaling (scale-out/scale-in).

The orchestration utilizes a wrapper script `redis-tool` to manage local SSH containers and execute Ansible playbooks, ensuring configuration reliability and zero downtime.

---

## Project Layout

```text
submission/
├── redis-tool                # Python 3 CLI wrapper
├── ansible/
│   ├── ansible.cfg           # Ansible execution settings
│   ├── group_vars/all.yml    # Global configuration variables
│   ├── inventory/hosts.ini   # Dynamically updated Ansible hosts mapping
│   ├── playbooks/
│   │   ├── provision.yml     # Compiles & installs Redis, forms initial cluster
│   │   ├── data_seed.yml     # Seeds deterministic keys using SHA256 values
│   │   ├── data_verify.yml   # Validates deterministic key presence & value hashes
│   │   ├── status.yml        # Generates detailed topology & node statistics
│   │   ├── upgrade.yml       # Orchestrates rolling zero-downtime upgrades
│   │   ├── verify_full.yml   # Executes extensive health audits on topology/lag
│   │   ├── rollback.yml      # Cleans database and downgrades nodes
│   │   └── scale_provision.yml # Installs Redis onto newly added scale nodes
│   └── roles/redis/          # Ansible Redis deployment role
├── infra/
│   ├── Containerfile         # Image specification for SSH-enabled Ubuntu nodes
│   ├── Dockerfile            # Duplicate of Containerfile for Docker compatibility
│   └── compose.yml           # Dynamically modified container orchestration file
├── logs/                     # Folder containing operation audit logs (*.jsonl)
├── output/                   # Folder containing detailed command terminal captures
└── README.md
```

---

## Prerequisites

`redis-tool` performs validation checks on dependencies before executing commands. Ensure the following are installed:

* **Container Runtime**: Podman or Docker. (If both are present, Podman is preferred).
* **Compose Engine**: `podman-compose`, `podman compose`, `docker compose`, or `docker-compose`.
* **Ansible**: `ansible-playbook` (version 2.14.0+ is required).
* **SSH Key Utility**: `ssh-keygen` (for creating local container keys).

### Installation Reference
* Podman: [podman.io/docs/installation](https://podman.io/docs/installation)
* Docker: [docs.docker.com/engine/install](https://docs.docker.com/engine/install/)
* Ansible: `pip install ansible` or install via system package manager.

---

## Quick Start: Bring Up Infrastructure

All commands should be executed from the `submission` directory:

```bash
# Make the CLI script executable
chmod +x ./redis-tool

# Start the environment containers (you can run without this also )
./redis-tool infra up
```

`infra up` performs the following steps:
1. Generates a secure ED25519 key pair under `infra/ssh/`.
2. Creates an authorized SSH keys list and builds the `redis-node-base` container image based on Ubuntu 22.04.
3. Starts six target containers on a bridge network (`redis-cluster`) with static IPs (`10.10.0.11` through `10.10.0.16`).
4. Maps container SSH ports (`22`) to local localhost ports (`2221` through `2226`) for Ansible access.

To tear down the containers and clean up the infrastructure:

```bash
./redis-tool infra down
```

---

## Command Reference

The `redis-tool` CLI provides the following subcommands:

### 1. Provision Cluster
Compile and deploy a specified Redis version and form the initial master-replica topology:
```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

### 2. Status Output
Generate a formatted status view detailing nodes role, version, hash slot assignments, database key counts, memory usage, and replication link status:
```bash
./redis-tool status
```

### 3. Data Seeding
Populate the database with deterministic test keys. Each key's value is set to the SHA256 checksum of its key string, enabling offline integrity checks:
```bash
./redis-tool data seed --keys 1000
```

### 4. Data Verification
Check keys in the database and re-calculate SHA256 checksums to verify that keys are present and data hasn't corrupted:
```bash
./redis-tool data verify
```

### 5. Rolling Upgrade
Perform a zero-downtime, no-data-loss rolling upgrade to a target Redis version:
```bash
./redis-tool upgrade --target-version 7.2.6 --strategy rolling
```

### 6. Full Health Verification
Run a comprehensive suite of checks including version consistency, slot coverage (verifying all 16,384 slots are covered), replication lag, and replica health:
```bash
./redis-tool verify --full
```

### 7. Rollback
Revert the cluster to a previous target version cleanly:
```bash
./redis-tool rollback --target-version 7.0.15 
```

### 8. Dynamic Scaling (Scale-Out & Scale-In)
Scale the cluster size horizontally.

* **Scale-Out**: Add a new master-replica pair (2 nodes) to the cluster and rebalance slots:
  ```bash
  ./redis-tool scale --add-nodes 2
  ```

* **Scale-In**: Remove a specific node (and its corresponding master or replica) from the cluster:
  ```bash
  # Can specify node using its name, container index, prefix, or exact node ID
  ./redis-tool scale --remove-node redis-node-7
  ```

---

## Core Operational Logic

### Rolling Upgrade Strategy
To maintain availability, the upgrade process follows a strict sequence:
1. **Pre-flight Checks**: Verifies cluster status (`cluster_state:ok`), checks connectivity, records current versions, and performs a baseline data integrity verify.
2. **Replicas First**: Upgrades replicas sequentially (`serial: 1`). Stops the service, installs/compiles the target Redis version, starts it, and polls until `master_link_status` reports `up`.
3. **Masters Failover & Upgrade**: For each master node (`serial: 1`):
   * Identifies its active, upgraded replica.
   * Promotes the replica to master via `CLUSTER FAILOVER`.
   * Waits until the promotion propagates and the old master demotes.
   * Stops the old master (now replica), installs the target version, restarts it, and confirms it syncs cleanly as a replica.
4. **Post-Checks**: Validates version uniformity across nodes and checks that the seeded keys are intact.

### Rollback Process
Reverting to an older version requires clearing data because Redis files are generally not backward-compatible:
1. Cleanly stops the Redis service on nodes.
2. Deletes DB files (`dump.rdb` and `appendonlydir`) to prevent downgrade compatibility issues.
3. Installs and compiles the downgraded version.
4. Restarts Redis and waits for the cluster state to stabilize.
5. Re-runs data seeding to restore test datasets.

### Scaling Mechanics
* **Scale-Out**:
  1. Detects the running version and appends two new node services to `infra/compose.yml` (allocating free ports and IPs).
  2. Launches containers and updates `ansible/inventory/hosts.ini`.
  3. Deploys Redis via `scale_provision.yml`.
  4. Integrates the new master, assigns the new replica to follow it, and runs `rebalance` to distribute hash slots.
* **Scale-In**:
  1. Resolves the requested node and its counterpart.
  2. Ensures the cluster keeps a minimum of 3 masters.
  3. Migrates slots off the targeted master to another active master node.
  4. Deletes nodes from the cluster structure (`CLUSTER MEET/FORGET` logic via `redis-cli`).
  5. Removes containers and updates the configuration inventory to clean up host entries.

---

## Log Capture & Output Audits

Each lifecycle command records output in two directories:
* **`output/`**: Stores plain-text terminal transcripts for each operation (e.g., `upgrade_output.txt`).
* **`logs/`**: Records chronological JSONL audit logs tracking the start, action, node targets, and exit statuses of each operational step (e.g., `upgrade.jsonl`).

---

## Troubleshooting

For diagnosing and solving common issues (such as Podman socket errors, SSH key conflicts, or compilation problems), refer to the [Troubleshooting Guide](file:///home/skyros/Desktop/Redis_Cluster_Lifecycle_Tool_with_Ansible/submission/TROUBLESHOOTING.md).
