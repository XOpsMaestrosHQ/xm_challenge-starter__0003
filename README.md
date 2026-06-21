# Rust Microservice Starter

This project is a platform-engineered microservices starter template for building distributed
systems with Rust. It is built for engineering teams that care about platform integrity and wish(as
much as possible) to avoid integrating external/third-party tooling for their distributed system
builds.

The base is intentionally minimal, with two services:

- `mesh` - a registry and control plane for service registration, heartbeat refresh, load-balanced
  service discovery, and more. It is adapted from
  [Rusty Mesh](https://github.com/okpainmo/rusty-mesh).
- `auth` - a PostgreSQL-backed auth/session/RBAC service, adapted from
  [Rusty Auth](https://github.com/okpainmo/rusty-auth).

## Table Of Contents

- [System Shape](#system-shape)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Runtime URLs](#runtime-urls)
- [How Service Discovery Works](#how-service-discovery-works)
- [Configuration](#configuration)
- [Common Operations](#common-operations)
- [Development Workflow](#development-workflow)
- [CI](#ci)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Repository Layout](#repository-layout)

## System Shape

The root Compose stack runs the project as one unit:

```text
Host
|-- 127.0.0.1:3080 -> mesh
|-- 127.0.0.1:8000 -> auth
`-- 127.0.0.1:5433 -> auth-db

Compose network
|-- mesh     -> service registry/control-plane
|-- auth     -> auth microservice; registers into mesh
`-- auth-db  -> PostgreSQL for auth
```

The `mesh` owns service registration, heartbeat refresh, discovery, and endpoint metadata. The
`auth` service on the other hand, starts after PostgreSQL is healthy, registers itself with the
mesh, refreshes its lease by heartbeat, and unregisters during graceful shutdown.

Mesh endpoint resolution for development is intentionally configured as:

```text
APP__REGISTRY__EXTERNAL_ENDPOINT_RESOLUTION=docker
```

That means Mesh inspects Docker to resolve the host port published for a registering container. The
root Compose file mounts `/var/run/docker.sock` into Mesh read-only for that purpose. Learn more
about this by reading the [rusty-mesh documentation](https://github.com/okpainmo/rusty-mesh).

## Requirements

- Docker with the Compose plugin
- Rust, for local service development outside Docker
- Bun, for project root static-analysis tooling
- `sqlx-cli`, only when running auth migrations manually outside Compose

The fastest path to a complete project start-up is Docker Compose. For that, Rust and the `sqlx-cli`
are not required.

## Quick Start

Copy the root environment sample:

```bash
cp .env.sample .env
```

Start the full stack:

```bash
docker compose up -d --build
```

For faster Compose builds on newer Docker setups, you can enable Bake:

```bash
COMPOSE_BAKE=true docker compose up -d --build
```

The first build can take several minutes because the Rust services compile their release
dependencies inside Docker. Later builds should reuse Docker and Cargo build cache layers.

Check running containers:

```bash
docker compose ps
```

Check Mesh health:

```bash
curl http://127.0.0.1:3080/api/v1/mesh/health
```

List services registered with Mesh:

```bash
curl -H "authorization: Bearer ${MESH_TOKEN:-local-dev-mesh-token}" \
  http://127.0.0.1:3080/api/v1/mesh/services
```

You should see `auth-service` after Auth has started and registered.

If `auth` takes a little longer to appear, check its logs while it waits for PostgreSQL and
registers with the `mesh`:

```bash
docker compose logs -f auth
```

Smoke-test Auth by registering a disposable user:

```bash
curl -X POST http://127.0.0.1:8000/api/v1/auth/register \
  -H "content-type: application/json" \
  -d '{
    "first_name": "Smoke",
    "last_name": "User",
    "email": "smoke@example.com",
    "password": "password123",
    "country": "Testland",
    "country_code": "TL",
    "phone_number": "1000000001"
  }'
```

Stop the stack:

```bash
docker compose down
```

Remove the database volume when you want a clean database:

```bash
docker compose down -v
```

## Runtime URLs

| Service       | URL                                          | Notes                            |
| ------------- | -------------------------------------------- | -------------------------------- |
| Mesh health   | `http://127.0.0.1:3080/api/v1/mesh/health`   | Public health route              |
| Mesh registry | `http://127.0.0.1:3080/api/v1/mesh/services` | Requires mesh bearer token       |
| Auth API      | `http://127.0.0.1:8000/api/v1/auth`          | Public and protected auth routes |
| Auth database | `127.0.0.1:5433`                             | Local host mapping to Postgres   |

## How Service Discovery Works

Auth is configured in root Compose with:

```text
APP__MESH__ENABLED=true
APP__MESH__URL=http://mesh:3080
APP__MESH__SERVICE_NAME=auth-service
APP__MESH__SERVICE_VERSION=1.0.0
APP__MESH__ADVERTISE_HOST=auth
```

When Auth starts:

1. Auth binds inside the container on port `8000`.
2. Auth sends a registration request to Mesh over the Compose network.
3. Auth includes its container id through `x-mesh-container-id`.
4. Mesh inspects Docker because endpoint resolution is `docker`.
5. Mesh stores both endpoint views:
   - external host endpoint, for operator access from the host
   - internal Compose-network endpoint, for service-to-service calls
6. Auth refreshes the lease through heartbeat every `MESH_HEARTBEAT_INTERVAL_SECS`.

The public registry response should include fields like:

```json
{
  "name": "auth-service",
  "version": "1.0.0",
  "ip": "127.0.0.1",
  "port": 8000,
  "internal_ip": "auth",
  "internal_port": 8000,
  "url": "http://127.0.0.1:8000"
}
```

If a service should call another service inside the Compose network, discovery clients can request
the internal endpoint by sending:

```http
x-mesh-endpoint-scope: internal
```

## Configuration

Root runtime values live in [.env.sample](.env.sample). Copy it to `.env` and edit the values for
your environment.

Important root variables:

| Variable                       | Default                       | Purpose                                          |
| ------------------------------ | ----------------------------- | ------------------------------------------------ |
| `MESH_HTTP_PORT`               | `3080`                        | Host port for Mesh                               |
| `MESH_TOKEN`                   | `local-dev-mesh-token`        | Shared token for protected Mesh routes           |
| `MESH_PUBLIC_HOST`             | `127.0.0.1`                   | Host used for Docker-resolved external endpoints |
| `MESH_HEARTBEAT_INTERVAL_SECS` | `5`                           | Service heartbeat interval                       |
| `MESH_SERVICE_TTL_SECS`        | `15`                          | Registry lease TTL                               |
| `AUTH_HTTP_PORT`               | `8000`                        | Host port for Auth                               |
| `AUTH_JWT_SECRET`              | `change-me-before-production` | Auth JWT signing secret                          |
| `AUTH_POSTGRES_PORT`           | `5433`                        | Host port for Auth PostgreSQL                    |
| `AUTH_POSTGRES_USER`           | `rusty_auth`                  | Auth DB user                                     |
| `AUTH_POSTGRES_PASSWORD`       | `rusty_auth`                  | Auth DB password                                 |
| `AUTH_POSTGRES_DB`             | `rusty_auth`                  | Auth DB name                                     |

Each service also keeps its own standalone configuration and README:

- [microservices/mesh/README.md](microservices/mesh/README.md)
- [microservices/auth/README.md](microservices/auth/README.md)

Use the service READMEs when working inside a service directly. Use this root README when operating
the full project as a composed system.

## Common Operations

Build without starting:

```bash
docker compose build
```

Start in the foreground:

```bash
docker compose up --build
```

Start in the background:

```bash
docker compose up -d --build
```

Start in the background with Compose Bake enabled:

```bash
COMPOSE_BAKE=true docker compose up -d --build
```

View logs:

```bash
docker compose logs -f mesh
docker compose logs -f auth
docker compose logs -f auth-db
```

Restart one service:

```bash
docker compose restart auth
```

Rebuild one service:

```bash
docker compose up -d --build auth
```

Inspect the effective Compose configuration:

```bash
docker compose config
```

Open a Postgres shell:

```bash
docker compose exec auth-db psql -U "${AUTH_POSTGRES_USER:-rusty_auth}" \
  -d "${AUTH_POSTGRES_DB:-rusty_auth}"
```

## Development Workflow

Install root tooling:

```bash
bun install
```

Root scripts:

```bash
bun run format
bun run format:check
bun run services:check
bun run services:test
```

`services:check` discovers Rust services under `microservices/*/Cargo.toml` and runs:

- `cargo check --locked --all-targets --all-features`
- `cargo fmt --all -- --check`
- `cargo clippy --locked --all-targets --all-features -- -D warnings`

`services:test` runs full tests for services that do not need a database. For services with
migrations, it always runs library tests and runs DB-backed controller tests when PostgreSQL is
reachable.

Local hooks are installed by Husky:

- `pre-commit` validates shell hooks, Rust service quality checks, and repository formatting.
- `pre-push` runs service tests and repository formatting checks.
- `commit-msg` enforces Conventional Commit messages.

## CI

GitHub Actions validates the root and each integrated service:

- root formatting and shell-hook validation
- Rust quality checks for Auth and Mesh
- full Mesh tests
- full Auth tests with PostgreSQL and SQLx migrations
- Docker image builds for Auth and Mesh

The root project is not a Rust workspace, so CI intentionally enters each service directory instead
of running one Cargo command from the repository root.

## Security Notes

Change these before any shared or production-like deployment:

- `MESH_TOKEN`
- `AUTH_JWT_SECRET`
- `AUTH_POSTGRES_PASSWORD`

Mesh registry routes are protected by the mesh token. The health route remains public so load
balancers and operators can check liveness without holding internal credentials.

Docker endpoint resolution requires Docker socket access. Treat this as privileged control-plane
access. The socket is mounted read-only in the Compose file, but the Docker Engine API can still
expose sensitive runtime metadata. Keep this mode for trusted local or controlled deployments where
Mesh is allowed to inspect service containers.

For stricter environments, switch Mesh endpoint resolution to `none` and have services register
explicit external endpoint fields instead.

## Troubleshooting

Mesh registry returns `401`:

- Confirm `MESH_TOKEN` in `.env`.
- Send `Authorization: Bearer <token>` to `/api/v1/mesh/services` routes.

Auth does not appear in Mesh:

- Check `docker compose logs -f auth`.
- Check that `APP__MESH__ENABLED=true` in the rendered Compose config.
- Check that Auth can reach `http://mesh:3080` inside the Compose network.
- Confirm Mesh started with `APP__REGISTRY__EXTERNAL_ENDPOINT_RESOLUTION=docker`.

Mesh shows internal endpoint instead of host endpoint:

- Confirm `/var/run/docker.sock` is mounted into the Mesh container.
- Confirm the registering service publishes its internal port through Compose.
- Confirm the service sends `x-mesh-container-id`; Auth does this through its registry client.

Auth cannot connect to PostgreSQL:

- Check `docker compose ps auth-db`.
- Check `docker compose logs -f auth-db`.
- Confirm Auth DB environment variables match the Postgres service values.
- Reset the database volume if the existing volume was created with different credentials:

```bash
docker compose down -v
docker compose up -d --build
```

Port already in use:

- Change `MESH_HTTP_PORT`, `AUTH_HTTP_PORT`, or `AUTH_POSTGRES_PORT` in `.env`.
- Rerun `docker compose up -d`.

## Repository Layout

```text
.
|-- .github/              # CI, pull request template, issue templates
|-- .husky/               # local Git hooks
|-- compose.yaml          # whole-project runtime stack
|-- microservices/
|   |-- auth/             # Rusty Auth service
|   `-- mesh/             # Rusty Mesh service
|-- scripts/              # root service-check orchestration
|-- .env.sample           # root Compose environment sample
|-- CONTRIBUTING.md
|-- SECURITY.md
|-- CODE_OF_CONDUCT.md
|-- LICENSE
`-- README.md
```
