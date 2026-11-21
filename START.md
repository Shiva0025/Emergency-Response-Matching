# START.md — Getting Started with ERMS

This guide takes you from a clean machine to a running **Distributed Emergency Resource Matching System (ERMS)**,
and shows you how to test it. Every command here comes from the project's own files
(`docker-compose.yml`, `Dockerfile`s, `k8s/`, `scripts/`, `tests/`, `requirements.txt`).

> For design details (why things work the way they do), see [`README.md`](README.md).

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Prerequisites](#2-prerequisites)
3. [Installation and Setup](#3-installation-and-setup)
4. [How to Run the Project](#4-how-to-run-the-project)
5. [Testing](#5-testing)
6. [Project Structure](#6-project-structure)
7. [Common Issues and Troubleshooting](#7-common-issues-and-troubleshooting)
8. [Additional Information](#8-additional-information)

---

## 1. Project Overview

### What it does

ERMS connects people who need emergency help (water, food, medicine, transportation, shelter, rescue) with
available resources, volunteers and NGOs. A client creates a **request** ("family of 4 needs drinking water at
this location") and resources register themselves ("water tanker, can serve 2 requests, at this location").
The system matches each request to the **nearest suitable resource** that has spare capacity.

### Main features

- **REST API** for creating/tracking requests and registering/updating resources (Swagger UI included).
- **Asynchronous matching** through RabbitMQ, with a delayed retry queue for requests that can't be matched yet.
- **Synchronous matching** on demand via `POST /match/{requestId}`.
- **Sharded storage**: data is split across 3 shards using `sha256(key) mod 3`.
- **Replication**: every shard has 2 replicas; reads fail over automatically if one goes down.
- **Failure detection and recovery**: storage nodes send heartbeats; a node that comes back resyncs from its peer.
- **Safe concurrency**: compare-and-set prevents a resource from being over-allocated by parallel matchers.
- Runs on **Docker Compose** (local) or **Kubernetes** (minikube / kind / Docker Desktop).

### Architecture

```
client ──REST──▶ request-service (:8080) ──sync POST /match──▶ matching-service (×2)
                     │        │                                   │         │
              publish│        │storage ops                 consume│         │storage ops
                     ▼        ▼                                   │         ▼
                  RabbitMQ ──────────────────────────────────────┘    coordinator (×1, :8001)
          erms.requests → matching.requests                           routes by sha256(key) mod 3
          erms.retry    → matching.retry (TTL 5s, back to main)              │
                                                       ┌─────────────────────┼─────────────────────┐
                                                    shard 0               shard 1               shard 2
                                               storage-0a / 0b       storage-1a / 1b       storage-2a / 2b
                                                  (SQLite each)         (SQLite each)         (SQLite each)
```

| Component | Role |
|---|---|
| **request-service** | Client-facing REST API. Validates input, stores data via the coordinator, publishes new requests to RabbitMQ, forwards `POST /match` to the matching service. Stateless. |
| **matching-service** | Consumes the `matching.requests` queue, assigns the best resource, retries unmatched requests. Also runs a background "sweeper". Stateless (2 replicas by default). |
| **coordinator** | Exactly one. Keeps the registry of storage nodes, detects failures, routes every read/write to the right shard and its replicas. |
| **storage nodes** | 3 shards × 2 replicas = 6 nodes. Versioned key/value store in SQLite. |
| **RabbitMQ** | Message broker for asynchronous hand-off between request intake and matching. |

### Technologies used

| Category | Technology | Version (from project files) |
|---|---|---|
| Language | Python | 3.12 (`python:3.12-slim` base image) |
| Web framework | FastAPI | 0.115.6 |
| ASGI server | Uvicorn (`[standard]`) | 0.32.1 |
| HTTP client | httpx | 0.28.1 |
| RabbitMQ client | aio-pika | 9.5.4 |
| Validation | pydantic | 2.10.4 |
| Storage | SQLite (Python standard library `sqlite3`) | bundled with Python |
| Message broker | RabbitMQ | `rabbitmq:3.13-management` image |
| Containers | Docker, Docker Compose | — |
| Orchestration | Kubernetes (minikube, kind, or Docker Desktop) | — |
| Tests | pytest | not pinned |

---

## 2. Prerequisites

### Required for the standard (Docker Compose) setup

| Tool | Why | Notes |
|---|---|---|
| **Git** | Clone the repository | any recent version |
| **Docker Engine** (or Docker Desktop) | Builds and runs all 12 containers | must be running |
| **Docker Compose v2** | `docker compose ...` commands (note: space, not hyphen) | ships with modern Docker |
| **curl** | Used by `scripts/demo.sh` and the examples below | |
| **Python 3** (`python3`) | Used by the demo scripts to pretty-print JSON, and to run tests | 3.12 recommended (matches the containers) |
| **bash** | Shell scripts in `scripts/` and `k8s/` | Linux/macOS; on Windows use WSL |

**Hardware:** the Kubernetes instructions in this repo use `--cpus=4 --memory=4096` for minikube; plan for similar
resources if you use Kubernetes. Docker Compose needs less — exact minimum: **Requires manual configuration** (not specified in the project).

### Optional — only for Kubernetes

| Tool | Why |
|---|---|
| **kubectl** | Apply manifests, scale, inspect pods |
| **minikube** *or* **kind** *or* Docker Desktop Kubernetes | A local cluster. `k8s/deploy.sh` detects which one you use from the current kubectl context. |
| **metrics-server** (minikube addon) | Only for the optional autoscaler `k8s/60-hpa-optional.yaml` |

> **No database server is needed.** Each storage node uses an embedded SQLite file inside its container.
> **No Node.js / frontend toolchain is needed.** This project has no frontend; the UI is FastAPI's built-in Swagger page.

---

## 3. Installation and Setup

### Step 1 — Clone the repository

```bash
git clone https://github.com/Shiva0025/Emergency-Response-Matching.git
cd Emergency-Response-Matching
```

> All commands in this guide must be run **from the repository root** (the folder containing `docker-compose.yml`).
> The Dockerfiles build with the repository root as their context, and the scripts `cd` there themselves.

### Step 2 — Install dependencies

**For running the system: nothing to install manually.** Docker installs the Python dependencies from
`requirements.txt` inside each image during `docker compose up --build`.

**For running tests locally** (or running services outside Docker), create a virtual environment:

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt pytest
```

> **Python 3.13 or newer?** The pinned versions in `requirements.txt` target Python 3.12 and may fail to install
> on newer Python. The README recommends installing unpinned packages instead:
> ```bash
> .venv/bin/pip install fastapi uvicorn httpx aio-pika pydantic pytest
> ```
>
> The README's install line also adds `pyyaml`. No file in `tests/` imports it, so it's optional.

### Step 3 — Environment variables / `.env` files

**This project does not use `.env` files**, and there's no `.env.example`. Every setting is passed as an environment
variable that is already set in `docker-compose.yml` (for Compose) and in the `k8s/*.yaml` manifests (for Kubernetes).
**You don't need to configure anything to get started.**

To change a value, edit the `environment:` block of the service in `docker-compose.yml` (or the `env:` list in the
matching `k8s/` manifest) and restart. The full list of variables read by the code:

#### request-service
| Variable | Default in code | Set in Compose / k8s | Purpose |
|---|---|---|---|
| `COORDINATOR_URL` | `http://coordinator:8000` | `http://coordinator:8000` | Where the storage coordinator lives |
| `RABBITMQ_URL` | `amqp://guest:guest@rabbitmq:5672/` | same | RabbitMQ connection URL |
| `MATCHING_SERVICE_URL` | `http://matching-service:8000` | same | Target for `POST /match/{id}` |

#### matching-service
| Variable | Default in code | Set in Compose / k8s | Purpose |
|---|---|---|---|
| `COORDINATOR_URL` | `http://coordinator:8000` | same | Storage coordinator |
| `RABBITMQ_URL` | `amqp://guest:guest@rabbitmq:5672/` | same | RabbitMQ |
| `PREFETCH` | `2` | `2` | Unacknowledged messages per worker |
| `RETRY_DELAY_MS` | `5000` | `5000` | How long an unmatched request waits in `matching.retry` |
| `MAX_MATCH_ATTEMPTS` | `12` | `12` | Attempts before a request becomes `unmatched` |
| `MAX_MATCH_DISTANCE_KM` | `50` | `50` | Maximum matching distance (haversine) |
| `MATCH_DELAY_MS` | `0` | **`1500`** | Artificial work time per match so queueing is visible in demos. Set to `0` for real use. |
| `STALE_PENDING_SECONDS` | `60` | **`30`** | Sweeper re-queues auto-match requests idle longer than this |
| `SWEEP_INTERVAL` | `30` | **`10`** | Seconds between sweeper runs |

#### coordinator
| Variable | Default in code | Set in Compose / k8s | Purpose |
|---|---|---|---|
| `HEARTBEAT_TIMEOUT` | `6` | `6` | Seconds without a heartbeat before a node is marked `down` |
| `NODE_REQUEST_TIMEOUT` | `2` | not set | HTTP timeout for coordinator → storage node calls |

#### storage node
| Variable | Default in code | Set in Compose | Purpose |
|---|---|---|---|
| `NODE_ID` | container hostname | `storage-0a` … `storage-2b` | Unique node name |
| `SHARD_ID` | `0` | `0`, `1` or `2` | Which shard this node holds |
| `NODE_URL` | `http://<POD_IP or hostname>:<PORT>` | `http://storage-Xy:8000` | URL the coordinator uses to reach the node |
| `COORDINATOR_URL` | `http://coordinator:8000` | same | Where to register / heartbeat |
| `DATA_DIR` | `/data` | `/data` (a named volume) | Folder for the SQLite file |
| `HEARTBEAT_INTERVAL` | `2` | not set | Seconds between heartbeats |
| `PORT` | `8000` | not set | Only used to build the default `NODE_URL` |
| `POD_IP` | — | set in k8s via downward API | Used to build `NODE_URL` on Kubernetes |

#### shared (all services)
| Variable | Default | Purpose |
|---|---|---|
| `NUM_SHARDS` | `3` | Number of shards. If you change it, you must also add/remove storage nodes yourself in `docker-compose.yml` / `k8s/30-storage.yaml`: **Requires manual configuration**. |
| `LOG_LEVEL` | `INFO` | Python logging level |

#### RabbitMQ credentials
RabbitMQ runs with the image defaults **`guest` / `guest`** (no credentials are set in `docker-compose.yml`). These are
fine for local development only. For any non-local setup, configuring real credentials is
**Requires manual configuration**: the project doesn't provide it.

### Step 4 — Database setup, migrations and seed data

- **Migrations:** none needed. Each storage node creates its SQLite table automatically on startup
  (`CREATE TABLE IF NOT EXISTS kv ...` in `storage_node/app.py`).
- **Database location:** Docker named volumes `storage-0a` … `storage-2b`, mounted at `/data` in each storage container.
  On Kubernetes, storage pods use `emptyDir` volumes, so data lasts through a container restart but not a pod deletion.
  A recreated pod copies its data back from its peer replica.
- **Seed data:** there's no seed script. To fill the system with sample data, run the guided demo
  (`./scripts/demo.sh`, see [section 4](#4-how-to-run-the-project)). It creates requests and resources around Bengaluru.

---

## 4. How to Run the Project

### Option A — Docker Compose (recommended for development)

#### 1. Build and start everything

```bash
docker compose up -d --build
```

This builds 4 images (`erms/coordinator`, `erms/storage-node`, `erms/request-service`, `erms/matching-service`),
pulls `rabbitmq:3.13-management`, and starts **12 containers**:

| Container(s) | Count | Host port → container port |
|---|---|---|
| `rabbitmq` | 1 | `5672→5672` (AMQP), `15672→15672` (management UI) |
| `coordinator` | 1 | `8001→8000` |
| `storage-0a`, `storage-0b`, `storage-1a`, `storage-1b`, `storage-2a`, `storage-2b` | 6 | internal only |
| `request-service` | 1 | `8080→8000` |
| `matching-service` | 2 (replicas) | internal only |

Start order is handled for you: `request-service` and `matching-service` wait until RabbitMQ's healthcheck passes,
and storage nodes start after the coordinator.

#### 2. Check that it's running

```bash
docker compose ps
```
All 12 containers should show `Up`, and `rabbitmq` should show `(healthy)`.

```bash
curl -s localhost:8080/health
# {"status":"ok","instance":"<container-id>","broker_connected":true}

curl -s localhost:8001/health
# {"status":"ok","healthy_shards":[0,1,2],"nodes":6}
```

If the coordinator reports `"status":"degraded"`, wait a few seconds: storage nodes are still registering.

#### 3. Open the local URLs

| URL | What it is |
|---|---|
| http://localhost:8080/docs | **Swagger UI**: try every client API endpoint in the browser |
| http://localhost:8080 | Client REST API (request-service) |
| http://localhost:8080/cluster | Shortcut to the coordinator's cluster view |
| http://localhost:8001/cluster | Coordinator: shards, nodes, status, key counts, read statistics |
| http://localhost:15672 | RabbitMQ management UI. Login **guest / guest** |

#### 4. Expected startup output

Watch logs with:
```bash
docker compose logs -f                        # everything
docker compose logs -f coordinator storage-0a # specific services
```

Healthy startup looks like this (timestamps and container IDs will differ):

```text
coordinator-1       | ... [coordinator] INFO coordinator started: 3 shards, heartbeat timeout 6s
storage-0a-1        | ... [storage-storage-0a] INFO registered with coordinator (shard 0, url http://storage-0a:8000); peers=[]
storage-0a-1        | ... [storage-storage-0a] INFO no healthy peers in shard 0; serving local data (0 keys)
request-service-1   | ... WARNING RabbitMQ unavailable (<AMQPConnectionError: (111, 'Connection refused')>); retrying in 3s
request-service-1   | ... INFO publishing to exchange 'erms.requests'
request-service-1   | ... INFO connected to RabbitMQ; topology declared
matching-service-1  | ... INFO consuming from 'matching.requests' (prefetch=2)
matching-service-1  | ... INFO connected to RabbitMQ; topology declared
```

> A few `RabbitMQ unavailable ... retrying in 3s` warnings at startup are **normal**: the services retry until the broker
> is fully ready. They're only a problem if they never stop (see [Troubleshooting](#7-common-issues-and-troubleshooting)).

#### 5. Try it out

Create a request (with `auto_match` left at its default of `true`, it's queued for matching right away):
```bash
curl -X POST localhost:8080/requests -H 'Content-Type: application/json' -d '{
  "type": "water", "details": "Family of 4 stranded by flooding",
  "location": {"lat": 12.9716, "lon": 77.5946, "area": "Shivajinagar"}, "priority": "high"}'
```

Register a resource nearby:
```bash
curl -X POST localhost:8080/resources -H 'Content-Type: application/json' -d '{
  "type": "water", "kind": "volunteer", "name": "Ravi (tanker)", "availability": 2,
  "location": {"lat": 12.978, "lon": 77.599}}'
```

Within a few seconds the retry queue re-delivers the request and it gets matched. Check it with:
```bash
curl -s localhost:8080/requests/<id-from-first-response>
```

#### 6. Run the guided demo (optional)

The demo walks through creation, matching, shard distribution, async processing, a storage node failure and recovery:

```bash
./scripts/demo.sh          # pauses between steps, press Enter to continue
AUTO=1 ./scripts/demo.sh   # runs without pauses
```

> ⚠️ Step 6 of the demo **stops a storage container** and step 7 starts it again. That's intended.

#### 7. Stop the project

```bash
docker compose stop        # stop containers, keep them and their data
docker compose down        # remove containers, keep data volumes
docker compose down -v     # remove containers AND all stored data (clean reset)
```

#### Running individual components

All services are started together by `docker compose up`. You can also control them one by one:

```bash
docker compose up -d rabbitmq                        # start only the broker
docker compose restart request-service               # restart one service
docker compose stop storage-1a                       # simulate a storage node failure
docker compose start storage-1a                      # bring it back (it resyncs from storage-1b)
docker compose up -d --build request-service         # rebuild + restart after changing its code
```

> **Frontend vs. backend:** there's no separate frontend. The only UI is FastAPI's Swagger page at `/docs`.

---

### Option B — Running services directly with Python (without Docker for the app code)

The project has no script for this, but each Dockerfile's `CMD` is a plain `uvicorn` command, so you can run the same
commands from the virtual environment (Step 2 above). This is handy for debugging one service. The service hostnames in
the code defaults (`coordinator`, `rabbitmq`, `matching-service`) only resolve inside Docker, so **you must set the URLs
yourself**. The ports below are examples: any free ports work.

```bash
# 1) RabbitMQ: easiest via Compose
docker compose up -d rabbitmq

# 2) Coordinator
.venv/bin/uvicorn coordinator.app:app --port 9001

# 3) Storage nodes: one terminal each; use a separate DATA_DIR per node. Repeat for shards 1 and 2.
NODE_ID=local-0a SHARD_ID=0 NODE_URL=http://127.0.0.1:9010 COORDINATOR_URL=http://127.0.0.1:9001 \
DATA_DIR=./local-data/0a .venv/bin/uvicorn storage_node.app:app --port 9010

# 4) Matching service
COORDINATOR_URL=http://127.0.0.1:9001 RABBITMQ_URL=amqp://guest:guest@localhost:5672/ \
.venv/bin/uvicorn matching_service.app:app --port 9020

# 5) Request service
COORDINATOR_URL=http://127.0.0.1:9001 RABBITMQ_URL=amqp://guest:guest@localhost:5672/ \
MATCHING_SERVICE_URL=http://127.0.0.1:9020 .venv/bin/uvicorn request_service.app:app --port 9030
```

Steps 2–3 were checked: the coordinator logs `NODE REGISTERED: local-0a shard=0 ... -> READY`.
Until all 3 shards have a node, `/health` on the coordinator reports `"degraded"` and requests for missing shards
return `503`. Run commands from the repository root so Python can import the `common` package.
`local-data/` isn't in `.gitignore`, but `*.db` files are; delete the folder when you're done.

---

### Option C — Kubernetes

Works with **minikube**, **kind** or **Docker Desktop Kubernetes**.

#### 1. Start a cluster (minikube example from the README)

```bash
minikube start --driver=docker --cpus=4 --memory=4096
minikube addons enable metrics-server     # only needed for the optional HPA
```

#### 2. Build, load and deploy

```bash
./k8s/deploy.sh
```

What the script does (`k8s/deploy.sh`):
1. Builds the 4 `erms/*:latest` images with Docker.
2. Loads them into the cluster: `minikube image load` (context `minikube*`), `kind load docker-image` (context `kind-*`),
   or nothing for Docker Desktop (it shares the local image store).
3. Applies `k8s/00-namespace.yaml` … `k8s/50-request-service.yaml` into namespace **`erms`**.
4. Restarts the deployments so they use the newly built images, then waits for every rollout.
5. Prints `kubectl -n erms get pods -o wide` and the URLs.

Expected final output:
```text
Deployed. NodePorts:
  API (Swagger):  http://<minikube-ip>:30080/docs
  Coordinator:    http://<minikube-ip>:30081/cluster
  RabbitMQ UI:    http://<minikube-ip>:30672  (guest/guest)
Demo: ./scripts/k8s-demo.sh
```

The optional autoscaler is **not** applied by the script. To use it:
```bash
kubectl apply -f k8s/60-hpa-optional.yaml
```

#### 3. Access the services

| Service | minikube (NodePort) | kind / Docker Desktop (port-forward) |
|---|---|---|
| Client API | `http://$(minikube ip):30080/docs` | `kubectl -n erms port-forward svc/request-service 8080:8000` → http://localhost:8080/docs |
| Coordinator | `http://$(minikube ip):30081/cluster` | `kubectl -n erms port-forward svc/coordinator 8001:8000` |
| RabbitMQ UI | `http://$(minikube ip):30672` | `kubectl -n erms port-forward svc/rabbitmq 15672:15672` |

> NodePorts keep working while pods are killed and replaced; `kubectl port-forward` doesn't, because it binds to a single pod.

#### 4. Kubernetes demo and useful commands

```bash
./scripts/k8s-demo.sh      # pod recovery, scaling, storage failure on k8s (AUTO=1 for no pauses)
```
On kind / Docker Desktop, point the demo at your port-forwards with `API=`, `COORD=` and `RABBIT=` (the script supports
these overrides), e.g. `API=http://localhost:8080 COORD=http://localhost:8001 RABBIT=http://localhost:15672 ./scripts/k8s-demo.sh`.
Port-forwards to a pod that the demo deletes will break, so minikube is the smoothest option for this demo.

```bash
kubectl -n erms get pods
kubectl -n erms delete pod -l app=request-service --wait=false   # automatic recovery
kubectl -n erms scale deploy/matching-service --replicas=4       # scaling
kubectl -n erms scale sts/storage-shard-1 --replicas=1           # storage node failure (removes storage-shard-1-1)
kubectl -n erms scale sts/storage-shard-1 --replicas=2           # recovery: new pod resyncs from its peer
```

To remove everything: `kubectl delete namespace erms` (this is a standard kubectl command; the project has no teardown script).

---

### Development vs. production mode

The project has **one configuration**, tuned for local development and demos. There's no separate production profile.

| Setting | Current (demo) value | Suggested change for real use |
|---|---|---|
| `MATCH_DELAY_MS` | `1500` (artificial delay) | `0`, as the README recommends |
| RabbitMQ credentials | `guest` / `guest` | **Requires manual configuration** |
| Uvicorn auto-reload | not enabled | for live code reload in development, add `--reload` when using Option B |
| TLS, authentication, secrets, persistent volumes on k8s | not provided | **Requires manual configuration** |

---

## 5. Testing

### Unit tests (no Docker needed)

Fast tests for the pure logic: shard hashing, haversine distance, candidate ranking, and the last-writer-wins store.

```bash
.venv/bin/python -m pytest tests/unit
```
Expected: `7 passed` in under a second. (`tests/conftest.py` sets a temporary `DATA_DIR` automatically, because
`storage_node.app` opens a SQLite file on import.)

### Integration tests (against a running deployment)

End-to-end tests covering the API, async matching, concurrency (20 requests competing for 12 units), shard
distribution, **storage node failure/recovery** and **RabbitMQ outage**. They take about **70 seconds**.

```bash
docker compose up -d --build
.venv/bin/python -m pytest -v tests/integration
```

> ⚠️ These tests **stop and restart** one storage container and the `rabbitmq` container (with
> `docker compose stop/start`). Run them from the repository root so those commands find `docker-compose.yml`.
> Each run uses a unique random location, so data left over from earlier runs doesn't affect results.

Configurable through environment variables (defaults in `tests/integration/test_system.py`):

| Variable | Default |
|---|---|
| `API_URL` | `http://localhost:8080` |
| `COORD_URL` | `http://localhost:8001` |
| `STOP_CMD` / `START_CMD` | `docker compose stop {node}` / `docker compose start {node}` |
| `FAIL_REPLICA` | `first` (`first` or `last`) |
| `BROKER_STOP_CMD` / `BROKER_START_CMD` | `docker compose stop rabbitmq` / `docker compose start rabbitmq` |

**Against Kubernetes (minikube):**
```bash
IP=$(minikube ip); API_URL=http://$IP:30080 COORD_URL=http://$IP:30081 \
FAIL_REPLICA=last STOP_CMD="kubectl -n erms scale sts {statefulset} --replicas=1" \
START_CMD="kubectl -n erms scale sts {statefulset} --replicas=2" \
BROKER_STOP_CMD="kubectl -n erms scale deploy rabbitmq --replicas=0" \
BROKER_START_CMD="kubectl -n erms scale deploy rabbitmq --replicas=1" \
.venv/bin/python -m pytest -v tests/integration
```

Run a single test:
```bash
.venv/bin/python -m pytest -v tests/integration/test_system.py::test_async_matching_through_rabbitmq
```

### Linting and formatting

**Not configured.** The repository has no linter or formatter config (no `pyproject.toml`, `setup.cfg`, `.flake8`,
`ruff.toml` or pre-commit config). Choosing and setting up a linter is **Requires manual configuration**.

### Build

There's no separate build step for the Python code. "Building" means building the Docker images:

```bash
docker compose build                       # all images via Compose
# or one image by hand (command from the header of each Dockerfile):
docker build -f coordinator/Dockerfile      -t erms/coordinator:latest .
docker build -f storage_node/Dockerfile     -t erms/storage-node:latest .
docker build -f request_service/Dockerfile  -t erms/request-service:latest .
docker build -f matching_service/Dockerfile -t erms/matching-service:latest .
```

Validate the Compose file without starting anything:
```bash
docker compose config -q && echo OK
```

### CI

No CI pipeline is configured (no `.github/workflows/` or similar).

---

## 6. Project Structure

```
.
├── common/                     Shared library used by all services
│   ├── hashing.py              stable sha256-based shard selection (NUM_SHARDS)
│   ├── storage_client.py       HTTP client for the coordinator + error types (NotFound/Conflict/Unavailable)
│   ├── messaging.py            RabbitMQ topology (exchanges, queues, retry TTL) + reconnect loop
│   ├── resources.py            compare-and-set capacity assign/release for resources
│   ├── models.py               request types, statuses, key helpers
│   ├── geo.py                  haversine distance
│   └── logging_setup.py        log format, LOG_LEVEL
├── coordinator/
│   ├── app.py                  node registry, heartbeats/failure detection, shard routing, replication, /cluster
│   └── Dockerfile
├── storage_node/
│   ├── app.py                  SQLite key/value store, register + heartbeat, resync from peer
│   └── Dockerfile
├── request_service/
│   ├── app.py                  client REST API (requests, resources, match proxy, health)
│   └── Dockerfile
├── matching_service/
│   ├── app.py                  RabbitMQ consumer, retry logic, stale-request sweeper, /match, /stats
│   ├── matcher.py              candidate filtering/ranking and the matching transaction
│   └── Dockerfile
├── k8s/
│   ├── 00-namespace.yaml       namespace `erms`
│   ├── 10-rabbitmq.yaml        RabbitMQ Deployment + NodePort Service (30672)
│   ├── 20-coordinator.yaml     coordinator Deployment (1 replica) + NodePort Service (30081)
│   ├── 30-storage.yaml         3 StatefulSets × 2 pods + headless Services
│   ├── 40-matching-service.yaml  Deployment (2 replicas) + Service
│   ├── 50-request-service.yaml   Deployment (2 replicas) + NodePort Service (30080)
│   ├── 60-hpa-optional.yaml    optional HPA for matching-service (needs metrics-server)
│   └── deploy.sh               build images, load into cluster, apply manifests, wait for rollout
├── scripts/
│   ├── demo.sh                 guided Docker Compose demo (AUTO=1 = no pauses)
│   ├── k8s-demo.sh             guided Kubernetes demo
│   ├── lib.sh                  helpers shared by the demos (API/COORD/RABBIT URLs, curl wrappers)
│   └── show.py                 JSON pretty-printers for the demos
├── tests/
│   ├── conftest.py             adds repo root to sys.path, sets a temp DATA_DIR
│   ├── unit/test_logic.py      pure-logic tests
│   └── integration/test_system.py  end-to-end tests against a live deployment
├── docker-compose.yml          local deployment of all 12 containers
├── requirements.txt            pinned Python dependencies (used by every image)
├── .dockerignore               keeps .venv, tests, scripts, k8s, PDFs out of images
├── .gitignore                  ignores .venv, caches, *.db, *.pdf
└── README.md                   architecture and design decisions
```

> The assignment spec PDF (`Emergency Response Matching.pdf`) may be in a local copy of the repo, but `*.pdf` is
> git-ignored, so a fresh clone won't include it.

---

## 7. Common Issues and Troubleshooting

### Port already in use

**Symptom:** `docker compose up` fails with `Bind for 0.0.0.0:8080 failed: port is already allocated`
(or 8001 / 5672 / 15672).

**Fix:** find what is using the port and stop it, or change the **host** side of the mapping in `docker-compose.yml`:
```bash
sudo lsof -i :8080          # or: ss -ltnp | grep 8080
```
```yaml
# docker-compose.yml, request-service
ports: ["18080:8000"]       # host 18080 -> container 8000
```
If you change ports, also set `API_URL` / `COORD_URL` for the integration tests and `API` / `COORD` / `RABBIT` for
`scripts/demo.sh` (it reads these from the environment).

### `RabbitMQ unavailable ... retrying in 3s` never stops

A few of these at startup are normal. If they continue:
```bash
docker compose ps rabbitmq          # should be Up (healthy)
docker compose logs rabbitmq
docker compose restart rabbitmq
```
The services reconnect by themselves and re-declare queues. `GET /health` on the request service shows
`"broker_connected": true` once connected. Requests created while the broker is down are still stored
(`"queued": false`), and the matching service's sweeper re-queues them later.

### Coordinator says `"status":"degraded"` / API returns `503`

A shard has no `ready` replica. Check:
```bash
curl -s localhost:8001/cluster | python3 -m json.tool
docker compose ps
docker compose logs storage-1a      # replace with the affected node
```
Start any stopped node with `docker compose start <node>`. It shows `syncing` briefly, then `ready`.

### Request stays `pending`

- No resource of the same `type`, with `status: available`, spare `availability`, within **50 km**. Register one.
  The request is retried every 5 s, up to 12 attempts, then becomes `unmatched`.
- The request was created with `"auto_match": false`. Call `POST /match/{id}`.
- `"queued": false` in the create response means the broker was down. The sweeper re-queues it after
  `STALE_PENDING_SECONDS` (30 s in Compose).

### `POST /match/{id}` returns `409`

No suitable resource is available, the request is already `resolved`/`cancelled`, or the update ran into
contention. Retry in the contention case. An already-matched request returns `200` with `"result": "skipped"`.

### `pip install -r requirements.txt` fails

Usually you're on Python 3.13+ and the pinned versions have no wheels for it. Either use Python 3.12, or install
unpinned packages:
```bash
.venv/bin/pip install fastapi uvicorn httpx aio-pika pydantic pytest
```
This doesn't affect Docker, because the images always use Python 3.12.

### `ModuleNotFoundError: No module named 'common'`

You ran `uvicorn` or `pytest` from a subdirectory. Run from the repository root. `tests/conftest.py` fixes the path for
pytest, but `uvicorn` needs the root as the current directory.

### Missing environment variables

The services have no required variables: all have defaults (see [Step 3](#step-3--environment-variables--env-files)).
But the defaults use Docker hostnames (`coordinator`, `rabbitmq`, `matching-service`), so **outside Docker** you'll see
`storage coordinator unreachable` or `RabbitMQ unavailable` until you set `COORDINATOR_URL`, `RABBITMQ_URL` and
`MATCHING_SERVICE_URL` to reachable addresses.

### Stale data / want a clean slate

```bash
docker compose down -v && docker compose up -d --build
```

### Code changes don't show up

Images are built from your source, so rebuild:
```bash
docker compose up -d --build
```
On Kubernetes, re-run `./k8s/deploy.sh`. It rebuilds, reloads and restarts the deployments.

### Kubernetes: pods stuck in `ErrImagePull` / `ImagePullBackOff`

The manifests use `erms/*:latest` with `imagePullPolicy: IfNotPresent`. These images exist only locally, so they must be
loaded into the cluster. Run `./k8s/deploy.sh` (it does this for minikube and kind). On other clusters,
pushing images to a registry is **Requires manual configuration**.

### Kubernetes: `minikube ip` fails / URLs show `<node-ip>`

You aren't using minikube. Use the `kubectl port-forward` commands from [Option C, step 3](#3-access-the-services).

### Kubernetes: HPA shows `<unknown>` targets

metrics-server isn't installed. On minikube: `minikube addons enable metrics-server`.

### Docker permission denied (Linux)

`permission denied while trying to connect to the Docker daemon socket`: add your user to the `docker` group
(`sudo usermod -aG docker $USER`, then log out and back in) or use `sudo`.

---

## 8. Additional Information

### Client REST API (request-service, port 8080)

Interactive docs: **http://localhost:8080/docs**. OpenAPI JSON: `http://localhost:8080/openapi.json` (FastAPI default).

| Method | Endpoint | Purpose | Success |
|---|---|---|---|
| POST | `/requests` | Create request | 201 |
| GET | `/requests/{id}` | Get request (includes `_storage.shard` / `served_by`) | 200 |
| GET | `/requests?status=&type=` | List requests | 200 |
| PUT | `/requests/{id}/status` | Update status: `{"status": "...", "note": "..."}` | 200 |
| POST | `/resources` | Register resource / volunteer | 201 |
| GET | `/resources?type=&status=` | List resources | 200 |
| GET | `/resources/{id}` | Get resource | 200 |
| PUT | `/resources/{id}/status` | Update `status` and/or `availability` | 200 |
| POST | `/match/{requestId}` | Match now (idempotent) | 200 |
| GET | `/cluster` | Coordinator's cluster view (passthrough) | 200 |
| GET | `/health` | Liveness + `broker_connected` | 200 |

**Request body fields** (`POST /requests`):

| Field | Type | Required | Allowed values / default |
|---|---|---|---|
| `type` | string | yes | `water`, `food`, `medicine`, `transportation`, `shelter`, `rescue`, `other` |
| `location` | object | yes | `{"lat": -90..90, "lon": -180..180, "area": "optional"}` |
| `details` | string | no | default `""` |
| `priority` | string | no | `low`, `medium` (default), `high`, `critical` |
| `requester` | string | no | |
| `auto_match` | bool | no | default `true` (queue for async matching) |

**Resource body fields** (`POST /resources`): `type` (same values as above), `location`, `availability` (int ≥ 0,
default 1), `name`, `kind` (`resource` default, `volunteer`, `ngo`, `other`), `contact`.

**Statuses:**
- Request: `pending → matched → in_progress → resolved`. Any non-terminal status can go to `cancelled`.
  `pending → unmatched` after `MAX_MATCH_ATTEMPTS`. Setting `pending` again re-opens and re-queues a request.
- Resource: `available`, `busy` (availability reached 0), `offline` (never matched).

**Error codes:** `422` invalid input · `404` unknown id · `409` invalid state change / no suitable resource / contention ·
`400` invalid resource update · `503` storage shard or dependent service unavailable.

### Internal APIs

**Coordinator** (port 8001 on the host): `POST /internal/nodes/register`, `POST /internal/nodes/heartbeat`,
`GET /internal/nodes`, `PUT|GET|DELETE /internal/data/{key}` (`PUT` supports `?if_version=N` compare-and-set; `0` = create only),
`GET /internal/data?prefix=`, `GET /internal/shard-of/{key}`, `GET /cluster`, `GET /health`.

**Storage nodes** (internal only): `GET /health`, `PUT|GET|DELETE /internal/data/{key}`, `GET /internal/data`,
`GET /internal/snapshot` (full dump used for recovery).

**Matching service** (internal only): `POST /match/{id}`, `GET /stats`, `GET /health`. The port isn't published and the
image has no `curl`, so query it through Python inside the container:
```bash
docker compose exec matching-service python -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8000/stats').read().decode())"
docker compose exec --index 2 matching-service python -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8000/stats').read().decode())"
```

### Background workers and scheduled jobs

| Where | Job | Interval |
|---|---|---|
| matching-service | RabbitMQ consumer on `matching.requests` (prefetch 2, manual ack) | continuous |
| matching-service | **Stale-pending sweeper**: re-queues auto-match requests idle longer than `STALE_PENDING_SECONDS` | every `SWEEP_INTERVAL` (10 s in Compose) |
| matching-service, request-service | RabbitMQ reconnect loop that re-declares the topology after any disconnect | retries every 3 s |
| storage node | Register + **heartbeat** to coordinator; resync from a peer when told to | every `HEARTBEAT_INTERVAL` (2 s) |
| coordinator | **Failure detector**: marks nodes `down` after `HEARTBEAT_TIMEOUT` without a heartbeat | checks every 1 s |

There are no cron jobs or external scheduled tasks.

### RabbitMQ topology

- Exchange `erms.requests` (topic) → queue `matching.requests` (durable, binding `request.#`).
- Exchange `erms.retry` (topic) → queue `matching.retry` (TTL = `RETRY_DELAY_MS`, dead-letters back to `erms.requests`).
- Messages are persistent JSON: `{"request_id": "...", "attempt": N}`.

### External services

The only external dependency is **RabbitMQ**, which runs as a container. No cloud services, API keys or third-party APIs
are used.

### Useful observability commands

```bash
curl -s localhost:8001/cluster | python3 -m json.tool    # node states, key counts, reads per node, failovers
docker compose logs -f coordinator | grep NODE           # NODE DOWN / NODE RECOVERED / state changes
docker compose logs storage-0a | grep SYNC               # "SYNC complete ... applied N updates" after recovery
docker compose logs -f matching-service | grep MATCHED    # matches as they happen
```
