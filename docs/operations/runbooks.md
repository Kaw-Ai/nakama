# Operations Runbooks

Operational procedures for running this Nakama fork in staging and production.
Deployment assets referenced here live in [`deploy/`](../../deploy/README.md).

## Monitoring

### Metrics

Nakama exposes Prometheus metrics on the port set by `metrics.prometheus_port`
(9100 in the provided configs). The Kubernetes deployment manifest carries
`prometheus.io/scrape` annotations; the root `docker-compose.yml` includes a
Prometheus service to model scrape config from.

Dashboards should track at minimum:

- **Sessions**: active socket/session counts, authentication rates.
- **Matches**: active match count, matchmaker queue size and match rate.
- **Database**: query latency, connection pool utilization, error rates.
- **API**: request rate, latency percentiles, gRPC/HTTP error rates.
- **Process**: CPU, memory, goroutines, GC pause times.

### Logging

All environments log structured JSON to stdout (`logger.format: JSON`). Ship
container stdout to the log aggregation platform (e.g. Loki, ELK, CloudWatch).
Levels: `DEBUG` in dev, `INFO` in staging, `WARN` in production.

### Alerting

Define alerts for:

| Alert | Condition | Severity |
|-------|-----------|----------|
| Health check failing | `GET /` on 7350 non-200 for > 1 min | Critical |
| Error-rate spike | API 5xx / gRPC error rate above baseline | High |
| DB connection saturation | Pool utilization > 80% sustained | High |
| Migration job failed | `nakama-migrate` job in failed state | Critical |
| Memory pressure | Container memory > 90% of limit | Warning |

## Runbooks

### Restart the server

```
kubectl rollout restart deployment/nakama -n nakama
```

The deployment uses `maxUnavailable: 0` with a termination grace period, so
in-flight socket connections drain before the old pod stops. Note: realtime
state (matches, parties, presences) on the restarting node is lost — schedule
restarts in low-traffic windows where possible.

### Deploy a new version

1. CI publishes `ghcr.io/kaw-ai/nakama:<version>` on tag push (see
   `.github/workflows/release.yml`).
2. Take a database backup/snapshot.
3. Update the image tag in `deploy/kubernetes/migration-job.yaml` and
   `deployment.yaml`, then run the migration job to completion:
   ```
   kubectl apply -f deploy/kubernetes/migration-job.yaml
   kubectl wait --for=condition=complete job/nakama-migrate -n nakama --timeout=300s
   ```
4. Roll out the server:
   ```
   kubectl apply -f deploy/kubernetes/deployment.yaml
   kubectl rollout status deployment/nakama -n nakama
   ```
5. Smoke test: authentication, storage read/write, chat, matchmaker,
   leaderboards, and console login.

### Roll back a failed deployment

1. Redeploy the previous image tag:
   ```
   kubectl rollout undo deployment/nakama -n nakama
   ```
2. If the new version applied schema migrations that the previous version
   cannot run against, reverse them with the **new** image:
   ```
   /nakama/nakama migrate down --limit 1 --database.address <address>
   ```
   or restore the pre-deploy database backup.
3. Verify health and metrics return to baseline.

### Change configuration

1. Edit the relevant template in `deploy/config/` and mirror it into the
   `nakama-config` ConfigMap (`deploy/kubernetes/configmap.yaml`).
2. Secrets are only changed in the secret manager, never in git.
3. Apply and restart:
   ```
   kubectl apply -f deploy/kubernetes/configmap.yaml
   kubectl rollout restart deployment/nakama -n nakama
   ```

### Scale

OSS Nakama is single-node; scale **vertically** by raising the CPU/memory
requests/limits in `deployment.yaml`. Size from load tests of the realtime
paths (socket connections, matchmaker throughput).

### Database backup and restore

- Schedule automated backups on the managed PostgreSQL/CockroachDB instance
  (daily full + point-in-time recovery where available).
- Always snapshot before running `nakama migrate up` in production.
- To restore: stop the Nakama deployment (`kubectl scale deployment/nakama
  --replicas=0`), restore the snapshot, verify schema version with
  `nakama migrate status`, then scale back up.

## Security notes

- TLS terminates at the load balancer/ingress for ports 7349 and 7350; the
  console (7351) is reachable only via VPN or IP allowlist.
- Rotate the console admin password and signing key per environment.
- Database connections should use TLS (`sslmode=require` in the address DSN
  for PostgreSQL).
- CI runs govulncheck and Trivy scans (`.github/workflows/security.yml`).
