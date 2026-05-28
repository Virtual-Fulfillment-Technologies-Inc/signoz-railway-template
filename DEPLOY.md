# Deploying SigNoz on Railway

Step-by-step guide for deploying SigNoz v0.122.0 on Railway using this template repository.

## Architecture Overview

```
                                    Your Applications
                                     |           |
                                  gRPC:4317   HTTP:4318
                                     |           |
                              +------+-----------+------+
                              |   otel-collector        |
                              |   (signoz-otel-collector|
                              |    v0.144.3)            |
                              +------+------------------+
                                     |
                              tcp://clickhouse:9000
                                     |
+-----------+                +-------+--------+            +----------+
|  SigNoz   | ---reads--->  |   ClickHouse   | <---zk---  | ZooKeeper|
|  v0.122.0 |               |    25.5.6      |            |  3.7.1   |
+-----------+                +----------------+            +----------+
                                     ^
                                     |
                              +------+--------+
                              |   Migrator    |  (one-shot, runs schema
                              | (otel-collector|  migrations then idles)
                              |  v0.144.3)    |
                              +---------------+
```

**5 Railway services** are needed (the migrator uses the same image as otel-collector):

| # | Service Name | Repo Path / Image | Purpose |
|---|---|---|---|
| 1 | **zookeeper** | `zookeeper/Dockerfile.zookeeper` (from repo) | Distributed coordination for ClickHouse |
| 2 | **clickhouse** | `clickhouse/Dockerfile.clickhouse` (from repo) | Columnar database for all telemetry data |
| 3 | **signoz-migrator** | `signoz/Dockerfile.migrator` (from repo) | One-shot schema migration service |
| 4 | **signoz** | `signoz/Dockerfile.signoz` (from repo) | Dashboard UI + query service + alertmanager |
| 5 | **otel-collector** | `signoz/Dockerfile.otel` (from repo) | Receives telemetry, writes to ClickHouse |

## Prerequisites

- A Railway account with a project created
- This repo connected to Railway (or forked to your GitHub)
- A RabbitMQ instance (if using the RabbitMQ metrics receiver)

## Step 1: Create the Railway Project

Create a new project in Railway. All 5 services will live in this project.

## Step 2: Deploy ZooKeeper

ZooKeeper is deployed from `zookeeper/Dockerfile.zookeeper`, which wraps the upstream `signoz/zookeeper:3.7.1` image to set `USER root`. This is required because Bitnami-based images default to user `1001`, which cannot write to Railway-mounted volumes (owned by root).

1. **Add service** > **GitHub Repo** > select this repository
2. **Root directory**: `zookeeper`
3. **Dockerfile path**: `Dockerfile.zookeeper`
4. **Service name**: `zookeeper`
5. **Environment variables**:
   ```
   ZOO_SERVER_ID=1
   ALLOW_ANONYMOUS_LOGIN=yes
   ZOO_AUTOPURGE_INTERVAL=1
   ```
6. **Volume**: Mount a persistent volume at `/bitnami/zookeeper`
7. **Networking**: No public domain needed. Private hostname will be `zookeeper.railway.internal`

Wait for ZooKeeper to be healthy before proceeding. If you see `mkdir: cannot create directory '/bitnami/zookeeper/data': Permission denied`, you're using the raw `signoz/zookeeper:3.7.1` image instead of the wrapped Dockerfile — switch to the repo's Dockerfile.

## Step 3: Deploy ClickHouse

1. **Add service** > **GitHub Repo** > select this repository
2. **Root directory**: `clickhouse`
3. **Dockerfile path**: `Dockerfile.clickhouse`
4. **Service name**: `clickhouse`
5. **Environment variables**:
   ```
   CLICKHOUSE_SKIP_USER_SETUP=1
   ```
6. **Volume**: Mount a persistent volume at `/var/lib/clickhouse/`
7. **Networking**: No public domain needed. Private hostname will be `clickhouse.railway.internal`

Wait for ClickHouse to be healthy. You can verify by checking logs for `Ready for connections`.

## Step 4: Deploy the Migrator

The migrator creates ClickHouse databases and runs schema migrations. It must complete before SigNoz or the otel-collector can function.

1. **Add service** > **GitHub Repo** > select this repository
2. **Root directory**: `signoz`
3. **Dockerfile path**: `Dockerfile.migrator`
4. **Service name**: `signoz-migrator`
5. **Environment variables**:
   ```
   SIGNOZ_OTEL_COLLECTOR_CLICKHOUSE_DSN=tcp://clickhouse.railway.internal:9000
   SIGNOZ_OTEL_COLLECTOR_CLICKHOUSE_CLUSTER=cluster
   SIGNOZ_OTEL_COLLECTOR_CLICKHOUSE_REPLICATION=true
   SIGNOZ_OTEL_COLLECTOR_TIMEOUT=10m
   ```
6. **No volume** needed
7. **No public domain** needed

Wait for the migrator logs to show `All migrations complete. Idling.` before proceeding.

If the migrator fails because ClickHouse isn't ready yet, **redeploy** the migrator service. The migration commands are idempotent.

## Step 5: Deploy SigNoz

1. **Add service** > **GitHub Repo** > select this repository
2. **Root directory**: `signoz`
3. **Dockerfile path**: `Dockerfile.signoz`
4. **Service name**: `signoz`
5. **Environment variables**:
   ```
   SIGNOZ_ALERTMANAGER_PROVIDER=signoz
   SIGNOZ_TELEMETRYSTORE_CLICKHOUSE_DSN=tcp://clickhouse.railway.internal:9000
   SIGNOZ_SQLSTORE_SQLITE_PATH=/var/lib/signoz/signoz.db
   SIGNOZ_TOKENIZER_JWT_SECRET=<generate-a-strong-secret>
   ```
   **IMPORTANT**: Generate a real JWT secret (e.g., `openssl rand -hex 32`). Do not use `secret`.
