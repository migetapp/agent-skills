# Metrics and logs

Where to look when an application is up but wrong. Metrics are
Prometheus-compatible and logs are Loki-compatible, so the query languages are the
ones you already know.

Read this before telling a customer you cannot see why something is failing.

Every app on Miget automatically gets metrics, logs, and pre-built Grafana dashboards — no setup. For **any** observability question (resource usage, request rates, restarts, errors, why a pod or cron run misbehaved), look here and in the Miget monitoring docs first: the REST API deliberately does **not** expose runtime metrics or app logs.

**Docs:** https://docs.miget.com/monitoring/overview · `/metrics` · `/metrics-api` · `/logs`

## In Grafana (UI)

Click **Monitoring** on an app's dashboard to open Grafana (automatic login — no credentials). Three pre-built dashboards ship with every app: **App Overview**, **Pod Details**, and **Logs**; custom dashboards can be built in Grafana.

## Metrics API (Prometheus-compatible)

Base URL `https://metrics.miget.com`. Auth: `Authorization: Bearer $MIGET_API_TOKEN` — the **same Miget API token** you already use for the REST API (see Authentication); there is no separate Grafana credential. Scope with the `X-Workspace-Id: <workspace-uuid>` header. Query language: **PromQL**. Subject to fair-use rate limits (`429` on throttle).

- Instant query: `GET /prometheus/api/v1/query?query=<PromQL>[&time=<ts>]`
- Range query: `GET /prometheus/api/v1/query_range?query=<PromQL>&start=<ts>&end=<ts>&step=<e.g. 60s>`
- Label discovery: `GET /prometheus/api/v1/labels` · `GET /prometheus/api/v1/label/{name}/values`

Metrics are prefixed `miget_` and carry common labels `namespace`, `app`, `addon`, `addon_type`, `instance` (HTTP metrics add `status`/`method`; disk adds `device`). Common series:
- **App:** `miget_app_replicas_desired`, `miget_app_replicas_available`, `miget_app_http_responses_total`, `miget_app_http_response_time_seconds_bucket`
- **Instance (pod):** `miget_instance_cpu_usage`, `miget_instance_memory_used_bytes`, `miget_instance_net_recv_bytes_total`, `miget_instance_disk_read_bytes_total`, `miget_instance_status_phase`, `miget_instance_restarts_total`
- **Volume:** `miget_volume_used_bytes`, `miget_volume_size_bytes`, `miget_volume_iops_limit`
- **Addon (PostgreSQL):** connections, database size, replication lag

Example — last hour of CPU for an app:
`GET https://metrics.miget.com/prometheus/api/v1/query_range?query=miget_instance_cpu_usage{app="my-app"}&start=<ts>&end=<ts>&step=60s`

## Logs API (Loki-compatible)

Same host and auth as the Metrics API. Query language: **LogQL**.
- `GET https://metrics.miget.com/loki/api/v1/query_range?query={app="my-app"}&limit=100`
- Narrow to a specific cron run by pod: `{app="my-app", pod="<last_job_name>"}`.

See https://docs.miget.com/monitoring/logs#logs-via-api.

## Retention (by plan)

| Plan | Metrics | Logs |
|---|---|---|
| Free | 30 days | 3 days |
| Pay as you grow | 13 months | 7 days |

Short-lived cron pods may fall between metric scrapes, so their `miget_instance_*` series can be sparse or absent; their **logs** are still queryable via the Logs API (and the cron `stream_logs` endpoint) while retained.

---
