# Deployment Assets

This directory contains deployment configuration for running this Nakama fork
in development, staging, and production environments.

Container images are published to GHCR by the [release workflow](../.github/workflows/release.yml)
as `ghcr.io/kaw-ai/nakama:<version>` (semver tags) and `ghcr.io/kaw-ai/nakama:sha-<commit>`.

## Layout

| Path | Purpose |
|------|---------|
| `config/dev.yml` | Local development configuration |
| `config/staging.yml` | Staging environment configuration template |
| `config/production.yml` | Production environment configuration template |
| `kubernetes/` | Kubernetes manifests (migration job, deployment, services) |

## Configuration and secrets

The YAML files under `config/` are **templates**. Nakama does not expand
environment variables inside YAML config files, so any value marked
`# SECRET:` must be supplied at deploy time by one of:

1. Overriding via command-line flags (flags take precedence over YAML), e.g.
   `--database.address "$DB_ADDRESS" --session.encryption_key "$SESSION_KEY"`.
2. Rendering the file from a secret manager (Vault, AWS Secrets Manager,
   Kubernetes Secrets) into the container before start.

Never commit real credentials to this repository.

## Database strategy

- **Development**: single-node CockroachDB via `docker-compose.yml` at the repo root.
- **Staging/Production**: a managed PostgreSQL (or CockroachDB) instance.
  Set the connection with `--database.address user:password@host:5432/nakama`.
- **Migrations**: always run `nakama migrate up` as an explicit pre-deploy step
  (see `kubernetes/migration-job.yaml`). The server rollout must only start
  after the migration job completes successfully.
- **Rollback**: `nakama migrate down --limit 1` reverses the most recent
  migration. Take a database backup/snapshot before every production migration
  so point-in-time restore is available if a rollout fails.

## Runtime modules

Custom Lua/JS modules live in `data/modules` and are mounted into the
container at `/nakama/data/modules` (see the Kubernetes deployment manifest
and `docker-compose.yml`). Go plugin modules must be built with
`heroiclabs/nakama-pluginbuilder` matching the server's Go toolchain
(**Go 1.22.5** for this release line) — a Go plugin built with a different Go
version or dependency set will fail to load. Version module artifacts with the
same tag as the server image they target.

## Network topology

| Port | Protocol | Purpose | Exposure |
|------|----------|---------|----------|
| 7349 | gRPC | Client API | Load balancer (TLS termination) |
| 7350 | HTTP/WebSocket | Client API + realtime sockets | Load balancer (TLS termination) |
| 7351 | HTTP | Developer console | Internal only (VPN / IP allowlist) |
| 9100 | HTTP | Prometheus metrics | Cluster-internal scrape only |

Terminate TLS at the load balancer or ingress; Nakama's built-in SSL support
is not recommended for production. `GET /` on port 7350 returns HTTP 200 and
is used as the health/readiness endpoint.

## Scaling notes

Open-source Nakama runs as a **single node per deployment** — clustering is an
enterprise feature. Scale vertically (CPU/memory) and size capacity from load
tests of realtime paths (sockets, matchmaker). Do not set `replicas > 1` on
the Kubernetes deployment unless the workloads are fully partitioned per node.

## Operations

See [docs/operations/runbooks.md](../docs/operations/runbooks.md) for restart,
scale, config change, migration, and rollback procedures, plus monitoring and
alerting guidance.