6. **Volume**: Mount a persistent volume at `/var/lib/signoz/`
7. **Public domain**: Enable — this is how you access the SigNoz dashboard
8. **Port**: `8080`

## Step 6: Deploy the OTel Collector

1. **Add service** > **GitHub Repo** > select this repository
2. **Root directory**: `signoz`
3. **Dockerfile path**: `Dockerfile.otel`
4. **Service name**: `otel-collector`
5. **Environment variables**:
   ```
   OTEL_RESOURCE_ATTRIBUTES=host.name=signoz-host,os.type=linux
   LOW_CARDINAL_EXCEPTION_GROUPING=false
   SIGNOZ_OTEL_COLLECTOR_CLICKHOUSE_DSN=tcp://clickhouse.railway.internal:9000
   SIGNOZ_OTEL_COLLECTOR_CLICKHOUSE_CLUSTER=cluster
   SIGNOZ_OTEL_COLLECTOR_CLICKHOUSE_REPLICATION=true
   SIGNOZ_OTEL_COLLECTOR_TIMEOUT=10m
   RABBITMQ_ENDPOINT=http://rabbitmq.railway.internal:15672
   RABBITMQ_USERNAME=<your-rabbitmq-username>
   RABBITMQ_PASSWORD=<your-rabbitmq-password>
   ```
   If you are not using RabbitMQ monitoring, remove the `RABBITMQ_*` variables and edit `otel-collector-config.yaml` to remove the `rabbitmq` receiver and its reference in the `metrics` pipeline.
6. **No volume** needed
7. **Public domain**: Enable if you need external applications to send telemetry. Map port `4317` (gRPC) and/or `4318` (HTTP).
   For internal-only ingestion (apps in the same Railway project), use `otel-collector.railway.internal:4317` — no public domain needed.

The collector will run `migrate sync check` on startup, which retries with exponential backoff until migrations are confirmed complete. If the migrator hasn't finished yet, the collector will wait.

## Post-Deployment Checklist

### 1. Verify Services Are Healthy

Check Railway dashboard — all 5 services should show as running:
- **zookeeper**: Running
- **clickhouse**: Running, logs show `Ready for connections`
- **signoz-migrator**: Running, logs show `All migrations complete. Idling.`
- **signoz**: Running, accessible via public domain
- **otel-collector**: Running, logs show collector started

### 2. Access SigNoz Dashboard

Open the public domain assigned to the `signoz` service. Create your first user account on initial visit.

### 3. Configure Your Applications

Point your applications' OpenTelemetry SDK to the otel-collector:

**Internal (same Railway project)**:
```
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.railway.internal:4317
```

**External (public domain)**:
```
OTEL_EXPORTER_OTLP_ENDPOINT=https://<your-public-domain>
```

### 4. Verify RabbitMQ Metrics (if configured)

After the collector starts, check the SigNoz dashboard under **Metrics** for RabbitMQ-related metrics (queue message counts, node memory, etc.). If no metrics appear:
- Verify `RABBITMQ_ENDPOINT` is `http://rabbitmq.railway.internal:15672` (management API, NOT AMQP port 5672)
- Verify the RabbitMQ instance uses the `rabbitmq:management` image variant
- Check otel-collector logs for receiver errors

## Troubleshooting

### "connection refused" to ClickHouse during migrations

**Cause**: ClickHouse isn't ready or isn't running.

**Fix**: Check ClickHouse logs. Ensure the volume is mounted at `/var/lib/clickhouse/`. Redeploy the migrator after ClickHouse is healthy.

### otel-collector stuck on "sync check"

**Cause**: Migrations haven't been applied yet.

**Fix**: Check the migrator service logs. If it shows errors, redeploy it after confirming ClickHouse is healthy. The otel-collector will automatically proceed once `sync check` passes.

### SigNoz dashboard shows no data

**Cause**: Either the otel-collector isn't running, or applications aren't sending telemetry.

**Fix**:
1. Verify otel-collector logs show it started successfully
2. Verify your application's `OTEL_EXPORTER_OTLP_ENDPOINT` points to the collector
3. Check that the collector can reach ClickHouse (`clickhouse.railway.internal:9000`)

### Migrator keeps restarting

**Cause**: ClickHouse wasn't reachable when the migrator started.

**Fix**: The migrator will succeed on retry since Railway restarts failed containers. If it persists, check ClickHouse health first, then redeploy the migrator.

## Deployment Order Summary

If doing a fresh deploy or full redeploy, always follow this order:

```
1. ZooKeeper        (wait until healthy)
2. ClickHouse       (wait until "Ready for connections")
3. signoz-migrator  (wait until "All migrations complete")
4. SigNoz           (wait until dashboard accessible)
5. otel-collector   (sync check will gate startup automatically)
```

## Version Matrix

All SigNoz component versions must be kept in sync:

| Component | Image | Version |
|---|---|---|
| SigNoz | `signoz/signoz` | `v0.122.0` |
| OTel Collector | `signoz/signoz-otel-collector` | `v0.144.3` |
| Migrator | `signoz/signoz-otel-collector` | `v0.144.3` |
| ClickHouse | `clickhouse/clickhouse-server` | `25.5.6` |
| ZooKeeper | `signoz/zookeeper` | `3.7.1` |

When upgrading, update all SigNoz images together and redeploy in the order above.
