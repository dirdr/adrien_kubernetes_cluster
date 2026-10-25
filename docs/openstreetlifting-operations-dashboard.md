# OpenStreetLifting operations dashboard

Use one Grafana dashboard to compare production (`osl`) and staging
(`osl-staging`), investigate failures, and check imports. Metrics come from
Prometheus, logs from Loki, and traces from Tempo.

## Collection and availability

The `prometheus-blackbox-exporter` Argo CD application deploys `osl-blackbox`
in `monitoring`. It checks six targets every 30 seconds, with an eight-second
probe timeout and a ten-second scrape timeout.

| Probe | Namespace | Scope | Success means |
|---|---|---|---|
| `production-website` | `osl` | `public` | Public HTTPS homepage returns 200 and HTML |
| `production-api` | `osl` | `public` | Public HTTPS `/health/ready` returns 200 and `status: ok` |
| `staging-website` | `osl-staging` | `internal` | Frontend Service returns 200 and HTML |
| `staging-api` | `osl-staging` | `internal` | Backend Service readiness returns 200 and `status: ok` |
| `staging-website-auth` | `osl-staging` | `auth-gate` | Public HTTPS frontend returns 401 with a Basic Auth challenge |
| `staging-api-auth` | `osl-staging` | `auth-gate` | Public HTTPS API returns 401 with a Basic Auth challenge |

The API readiness handler executes `SELECT 1` against PostgreSQL. Website
checks establish HTML delivery; they do not exercise every page or browser
interaction. Redirects fail these checks so an unexpected login or fallback
redirect cannot pass as application health. Public probes validate TLS.

Staging requires Basic Auth. Its public probes check the auth gate, while its
internal probes check the application behind it. A successful auth-gate check
does not establish that the application works. Authenticated public staging
checks would need separately provisioned credentials; the sealed htpasswd
hash cannot supply a probe password.

These probes run inside the cluster against public DNS or internal Services.
An independent monitor outside the cluster is needed to detect a total cluster
or monitoring outage and measure an external user's network path.

The PostgreSQL chart also deploys a PodMonitor for the CloudNativePG metrics
port, `9187`. Both environments receive it. Prometheus retains samples for up
to 35 days, capped at 16GB on its 20Gi volume. The size cap can shorten history.
A new installation needs time to accumulate a 30-day availability record.

## Grafana setup

Use the existing Prometheus, Loki, and Tempo data sources. Set the dashboard
range to one hour, refresh to 30 seconds, and the Prometheus scrape interval
to 30 seconds so `$__rate_interval` has enough samples.

| Variable | Type and data source | Configuration |
|---|---|---|
| `namespace` | Custom | `osl,osl-staging`; single value; default `osl` |
| `probe` | Query, Prometheus | Label values of `probe_success{namespace="$namespace",osl_probe="true"}`, label `probe`; allow multiple and All |
| `importer_pod` | Query result, Prometheus | Use the latest Pod expression below and regex `/pod="([^"]+)"/`; single value |
| `trace_id` | Text box | Empty default; paste a trace ID when investigating |

Refresh dependent variables on dashboard load and time-range changes.
`namespace` changes must refresh the dependent probe and importer selections.
Keep the default All interpolation for `probe`; queries use `${probe:regex}`.

| Row | Panels |
|---|---|
| Health | Current checks, 24-hour and 30-day availability, coverage, image version |
| API | Traffic, p50/p95/p99, 5xx count and percentage, slow routes, active requests |
| Resources | CPU and memory versus limits, replicas, restarts, OOM indicators |
| Database and cache | PostgreSQL health, pool usage, storage, database size, cache behavior |
| Importer | Current Job flags, successful duration, last success, run summary and logs |
| Investigation | Backend/frontend logs, warnings/errors, trace lookup |

Use instant queries for Stat, Gauge, and Table panels; use range queries for
Time series and State timeline panels. Preserve No data as Unknown. Do not
turn missing telemetry into a green status. Configure disconnected timeline
gaps instead of connecting null values.

## Health panels

### Current probe status

Stat and State timeline; legend `{{probe}} · {{scope}}`. Map 1 to Healthy and
0 to Failed. Repeat the Stat panel for each selected `probe`.

```promql
probe_success{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"}
```

### Probe scraper status

Stat or State timeline; legend `{{probe}}`. This detects exporter/scrape
failure, separately from a reachable exporter reporting a failed HTTP probe.
Target relabeling gives `up` the same application namespace and probe labels.

```promql
up{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"}
```

