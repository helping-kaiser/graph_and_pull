# Development Guide

## Prerequisites

| Tool | Purpose | Install |
|---|---|---|
| Rust (stable) | Language toolchain | https://rustup.rs |
| Docker + Compose | Local databases | https://docs.docker.com/get-docker |
| sqlx-cli | Running migrations | `cargo install sqlx-cli --no-default-features --features postgres` (or use `make init`) |

Verify everything is in place:
```bash
rustc --version        # >= 1.75
cargo --version
docker --version
docker compose version
sqlx --version
```

---

## First-Time Setup

```bash
# Everything in one command: copies .env, installs sqlx-cli, starts DBs,
# runs migrations, starts the API
make run
```

Or step by step:
```bash
make init         # copy .env, check/install dependencies
make dev          # start DBs + migrate + start API
```

The API will be available at `http://localhost:8080`.
GraphQL playground: `http://localhost:8080/playground`.
Memgraph Lab (visual graph browser): `http://localhost:3000`.

---

## Environment Variables

All variables are in `.env` (gitignored, copied from `.env.example`).

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgres://gnp:gnp_secret@localhost:5432/gnp_db` | Full Postgres connection URL (used by sqlx-cli and the app) |
| `POSTGRES_USER` | `gnp` | Postgres username (used by Docker and Makefile) |
| `POSTGRES_PASSWORD` | `gnp_secret` | Postgres password |
| `POSTGRES_DB` | `gnp_db` | Postgres database name |
| `POSTGRES_PORT` | `5432` | Exposed host port |
| `MEMGRAPH_HOST` | `localhost` | Memgraph bolt host |
| `MEMGRAPH_PORT` | `7687` | Memgraph bolt port |
| `API_HOST` | `0.0.0.0` | API bind address |
| `API_PORT` | `8080` | API bind port |
| `RUST_LOG` | `debug` | Log level filter (`trace`, `debug`, `info`, `warn`, `error`) |

---

## Make Commands

```
make init         First-time setup: copy .env, check/install dependencies
make run          Full start: init + dev (first-time friendly)
make dev          Start DBs + migrate + start API
make api          Start the API server only
make up           Start Postgres + Memgraph in background
make down         Stop all services (data persists in volumes)
make reset-db     Wipe all volumes, restart services, re-run migrations
make migrate      Run pending Postgres migrations only
make ci           Full CI pipeline: lint then test
make lint         cargo clippy + cargo fmt --check (read-only)
make fmt          cargo fmt --all (writes files)
make test         cargo test --all
make build        cargo build --all
make logs         Follow docker compose logs (Ctrl+C to stop)
```

---

## Database Tools

### Memgraph Lab

Available at http://localhost:3000 when services are running. Lets you:
- Run Cypher queries interactively
- Visualize the graph with a node/edge explorer
- Inspect schema and indexes

Useful queries to get started:
```cypher
-- See all nodes
MATCH (n) RETURN n LIMIT 50;

-- See all relationships
MATCH ()-[r]->() RETURN r LIMIT 50;

-- Show the schema
CALL schema.node_type_properties();
```

### Postgres

Connect with any Postgres client using credentials from `.env`:
```
host:     localhost
port:     5432
user:     gnp
password: gnp_secret
database: gnp_db
```

Or via Docker:
```bash
docker exec -it gnp_postgres psql -U gnp -d gnp_db
```

---

## Migrations

Migrations live in `migrations/` and are managed by sqlx-cli.

```bash
# Create a new migration
sqlx migrate add <name>

# Run pending migrations
make migrate

# Revert is not supported by SQLx by default — write down migrations manually
```

Migration files are numbered and named, e.g. `20240101000000_create_users.sql`.

SQLx compile-time query checking (`sqlx::query!` macros) requires a live database or a `.sqlx/` cache directory. During development, keep `make up` running. In CI, the database service is started before the build step.

---

## Running Tests

```bash
# All tests
make test

# Single crate
cargo test -p graph-engine

# Single test
cargo test -p postgres-store test_name

# With output
cargo test -- --nocapture
```

Integration tests that hit the databases require services to be running (`make up`).

---

## Code Style

- `cargo fmt` enforced in CI — run `make fmt` before committing
- `clippy -D warnings` enforced in CI — run `make lint` to check
- No `unwrap()` in library code — use `thiserror` / `anyhow` appropriately
- Cypher queries in `graph-engine` only, SQL in `postgres-store` only
