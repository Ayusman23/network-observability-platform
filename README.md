# Network Observability Platform

> A full-stack, extensible network observability platform — agents that collect network metrics, a Go orchestration API and real-time service, Python analysis agents and alerting, a React + Tailwind dashboard, and recommended deployment using Docker & Kubernetes.

---

## Project overview

**Name:** Network Observability Platform (NOP)

**Goal:** Provide an end-to-end observability solution for network engineers and SREs. NOP runs lightweight Python agents to collect network checks (ping, DNS, HTTP), streams metrics to a backend (Prometheus + optional Kafka), exposes REST/GraphQL APIs via a Go orchestration service, and presents real-time dashboards, topology visualization, and alerts in a React frontend.

**Key features**

* Agent-driven monitoring (Ping, HTTP, DNS, traceroute logs)
* Real-time metrics (latency, packet loss, throughput) stored in Prometheus
* Agent orchestration and configuration via Go service (start/stop/update agents)
* Alerts and notification engine (rules-based) with Slack / Email hooks
* Interactive network topology visualization in the web UI
* Search/filter nodes, flows, and alerts
* WebSocket-based live updates + Redis pub/sub
* Docker Compose local dev and Kubernetes production manifests

---

## Repository structure

```
/frontend        -> React.js + Tailwind + ShadCN UI
/backend         -> Go orchestration service (REST/GraphQL, WebSocket)
/agents          -> Python agents (ping/http/dns) + alerting
/db              -> PostgreSQL schema + migrations
/deploy          -> Docker Compose, Kubernetes manifests, Helm charts (optional)
/docs            -> design notes, API specs, runbooks
/scripts         -> helper scripts (dev DB init, seed, build)
```

---

## Prerequisites (local dev)