### Availability over recorded checks

Stat; percent 0–100; legend `{{probe}}`. Each expression measures its named
window regardless of the dashboard time range. Keep coverage beside these
panels: averages exclude missing samples and are not a guarantee of uptime
during a monitoring outage.

```promql
100 * avg_over_time(probe_success{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"}[24h])
```

```promql
100 * avg_over_time(probe_success{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"}[30d])
```

### Monitoring coverage

Stat; percent 0–100. Expected sample counts assume the configured 30-second
interval. These approximate coverage panels reveal short history, missed
scrapes, and retention truncation; tiny boundary differences are normal.

```promql
clamp_max(100 * count_over_time(probe_success{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"}[24h]) / 2880, 100)
```

```promql
clamp_max(100 * count_over_time(probe_success{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"}[30d]) / 86400, 100)
```

### HTTP response time and status code

Time series; seconds for duration, number for status code.

```promql
probe_duration_seconds{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"}
```

```promql
probe_http_status_code{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"}
```

### TLS certificate expiry

Stat; days. Internal HTTP targets have no TLS series. Start with a warning
below 14 days and a critical threshold below seven days.

```promql
(probe_ssl_earliest_cert_expiry{namespace="$namespace",osl_probe="true",probe=~"${probe:regex}"} - time()) / 86400
```

### API metrics scraping

Stat; 1 means Prometheus can scrape the API, not that public traffic succeeds.

```promql
min(up{namespace="$namespace",service="openstreetlifting-backend"})
```

### Deployed images

Table. Apply Labels to fields, keep `container` and `image`, and hide Value.

```promql
max by (container, image) (kube_pod_container_info{namespace="$namespace",container=~"backend|frontend"})
```

## API panels

These metrics measure the Rust API, including SSR calls from the frontend.
They do not measure browser rendering or Core Web Vitals. Latency ends when
response headers are ready, including middleware and database work. Health
checks are excluded so probes do not dominate quiet staging traffic.

### Requests per second

Time series; requests per second.

```promql
sum(rate(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval]))
```

### Requests during the selected period

Stat; instant query. Counter increases are estimates and can be fractional.

```promql
sum(increase(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*"}[$__range]))
```

### Latency percentiles

One Time series panel with three queries; legends p50, p95, and p99; unit
seconds. Percentiles are histogram estimates. Low request volume makes p99
noisy; inspect counts and widen the time range before drawing conclusions.

```promql
histogram_quantile(0.50, sum by (le) (rate(http_server_request_duration_seconds_bucket{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval])))
```

```promql
histogram_quantile(0.95, sum by (le) (rate(http_server_request_duration_seconds_bucket{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval])))
```

```promql
histogram_quantile(0.99, sum by (le) (rate(http_server_request_duration_seconds_bucket{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval])))
```

### Slowest routes

Table or Time series; seconds; legend `{{http_route}}`.

```promql
topk(5, histogram_quantile(0.95, sum by (le, http_route) (rate(http_server_request_duration_seconds_bucket{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval]))))
```

### Traffic by route and status

Two Time series panels; requests per second. Use the grouping label as legend.

```promql
sum by (http_route) (rate(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval]))
```

```promql
sum by (http_response_status_code) (rate(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval]))
```

### Server error count

Stat; instant query; selected dashboard period. The fallback returns zero
when requests exist but no 5xx series has been created; absent request data
stays absent.

```promql
sum(increase(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*",http_response_status_code=~"5.."}[$__range]))
or
(0 * sum(increase(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*"}[$__range])))
```

### Server error percentage

Time series; percent 0–100. With no traffic the ratio is undefined. Do not
interpret that as an outage or force it to a healthy value.

```promql
100 *
(
  sum(rate(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*",http_response_status_code=~"5.."}[$__rate_interval]))
  or (0 * sum(rate(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval])))
)
/
sum(rate(http_server_requests_total{namespace="$namespace",http_route!~"/health/.*"}[$__rate_interval]))
```

### Active requests

Time series; requests awaiting response headers.

```promql
sum(http_server_active_requests{namespace="$namespace",http_route!~"/health/.*"})
```

## Resource panels

Kube-prometheus-stack supplies kubelet/cAdvisor and kube-state-metrics.
Resource legends use `{{pod}} · {{container}}`. Kubernetes replica panels use
`{{deployment}}`. Rollouts can briefly show both old and new Pods.

### CPU cores and percentage of limit

Two Time series panels; cores and percent 0–100 respectively.

