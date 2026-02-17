# Deploy and Host SigNoz on Railway

**[SigNoz](https://signoz.io)** is an open-source observability platform that enables you to collect, store, and analyze distributed application **traces, metrics, and logs** using the OpenTelemetry standard.

## About Hosting SigNoz

When you deploy SigNoz on Railway, the following core services are provisioned:
- SigNoz
- SigNoz Otel Collector
- ClickHouse
- Zookeeper

The Railway template automatically sets up these services with the necessary environment variables, health checks, and persistent storage. This allows you to quickly go from deployment to creating dashboards. Simply point your application's OpenTelemetry SDK or agent to the provided ingest URL, and SigNoz will immediately begin visualizing service dependencies, latency, and errors.

## Common Use Cases

- **Application Performance Monitoring**: Monitor metrics, logs, and traces across your entire Railway application stack.
- **Debugging and Troubleshooting**: Correlate logs, metrics, and traces to quickly identify and resolve issues.
- **Infrastructure Observability**: Monitor system health, resource usage, and service dependencies in real time.
- **Alerting and Incident Response**: Set up alerts based on metrics and log patterns for proactive incident management.

## Dependencies for SigNoz Hosting

- **Persistent Storage**: Use a Railway volume (or external block storage) for ClickHouse and SigNoz data.
- **Ingest Traffic**: Applications should export OpenTelemetry traces, metrics, or logs over HTTP or gRPC.

### Deployment Dependencies

- [SigNoz Documentation](https://signoz.io/docs/)
- [OpenTelemetry Specification](https://opentelemetry.io/docs/)
- [ClickHouse Server](https://clickhouse.com/docs/en/)

### Implementation Details

To run the SigNoz stack on Railway, ensure the following:

#### OpenTelemetry Ingestion
- You may need to configure **Domains / Proxy** settings in Railway for the `signoz-otel-collector` service, depending on your use case.  
- Port **4317** is open for ingestion by default.

#### SigNoz UI
- A public domain is configured automatically in Railway to access the SigNoz dashboard.

#### Schema-Migration Order  
ClickHouse migrations run in the dedicated **`signoz-schema-migrator`** job. As the Railway does not yet offer Docker-style `depends_on`, dependent services can occasionally start before migrations finish and fail on their first boot.  
If that happens, **redeploy these services after the migrator job completes**, in the exact order shown:

1. **signoz-async-schema-migrator**  
2. **signoz** (main application)  
3. **signoz-otel-collector**

After redeploying in this sequence, all components will connect to ClickHouse with the correct schema and operate normally.

## Post-Deploy Checklist

After deploying the template, apply these changes for a stable setup.

### 1. Set JWT secret (required — security)

```bash
railway service signoz
railway variables set SIGNOZ_TOKENIZER_JWT_SECRET=$(openssl rand -hex 32)
```

Without this, sessions are signed with an empty string — anyone can forge valid tokens.

### 2. Rename deprecated env vars (recommended — removes warnings)

Railway dashboard → **signoz** → **Variables**:

| Old (template default) | New | Value |
|------------------------|-----|-------|
| `TELEMETRY_ENABLED` | `SIGNOZ_ANALYTICS_ENABLED` | `true` |
| `STORAGE` | `SIGNOZ_TELEMETRYSTORE_PROVIDER` | `clickhouse` |

Delete the old variables after adding the new ones.

### 3. Complete initial setup (recommended — stops OpAmp errors)

Open the SigNoz UI and create the first user/organization. Until this is done, the otel-collector logs repeated errors:

```
[ERRO] Failed to find or create agent error="cannot create agent without orgId"
```

### 4. Keep all SigNoz images in sync (required — prevents data loss)

All SigNoz Dockerfiles in this repo are pinned to specific versions. When upgrading, update all three in lockstep:

| Component | Location | Docker Hub |
|-----------|----------|------------|
| signoz | `signoz/Dockerfile.signoz` | [signoz/signoz](https://hub.docker.com/r/signoz/signoz/tags) |
| signoz-otel-collector | `signoz/Dockerfile.otel` | [signoz/signoz-otel-collector](https://hub.docker.com/r/signoz/signoz-otel-collector/tags) |
| signoz-schema-migrator | Railway dashboard (Docker image) | [signoz/signoz-schema-migrator](https://hub.docker.com/r/signoz/signoz-schema-migrator/tags) |

If you see errors like `No such column resource in table signoz_traces.distributed_signoz_index_v3` in the collector logs, update all images to matching versions and redeploy in dependency order (ClickHouse → migrators → signoz → otel-collector).

### 5. Fix schema migrator start commands (required — prevents crash)

The official SigNoz Railway template has a bug in the async migrator start command — it uses `tcp://[clickhouse]:9000` which is not valid Railway variable syntax. The `[clickhouse]` is passed as a literal string, causing a DSN parse error.

Update both migrator start commands on the Railway dashboard:

| Migrator | Start Command |
|----------|--------------|
| **sync** | `/bin/sh -c "sleep 120 && exec ./signoz-schema-migrator sync --dsn=tcp://${{clickhouse.RAILWAY_PRIVATE_DOMAIN}}:9000 --up="` |
| **async** | `./signoz-schema-migrator async --dsn=tcp://${{clickhouse.RAILWAY_PRIVATE_DOMAIN}}:9000 --up=` |

Also remove the `npm run migrate` pre-deploy command on both migrators — it's a leftover from the Railway template boilerplate and does nothing in a Go binary image.

## Why Deploy

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying SigNoz on Railway, you are one step closer to supporting a complete full-stack application with minimal burden. Host your servers, databases, AI agents, and more on Railway.
