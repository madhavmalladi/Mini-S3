# Mini-S3

A distributed object storage system written in Go, modeled after Amazon S3 and NVIDIA AIStore.

## What it is

Two services:

- **Gateway** — public REST API. Runs a consistent hash ring, decides which storage node owns each object, handles replication, and health-checks nodes.
- **Storage node** — dumb binary that reads and writes files to disk. Identical instances run as a fleet behind the gateway.

Deployment target: 1 gateway + 3 storage nodes, orchestrated with Docker Compose, then migrated to Kubernetes via minikube.

## REST API

```
PUT    /objects/{key}            upload an object
GET    /objects/{key}            download an object
DELETE /objects/{key}            delete an object
GET    /objects                  list all keys
GET    /objects/{key}/integrity  verify SHA-256 checksum
GET    /cluster/ring             inspect the hash ring
GET    /cluster/health           per-node health
GET    /cluster/status           cluster-wide status
```

## Architecture

```
                  ┌─────────────┐
   client ──────▶ │   gateway   │ ──── consistent hash ring (150 vnodes/node, CRC32)
                  └─────────────┘
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
       ┌────────┐  ┌────────┐  ┌────────┐
       │ node 1 │  │ node 2 │  │ node 3 │
       └────────┘  └────────┘  └────────┘
                  flat-file storage on disk
```

**Replication factor 2**: every write goes to a primary node (chosen by the ring) and the next node clockwise. Reads fall back to the replica if the primary is down. A background health checker on a 10s ticker evicts nodes after three consecutive failed checks. An in-memory repair queue drains when a node recovers.

## Key design decisions

- **Consistent hashing over round-robin** — adding or removing a node remaps only `1/n` keys instead of all of them.
- **150 virtual nodes per real node** — without virtual nodes, key distribution is uneven and hotspots form. 150 vnodes statistically smooths the distribution.
- **Replication factor 2, not 3** — one primary + one replica tolerates single-node failure. Factor 3 (HDFS-style) is overkill at this scale.
- **Gateway/node split** — the gateway is the brain (routing, replication, health). Nodes are dumb storage. Mirrors how real object stores are architected and makes the system horizontally scalable: add nodes without touching gateway code.
- **Three-strike eviction** — a single missed health check could be a transient network blip. Three strikes avoid false-positive evictions.

## Repository layout

```
mini-s3/
  node/
    main.go         entry point
    handler.go      HTTP handlers
    store.go        disk read/write logic
    Dockerfile
  gateway/
    main.go         entry point
    handler.go      HTTP handlers, proxies to nodes
    ring.go         consistent hash ring
    client.go       HTTP client for node communication
    health.go       background health checker
    replication.go  replication writes + repair queue
    Dockerfile
  docker-compose.yml
  k8s/
    node-deployment.yaml
    node-service.yaml
    gateway-deployment.yaml
    gateway-service.yaml
    configmap.yaml
    pvc.yaml
```

## Running

Single node:

```bash
go run ./node
# server listens on :8080 by default; storage dir defaults to ./data
```

Configurable via env vars: `PORT`, `STORAGE_DIR`.

Full cluster (once Phase 4 is in):

```bash
docker compose up
```