```promql
sum by (pod, container) (rate(container_cpu_usage_seconds_total{namespace="$namespace",job="kubelet",metrics_path="/metrics/cadvisor",container=~"backend|frontend"}[$__rate_interval]))
```

```promql
100 *
sum by (pod, container) (rate(container_cpu_usage_seconds_total{namespace="$namespace",job="kubelet",metrics_path="/metrics/cadvisor",container=~"backend|frontend"}[$__rate_interval]))
/
max by (pod, container) (kube_pod_container_resource_limits{namespace="$namespace",container=~"backend|frontend",resource="cpu",unit="core"})
```

### Memory bytes and percentage of limit

Two Time series panels; bytes and percent 0–100 respectively.

```promql
max by (pod, container) (container_memory_working_set_bytes{namespace="$namespace",job="kubelet",metrics_path="/metrics/cadvisor",container=~"backend|frontend"})
```

```promql
100 *
max by (pod, container) (container_memory_working_set_bytes{namespace="$namespace",job="kubelet",metrics_path="/metrics/cadvisor",container=~"backend|frontend"})
/
max by (pod, container) (kube_pod_container_resource_limits{namespace="$namespace",container=~"backend|frontend",resource="memory",unit="byte"})
```

### Available and desired replicas

One Time series panel with both queries. Available should reach desired.

```promql
max by (deployment) (kube_deployment_status_replicas_available{namespace="$namespace",deployment=~"openstreetlifting-(backend|frontend)"})
```

```promql
max by (deployment) (kube_deployment_spec_replicas{namespace="$namespace",deployment=~"openstreetlifting-(backend|frontend)"})
```

### Container restarts

Table or Stat; selected dashboard period.

```promql
sum by (pod, container) (increase(kube_pod_container_status_restarts_total{namespace="$namespace",container=~"backend|frontend"}[$__range]))
```

### Last termination was an OOM kill

Table; instant query. This is a last-termination indicator, not an event count.
An empty result means no matching indicator; check scrape health separately.

```promql
max by (pod, container) (kube_pod_container_status_last_terminated_reason{namespace="$namespace",container=~"backend|frontend",reason="OOMKilled"}) == 1
```

## Database and cache panels

### PostgreSQL health and metric collection

Stat panels; PostgreSQL health uses CloudNativePG's connection check. Legends
use `{{pod}}`. Scraper health and collector errors help diagnose missing data.

```promql
cnpg_collector_up{namespace="$namespace",osl_postgres="true"}
```

```promql
up{namespace="$namespace",osl_postgres="true"}
```

The collection-error metric name depends on the installed operator version.
Both names below express 1 for error; a missing series stays Unknown.

```promql
cnpg_collector_last_collection_error{namespace="$namespace",osl_postgres="true"}
or
cnpg_last_error{namespace="$namespace",osl_postgres="true"}
```

### Database size

Time series; bytes; legend `{{datname}}`. Default CloudNativePG SQL metrics
must be enabled in the operator. Taking the maximum avoids counting replica
copies as extra logical database size.

```promql
max by (datname) (cnpg_pg_database_size_bytes{namespace="$namespace",osl_postgres="true",datname="appdb"})
```

### API connection pool

Used and idle connections; Time series; legend `{{state}}`. These are backend
pool connections, not the total across all PostgreSQL clients.

```promql
sum by (state) (db_client_connections{namespace="$namespace"})
```

Maximum connections; add as a line to the same panel.

```promql
sum(db_client_connections_max{namespace="$namespace"})
```

Pool saturation per backend Pod; percent 0–100.

```promql
100 * sum by (pod) (db_client_connections{namespace="$namespace",state="used"})
/
sum by (pod) (db_client_connections_max{namespace="$namespace"})
```

### Storage usage and free space

Two panels; percent 0–100 and bytes; legend `{{persistentvolumeclaim}}`.
Volume statistics require storage-driver support. Repeat these panels with
`namespace="monitoring"` for Prometheus, Loki, and Tempo storage.

```promql
100 * max by (persistentvolumeclaim) (kubelet_volume_stats_used_bytes{namespace="$namespace"})
/
max by (persistentvolumeclaim) (kubelet_volume_stats_capacity_bytes{namespace="$namespace"})
```

```promql
min by (persistentvolumeclaim) (kubelet_volume_stats_available_bytes{namespace="$namespace"})
```

### Cache traffic and hit percentage

Time series; requests per second; legend `{{cache}} · {{result}}`.