* OS: Linux / macOS (Windows WSL2 recommended)
* [VS Code](https://code.visualstudio.com/) with recommended extensions (see below)
* Node.js LTS (18.x or 20.x)
* npm or pnpm
* Go 1.20+ (recommended latest stable)
* Python 3.10+
* Docker & Docker Compose
* kubectl (for Kubernetes dev)
* PostgreSQL 13+ (or run via Docker)
* Prometheus + Grafana (local via Docker Compose)
* Redis (local via Docker Compose)
* (Optional) Kafka & Zookeeper (for event streaming)

---

## Recommended VS Code extensions

* ES7+ React/Redux/GraphQL/React-Native snippets
* Tailwind CSS IntelliSense
* Prisma (if used) or SQLTools
* Go, Python
* Docker
* Kubernetes
* Prettier
* ESLint

---

## Quickstart — development (Docker Compose)

`deploy/docker-compose.dev.yml` contains a single-command dev environment for local testing. It includes: Postgres, Redis, Prometheus, Grafana, Go backend, Python agents (as a service), and frontend.

1. Clone repo

```bash
git clone https://github.com/<you>/network-observability-platform.git
cd network-observability-platform
```

2. Copy example env files

```bash
cp backend/.env.example backend/.env
cp agents/.env.example agents/.env
cp frontend/.env.example frontend/.env
```

3. Start dev stack

```bash
docker compose -f deploy/docker-compose.dev.yml up --build
```

4. Visit

* Frontend: [http://localhost:3000](http://localhost:3000)
* Grafana: [http://localhost:3001](http://localhost:3001) (default creds in deploy readme)
* Prometheus UI: [http://localhost:9090](http://localhost:9090)
* Backend API: [http://localhost:8080](http://localhost:8080)

---

## Frontend (React) — quick dev

**Stack:** React, Vite, TailwindCSS, ShadCN UI, TanStack Query, Recharts (or vis-network / cytoscape for topology), Zustand (or Redux) for state.

1. Install

```bash
cd frontend
npm install
```

2. Run dev server

```bash
npm run dev
```

3. Important files

* `src/pages` — page routes (Login, Dashboard, Topology, Alerts)
* `src/components` — UI components (TopologyGraph, MetricsChart, AlertsPanel)
* `src/api` — client wrappers for REST/GraphQL + WebSocket
* `src/lib/socket.ts` — WebSocket connection using token auth

4. Packages to install (example)

```
react, react-dom, react-router-dom, axios, @tanstack/react-query,
tailwindcss, @shadcn/ui (or shadcn libs), zustand, cytoscape, recharts
```

---

## Backend (Go) — quick dev

**Responsibilities**

* Provide REST (and GraphQL optional) APIs for frontend
* Orchestrate agents (start/stop/configure) via RPC or via control topics in Redis/Kafka
* Serve WebSocket endpoints for real-time metric streams
* Persist users, config, alert rules in PostgreSQL
* Forward metrics to Prometheus Pushgateway (or expose metrics endpoint scraped by Prometheus)

**Tech choices**

* `gorilla/mux` or `chi` for routing
* `graphql-go` or `gqlgen` if GraphQL is desired
* `gorm` or `pgx` for database access
* `promhttp` for Prometheus metrics endpoint
* `gorilla/websocket` for WebSocket
* `redis` client for pub/sub

1. Install deps

```bash
cd backend
go mod download
```

2. Build & run

```bash
go run ./cmd/server
# or
go build -o bin/nop ./cmd/server && ./bin/nop
```

3. Env vars (backend/.env)

```
PORT=8080
DATABASE_URL=postgres://nop:nop@db:5432/nop_db?sslmode=disable
REDIS_URL=redis:6379
PROM_PUSHGATEWAY=http://prometheus:9091
JWT_SECRET=supersecret
```

4. Endpoints (examples)

* `POST /api/v1/auth/signup` — create user
* `POST /api/v1/auth/login` — returns JWT
* `GET /api/v1/agents` — list agents
* `POST /api/v1/agents` — create/start agent
* `PATCH /api/v1/agents/:id` — update config
* `DELETE /api/v1/agents/:id` — stop/remove
* `GET /api/v1/metrics` — aggregated metrics
* `GET /ws/stream` — WebSocket stream (requires JWT)

---

## Agents (Python)

**Responsibilities**

* Execute checks (ping, http, dns)
* Collect metrics & logs
* Push metrics to Prometheus Pushgateway OR publish to Kafka / Redis
* Accept remote config via HTTP or subscribe to Redis commands

**Tech choices**

* `asyncio` + `aiohttp` for HTTP checks
* `scapy` or `icmplib` for ping/traceroute (use icmplib for portability)
* `prometheus_client` for metrics push
* `aioredis` for control pub/sub

1. Python env & deps

```
python -m venv .venv
source .venv/bin/activate
pip install -r agents/requirements.txt
```

`agents/requirements.txt` should include:

```
aiohttp
prometheus_client
aioredis
icmplib
python-dotenv
```

2. Example agent CLI

```bash
cd agents
python agent.py --config configs/agent-nyc.yaml
```

3. Example `agent.py` behaviors

* Periodically run configured checks
* Expose a small HTTP server `/health` and `/metrics` (so Prometheus can scrape the agent directly if desired)
* Push metrics to Pushgateway or publish events to Kafka/Redis

---

## Database (Postgres)

**Schema highlights**

* `users` — id, name, email, password\_hash, role
* `agents` — id, name, type, owner\_id, config (jsonb), status
* `alerts` — id, rule\_id, agent\_id, severity, state, created\_at
* `alert_rules` — id, name, expression (json/DSL), severity, enabled

**Migrations**
Use `golang-migrate`, `migrate` CLI, or `Flyway`. Place migrations in `/db/migrations` and run during dev start or CI.

Example migration command:

```bash
migrate -path db/migrations -database "${DATABASE_URL}" up
```

---

## Metrics & Observability

* Prometheus scrapes metrics from either agents (pull) or scrapes backend metrics exposed by Go using `promhttp`.
* Agents can optionally push via Prometheus Pushgateway (push) for short-lived jobs.
* Grafana connects to Prometheus to show advanced dashboards.
* The frontend reads aggregated metrics via backend APIs plus real-time via WebSocket.

---

## Alerting Engine

Two possible architectures:

1. Simple rules engine in backend (Go): evaluate time-series from Prometheus via PromQL and send notifications.
2. Dedicated Python service: subscribes to metrics events (Redis/Kafka), runs rules (using `numpy/pandas` or `prometheus API`), and triggers notifications.

Notifications supported:

* Email (SMTP)
* Slack webhooks
* PagerDuty (optional)

---

## Docker & Kubernetes

**Local dev**: use `deploy/docker-compose.dev.yml` to run services easily.

**Kubernetes**: manifests in `deploy/k8s/` include:

* `backend-deployment.yaml`, `backend-service.yaml`
* `frontend-deployment.yaml`, `frontend-service.yaml`
* `agents` as a Deployment/DaemonSet (depending on topology)
* `postgres` StatefulSet + PVC
* `redis`, `prometheus`, `grafana` (or helm charts)

**Helm**: Consider packaging backend & agents as charts for easier configuration.

---

## CI / CD

* Use GitHub Actions to run tests and build Docker images.
* Push images to Docker Hub or a registry (GHCR, ECR).
* Deploy to Kubernetes via `kubectl apply` or use Argo CD for GitOps.

---

## Security & Auth

* JWT for frontend <-> backend traffic
* mTLS between agents and backend (optional for production)
* Store secrets in Kubernetes secrets or Vault
* Rate limit APIs and protect WebSocket endpoints

---

## Testing

* Unit tests for Go and Python services
* Integration tests that run docker-compose and execute sample agent checks
* Frontend e2e tests with Playwright or Cypress

---

## Troubleshooting

* If DB fails: `docker logs postgres` and check migrations
* If Prometheus not scraping: check targets page in Prometheus UI
* WebSocket issues: confirm JWT header and backend logs

---

## Useful scripts (in `/scripts`)

* `scripts/dev-setup.sh` — boot Docker compose + wait-for services
* `scripts/run-migrations.sh` — run DB migrations
* `scripts/seed-sample-data.py` — create demo agents and users

---

## API contract (short)

### Auth

* `POST /api/v1/auth/signup` — body: `{name,email,password}`
* `POST /api/v1/auth/login` — body: `{email,password}` returns `{token}`

### Agents

* `GET /api/v1/agents`
* `POST /api/v1/agents` — create agent (body: name, type, config)
* `POST /api/v1/agents/:id/actions` — actions like start/stop

### Metrics

* `GET /api/v1/metrics?agent_id=&metric=&from=&to=`
* `GET /ws/stream` — upgrade to websocket; send JWT as `Authorization: Bearer <token>`

---

## Next steps (suggested deliverables)

1. Generate scaffold code for each component (frontend, backend, agents).
2. Provide working minimal example: one Python agent that pings `8.8.8.8`, pushes metrics to Pushgateway, a Go server that lists agents and serves a WebSocket echo, and a React dashboard showing live metric updates.
3. Expand to full feature set: topology graph, alerting rules editor, authentication flow, DB migrations.

---

## License

MIT

---

## Contribution

PRs welcome. Please follow commit message guidelines and create a ticket for major features.

---

*README generated by ChatGPT — if you want, I can now scaffold the repository files (starter code) for each folder: `/frontend`, `/backend`, `/agents`, `/db`, and `/deploy`. Tell me which component to scaffold first or I can scaffold all with minimal working examples.*