```promql
sum by (cache, result) (rate(osl_cache_requests_total{namespace="$namespace"}[$__rate_interval]))
```

Hit ratio; percent 0–100; excludes bypasses. No cache activity leaves the
ratio undefined. Cache metrics appear only after activity creates the series.

```promql
100 *
(
  sum by (cache) (rate(osl_cache_requests_total{namespace="$namespace",result="hit"}[$__rate_interval]))
  or (0 * sum by (cache) (rate(osl_cache_requests_total{namespace="$namespace",result=~"hit|miss"}[$__rate_interval])))
)
/
sum by (cache) (rate(osl_cache_requests_total{namespace="$namespace",result=~"hit|miss"}[$__rate_interval]))
```

### Cache refresh results and average duration

Table; refresh counts over the selected period, grouped by success/error.

```promql
sum by (cache, result) (increase(osl_cache_refreshes_total{namespace="$namespace"}[$__range]))
```

Time series; seconds. No refreshes leave the average undefined.

```promql
sum by (cache) (rate(osl_cache_refresh_duration_seconds_sum{namespace="$namespace"}[$__rate_interval]))
/
sum by (cache) (rate(osl_cache_refresh_duration_seconds_count{namespace="$namespace"}[$__rate_interval]))
```

## Importer panels

The deployment competition Job is `openstreetlifting-import`. Instagram sync
is a separate Job, `openstreetlifting-instagram`. The competition Job name is
reused on each deployment; current status describes the currently retained
Job. Kubernetes deletes completed Jobs after 24 hours. Historical samples
and Loki logs remain available only within their respective retention windows.

### Active, succeeded, and terminally failed

Three Stat panels; instant queries. Active is a Pod count; succeeded and
failed are condition flags. Zero for one flag does not imply another flag is
true. Pending is possible, and No data means Unknown or an expired Job.

```promql
max(kube_job_status_active{namespace="$namespace",job_name="openstreetlifting-import"})
```

```promql
max(kube_job_complete{namespace="$namespace",job_name="openstreetlifting-import",condition="true"})
```

```promql
max(kube_job_failed{namespace="$namespace",job_name="openstreetlifting-import",condition="true"})
```

Use the terminal failure condition rather than failed Pod counts: a retried
Pod can fail while the Job eventually succeeds.

### Successful Job duration

Stat; seconds. Completion time exists for successful Jobs. This expression
uses the currently retained Job and includes time spent retrying.

```promql
max by (namespace, job_name) (kube_job_status_completion_time{namespace="$namespace",job_name="openstreetlifting-import"})
-
max by (namespace, job_name) (kube_job_status_start_time{namespace="$namespace",job_name="openstreetlifting-import"})
```

### Last observed success and its age

First panel: Date & time, with the timestamp converted to milliseconds.
Second panel: seconds/duration. The 30-day lookup requires retained samples.
No expected deployment means no expected new import: age alone is not failure.

```promql
1000 * max(max_over_time(kube_job_status_completion_time{namespace="$namespace",job_name="openstreetlifting-import"}[30d]))
```

```promql
time() - max(max_over_time(kube_job_status_completion_time{namespace="$namespace",job_name="openstreetlifting-import"}[30d]))
```

### Latest importer Pod variable

Use this as the Query result expression for `importer_pod`. It finds the
newest observed deployment competition-import Pod within seven days, even
after Kubernetes deletes it. It selects the newest attempt if a Job retries;
use all importer logs to inspect earlier attempts. It excludes Instagram
and manually created `openstreetlifting-importer-*` Jobs.

```promql
topk(1, max by (pod) (last_over_time(kube_pod_start_time{namespace="$namespace",pod=~"openstreetlifting-import-.+"}[7d])))
```

Apply regex `/pod="([^"]+)"/`. A Job too short to be observed by a scrape may
not appear here. In that case choose its Pod from Loki's label browser.

## Log panels

Use Loki and the Logs visualization. Preserve the original JSON lines so the
existing TraceID derived field can link into Tempo. Expand a line to inspect
its structured attributes. Application logs carry `severity`, `service_name`,
and `environment` labels; the container selectors also include non-JSON
startup and crash messages.

### Backend

```logql
{namespace="$namespace",container="backend"}
```

### Frontend

```logql
{namespace="$namespace",container="frontend"}
```

### Selected importer attempt

Set this panel's relative time to seven days so the last run is visible even
when the dashboard shows one hour. Loki must still retain its log entries.

```logql
{namespace="$namespace",container="importer",pod="$importer_pod"}
```

### All importer runs

```logql
{namespace="$namespace",container="importer"}
```

### Import completion summary

Logs panel; relative time seven days. The current successful completion event
contains a competition count. It does not emit a total participant count.
Do not substitute a count of log lines for participants.

```logql
{namespace="$namespace",container="importer",pod="$importer_pod"}
| json
| message="Competition import completed"
| line_format "{{.timestamp}} — {{.attributes_competitions}} competitions imported"
```

### Warnings and errors

```logql
{namespace="$namespace",container=~"backend|frontend|importer",severity=~"WARN|ERROR|FATAL"}
```

### Error log volume

Time series; log entries per interval; legend `{{container}}`. Missing error
streams can produce No data; interpret this alongside service and log health.

```logql
sum by (container) (
  count_over_time({namespace="$namespace",container=~"backend|frontend|importer",severity=~"ERROR|FATAL"}[$__interval])
)
```

### Trace lookup

An empty `trace_id` matches all lines. Paste an ID to filter, then open its
View trace link. Trace sampling and Tempo retention determine whether the
referenced trace is still available.

```logql
{namespace="$namespace",container=~"backend|frontend|importer"} |= "$trace_id"
```

## Rollout and verification

Merge and sync through Argo CD after review. The new Application lives in
`apps/`; confirm the live root application watches that directory in this
repository. The checked-in `bootstrap/root-app.yaml` uses a legacy repository
URL, so inspect the live root source before relying on automatic discovery.

Sync `kube-prometheus-stack` for retention, both PostgreSQL applications for
their PodMonitors, and `prometheus-blackbox-exporter` for the HTTP probes.
Prometheus Operator CRDs must exist before the monitors are installed.

```sh
kubectl get applications -n argocd
kubectl get servicemonitor -n monitoring -l release=kube-prometheus-stack
kubectl get podmonitor -n osl
kubectl get podmonitor -n osl-staging
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus-blackbox-exporter
```

In Grafana Explore, the first query should return six series, normally 1.
The second should return one PostgreSQL instance in each namespace with the
current single-instance topology. Allow a few scrape intervals after sync.

```promql
probe_success{osl_probe="true"}
```

```promql
cnpg_collector_up{namespace=~"osl|osl-staging",osl_postgres="true"}
```

For a failed probe, inspect exporter logs and its HTTP status/duration metrics.
Use a local port-forward for the exporter's debug endpoint:

```sh
kubectl port-forward -n monitoring svc/osl-blackbox 9115:9115
```

```sh
curl --get 'http://127.0.0.1:9115/probe' \
  --data-urlencode 'module=osl_https_ready' \
  --data-urlencode 'target=https://api.openstreetlifting.org/health/ready' \
  --data-urlencode 'debug=true'
```

If cAdvisor panels are empty, inspect the actual `job` and `metrics_path`
labels before adjusting selectors. If PostgreSQL metrics are empty, check
PodMonitor discovery, port `metrics`, and the operator's default SQL metrics.
PVC panels require storage-driver support.

## Alerts and remaining coverage

Dashboard panels do not create alerts. Configure alert rules and a tested
contact point separately. Start with these conditions and tune them against
normal production traffic:

| Condition | Initial policy |
|---|---|
| Production HTTP probe fails | Notify after two minutes |
| Probe scraping fails or expected target disappears | Notify after two minutes; missing is not healthy |
| Sustained production 5xx errors | Combine an error ratio with a minimum request/error count |
| Import Job terminally fails | Notify while the Job still exists |
| PostgreSQL unavailable | Notify after one minute |
| PVC nearly full | Warn above 80%; critical above 90% |
| TLS certificate nearing expiry | Warn below 14 days; critical below seven days |

Keep staging notifications separate from production. Attach image versions,
deployment times, Pod names, and trace links to investigations.

Backups and restore verification are not configured by this change. Backup
age panels require an actual backup schedule and its exporter metrics.
Durable import attempt history and participant totals require additional
application summary metrics or persisted run records. Browser performance
and independent external uptime checks also remain separate collection work.

## References

- [Blackbox Exporter configuration](https://github.com/prometheus/blackbox_exporter/blob/master/CONFIGURATION.md)
- [Blackbox Helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-blackbox-exporter)
- [CloudNativePG monitoring](https://cloudnative-pg.io/docs/1.28/monitoring/)
- [Prometheus query functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Kubernetes Job metrics](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/job-metrics.md)
- [Loki log queries](https://grafana.com/docs/loki/latest/query/log_queries/)
