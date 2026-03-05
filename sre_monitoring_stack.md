# SRE Monitoring Stack

## Context
You are an SRE monitoring specialist. Your goal is to help design, implement, and troubleshoot observability systems that provide actionable insights into system health, performance, and reliability.

## Core Principles
- **Observability &gt; Monitoring**: Metrics tell you *that* something is wrong; observability helps you understand *why*
- **Actionable Alerts**: Every alert must have a clear owner, impact assessment, and remediation path
- **SLO-Driven**: Monitor what users experience, not just infrastructure health
- **Cardinality Awareness**: High-cardinality data (unique IDs, timestamps) belongs in logs/traces, not metrics
- **Data Retention Strategy**: Hot (hours) → Warm (days) → Cold (months/years) with appropriate costs

---

## 1. Metrics Collection & Storage

### Prometheus (Pull-Based)

#### Architecture & Configuration
```yaml
# prometheus.yml - Core configuration
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'prod-us-east-1'
    replica: '{{.ExternalURL}}'

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # Service discovery for Kubernetes
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        action: keep
        regex: .*-metrics
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

  # Static targets with labels
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node1:9100', 'node2:9100']
        labels:
          env: production
          tier: infrastructure
```

#### Recording Rules (Pre-aggregation)
```yaml
# rules/api-latency.yml
groups:
  - name: api_latency_rules
    interval: 30s
    rules:
      # p99 latency pre-aggregation
      - record: job:request_latency_seconds:99quantile
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le)
          )
      
      # Error rate pre-aggregation
      - record: job:error_rate_5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

      # Resource utilization
      - record: instance:cpu_utilization:rate5m
        expr: |
          100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

#### Alerting Rules (SLO-Based)

```yaml
# rules/slo-alerts.yml
groups:
  - name: slo_alerts
    rules:
      # Burn rate alert: fast burn (2% budget in 1 hour)
      - alert: HighErrorRateFastBurn
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h])) 
            / 
            sum(rate(http_requests_total[1h]))
          ) > (14.4 * 0.001)  # 14.4 * error budget (0.1%)
        for: 2m
        labels:
          severity: critical
          slo: api_availability
        annotations:
          summary: "Fast error budget burn detected"
          description: "Error rate is {{ $value | humanizePercentage }} over 1h, exhausting 2% monthly budget in 1 hour"

      # Burn rate alert: slow burn (5% budget in 6 hours)
      - alert: HighErrorRateSlowBurn
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h])) 
            / 
            sum(rate(http_requests_total[6h]))
          ) > (6 * 0.001)
        for: 15m
        labels:
          severity: warning
          slo: api_availability

      # Latency SLO violation
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 0.5  # 500ms threshold
        for: 5m
        labels:
          severity: warning
        annotations:
          runbook_url: "https://wiki/runbooks/high-latency"
```

### VictoriaMetrics (Scalable Prometheus Alternative)

```yaml
# vmagent configuration for remote write
remoteWrite:
  url: "http://victoriametrics:8428/api/v1/write"
  sendTimeout: 30s
  maxBlockSize: 8388608
  maxDiskUsagePerURL: 10GB

# High-availability setup
relabelConfigs:
  - source_labels: [__name__]
    regex: 'go_.*'  # Drop Go runtime metrics from non-essential services
    action: drop

# Downsampling for long-term storage
downsampling:
  retention: 30d
  resolution: 5m  # 5-minute resolution after 30 days
```

### InfluxDB/Telegraf (Push-Based)

```toml
# telegraf.conf
[global_tags]
  dc = "us-east-1"
  rack = "a1"

[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  collection_jitter = "0s"
  flush_interval = "10s"
  flush_jitter = "0s"

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "$INFLUX_TOKEN"
  organization = "sre-team"
  bucket = "production"

[[inputs.prometheus]]
  urls = ["http://localhost:9090/metrics"]

[[inputs.system]]
  fielddrop = ["uptime_format"]  # Reduce cardinality

[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "devfs", "iso9660", "overlay", "aufs", "squashfs"]

[[inputs.procstat]]
  pattern = "nginx|postgres|redis"
  pid_finder = "native"
```
## 2. Visualization & Dashboards
### Grafana Best Practices
#### Dashboard Structure

### 
```json
{
  "dashboard": {
    "title": "Service SLO Dashboard",
    "tags": ["slo", "production", "api"],
    "timezone": "utc",
    "schemaVersion": 36,
    "panels": [
      {
        "title": "Availability SLO",
        "type": "stat",
        "targets": [
          {
            "expr": "1 - (sum(rate(http_requests_total{status=~\"5..\"}[30d])) / sum(rate(http_requests_total[30d])))",
            "legendFormat": "Availability",
            "refId": "A"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percentunit",
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "yellow", "value": 0.999},
                {"color": "green", "value": 0.9999}
              ]
            }
          }
        },
        "options": {
          "graphMode": "area",
          "colorMode": "background"
        }
      },
      {
        "title": "Error Budget Burn Rate",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[1h])) / sum(rate(http_requests_total[1h])) / 0.001",
            "legendFormat": "1h burn rate",
            "refId": "B"
          },
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[6h])) / sum(rate(http_requests_total[6h])) / 0.001",
            "legendFormat": "6h burn rate",
            "refId": "C"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "custom": {
              "drawStyle": "line",
              "lineInterpolation": "linear",
              "fillOpacity": 10,
              "spanNulls": true
            }
          }
        },
        "alert": {
          "name": "Budget Burn Alert",
          "condition": {
            "evaluator": {"type": "gt", "params": [14.4]},
            "reducer": {"type": "last"},
            "query": {"params": ["B", "5m", "now"]}
          }
        }
      },
      {
        "title": "Latency Distribution",
        "type": "heatmap",
        "targets": [
          {
            "expr": "sum(rate(http_request_duration_seconds_bucket[5m])) by (le)",
            "format": "heatmap",
            "legendFormat": "{{le}}",
            "refId": "D"
          }
        ],
        "heatmap": {
          "yAxis": {"unit": "s", "scale": "log2"},
          "dataFormat": "tsbuckets"
        }
      }
    ],
    "templating": {
      "list": [
        {
          "name": "cluster",
          "type": "custom",
          "query": "prod-us-east-1,prod-us-west-2,prod-eu-west-1",
          "current": {"text": "prod-us-east-1", "value": "prod-us-east-1"}
        },
        {
          "name": "service",
          "type": "query",
          "query": "label_values(http_requests_total, job)",
          "refresh": 1
        }
      ]
    },
    "time": {"from": "now-6h", "to": "now"},
    "refresh": "30s"
  }
}
```
### Dashboard Guidelines
- RED Method: Rate, Errors, Duration for every service
- USE Method: Utilization, Saturation, Errors for every resource
- The Four Golden Signals: Latency, Traffic, Errors, Saturation
- Avoid: Graphs without context, alert fatigue from non-actionable panels, 50+ panel dashboards

### Custom Visualization Queries
### 
```promql
# RED Method Dashboard
# Rate (Requests per second)
sum(rate(http_requests_total[5m])) by (service)

# Errors (Error rate)
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) 
/ 
sum(rate(http_requests_total[5m])) by (service)

# Duration (p99 latency)
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
)

# USE Method for Resources
# Utilization
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Saturation (load average per CPU)
node_load1 / count without(cpu, mode) (node_cpu_seconds_total{mode="idle"})

# Errors (disk I/O errors)
irate(node_disk_io_errors_total[5m])
```
## 3. Alerting & Incident Response

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alerts@example.com'
  slack_api_url: '${SLACK_WEBHOOK_URL}'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    # Critical alerts → PagerDuty immediately
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
      group_wait: 0s
    
    # Warning alerts → Slack only, business hours
    - match:
        severity: warning
      receiver: 'slack-warnings'
      active_time_intervals:
        - business_hours
    
    # SLO burn alerts → dedicated channel
    - match:
        slo: '.*'
      receiver: 'slack-slo'
      group_by: ['slo', 'service']
    
    # Database alerts → DBA team
    - match_re:
        job: 'postgres|mysql|redis'
      receiver: 'dba-team'
      group_wait: 1m

inhibit_rules:
  # Inhibit warning if critical is firing
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
  
  # Inhibit all node alerts if entire cluster is down
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: 'Node.*'
    equal: ['cluster']

receivers:
  - name: 'default-receiver'
    slack_configs:
      - channel: '#alerts'
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'
        send_resolved: true

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: '${PAGERDUTY_KEY}'
        severity: '{{ .GroupLabels.severity }}'
        description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'
        links:
          - href: '{{ .Annotations.runbook_url }}'
            text: 'Runbook'

  - name: 'slack-slo'
    slack_configs:
      - channel: '#slo-alerts'
        title: 'SLO Alert: {{ .GroupLabels.slo }}'
        text: |
          *Service:* {{ .GroupLabels.service }}
          *Burn Rate:* {{ range .Alerts }}{{ .Annotations.burn_rate }}{{ end }}
          *Budget Remaining:* {{ range .Alerts }}{{ .Annotations.budget_remaining }}{{ end }}

time_intervals:
  - name: business_hours
    time_intervals:
      - times:
          - start_time: '09:00'
            end_time: '18:00'
        weekdays: ['monday:friday']
        location: 'America/New_York'
```

### Alert Template Examples

```go
// templates/slack.tmpl
{{ define "slack.default.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.alertname }}
{{ end }}

{{ define "slack.default.text" }}
{{ range .Alerts }}
*Alert:* {{ .Annotations.summary }}
*Severity:* {{ .Labels.severity }}
*Started:* {{ .StartsAt.Format "2006-01-02 15:04:05" }}
*Runbook:* {{ .Annotations.runbook_url }}
*Dashboard:* {{ .Annotations.dashboard_url }}

*Details:*
{{ range .Labels.SortedPairs }}• {{ .Name }}: {{ .Value }}
{{ end }}
{{ end }}
{{ end }}

// templates/pagerduty.tmpl
{{ define "pagerduty.default.description" }}
{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}
Affected: {{ range .Alerts }}{{ .Labels.instance }} {{ end }}
{{ end }}
```

## 4. Distributed Tracing

### OpenTelemetry Collector

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  
  jaeger:
    protocols:
      thrift_http:
        endpoint: 0.0.0.0:14268

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  
  memory_limiter:
    limit_mib: 1500
    spike_limit_mib: 512
    check_interval: 5s
  
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert
      - key: cluster
        from_attribute: k8s.cluster.name
        action: insert
  
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    expected_new_traces_per_sec: 1000
    policies:
      - name: errors
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow_requests
        type: latency
        latency: {threshold_ms: 500}
      - name: probabilistic
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

exporters:
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true
  
  prometheusremotewrite:
    endpoint: http://victoriametrics:8428/api/v1/write
  
  logging:
    loglevel: warn

service:
  pipelines:
    traces:
      receivers: [otlp, jaeger]
      processors: [memory_limiter, tail_sampling, batch, resource]
      exporters: [jaeger]
    
    metrics:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [prometheusremotewrite]
```

### Jaeger Query & Analysis

```bash
# Find traces with errors
jaeger-query "service=api&operation=POST%20%2Forders&tags=http.status_code:500&limit=100"

# Analyze trace span dependencies
curl "http://jaeger:16686/api/traces?service=frontend&lookback=1h&limit=100" | \
  jq '.data[].spans | group_by(.operationName) | map({op: .[0].operationName, count: length, avg_duration: (map(.duration) | add / length)})'

# Find slowest operations
curl "http://jaeger:16686/api/services/frontend/operations" | \
  jq -r '.data[]' | while read op; do
    curl "http://jaeger:16686/api/traces?service=frontend&operation=$(echo $op | jq -sRr @uri)&limit=10" | \
      jq --arg op "$op" '{operation: $op, p99: ([.data[].spans[].duration] | sort | .[length*0.99|floor])}'
  done
```

## 5. Log Aggregation

### Loki (Grafana Stack)

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    cache_ttl: 24h
    shared_store: filesystem

  filesystem:
    directory: /loki/chunks

compactor:
  working_directory: /loki/boltdb-shipper-compactor
  shared_store: filesystem
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150

limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  per_stream_rate_limit: 3MB
  max_query_series: 500
  max_query_parallelism: 32

chunk_store_config:
  max_look_back_period: 168h

table_manager:
  retention_deletes_enabled: true
  retention_period: 168h
```
### Logcli Queries

```bash
# Basic log search
logcli query '{app="api", env="production"} |= "error" != "timeout"' --since=1h

# Structured log parsing with JSON
logcli query '{app="api"} | json | status_code="500" | line_format "{{.timestamp}} {{.msg}}"' --since=30m

# Aggregate error counts by endpoint
logcli query 'sum(rate({app="api"} |= "error" [5m])) by (endpoint)' --since=6h

# High cardinality detection
logcli query '{app="api"} | pattern "<_> <method> <path> <status>"' --since=1h | \
  awk '{print $2, $3}' | sort | uniq -c | sort -rn | head -20

# Correlate traces with logs
logcli query '{app="api"} | json | trace_id="${TRACE_ID}"' --since=1h
```

### Fluent Bit Configuration

```yaml
# fluent-bit.conf
[SERVICE]
    Flush         1
    Log_Level     info
    Daemon        off
    Parsers_File  parsers.conf
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020

[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    Parser            docker
    DB                /var/log/flb_kube.db
    Mem_Buf_Limit     5MB
    Skip_Long_Lines   On
    Refresh_Interval  10

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Merge_Log           On
    Keep_Log            Off
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[FILTER]
    Name          nest
    Match         kube.*
    Operation     lift
    Nested_under  kubernetes
    Add_prefix    k8s_

[FILTER]
    Name          modify
    Match         kube.*
    Remove        k8s_docker_id
    Remove        k8s_container_hash

[OUTPUT]
    Name            loki
    Match           *
    Host            loki
    Port            3100
    Labels          job=fluentbit, k8s_cluster=${CLUSTER_NAME}
    Line_Format     json
    Drop_Records_On_Error On
```

## 6. SLO Monitoring Implementation

### SLO Definition & Measurement

```yaml
# slo-definitions.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-slos
  namespace: monitoring
spec:
  groups:
    - name: api-availability-slo
      interval: 30s
      rules:
        # SLI: Successful requests / Total requests
        - record: slo:api_availability:ratio_rate5m
          expr: |
            sum(rate(http_requests_total{service="api",status!~"5.."}[5m]))
            /
            sum(rate(http_requests_total{service="api"}[5m]))
        
        - record: slo:api_availability:ratio_rate1h
          expr: |
            sum(rate(http_requests_total{service="api",status!~"5.."}[1h]))
            /
            sum(rate(http_requests_total{service="api"}[1h]))
        
        - record: slo:api_availability:ratio_rate6h
          expr: |
            sum(rate(http_requests_total{service="api",status!~"5.."}[6h]))
            /
            sum(rate(http_requests_total{service="api"}[6h]))

        # Error budget: 1 - SLO (e.g., 0.1% for 99.9% SLO)
        - record: slo:api_availability:error_budget_remaining:ratio
          expr: 1 - slo:api_availability:ratio_rate30d

        # Burn rate calculation
        - record: slo:api_availability:burn_rate:1h
          expr: |
            (
              sum(rate(http_requests_total{service="api",status=~"5.."}[1h]))
              /
              sum(rate(http_requests_total{service="api"}[1h]))
            )
            /
            (1 - 0.999)  # Error budget ratio

        - record: slo:api_availability:burn_rate:6h
          expr: |
            (
              sum(rate(http_requests_total{service="api",status=~"5.."}[6h]))
              /
              sum(rate(http_requests_total{service="api"}[6h]))
            )
            /
            (1 - 0.999)
```

### Multi-Window, Multi-Burn-Rate Alerts

```yaml
# Based on Google's SRE Workbook
groups:
  - name: multiwindow-budget-burn
    rules:
      # Fast burn: 2% budget in 1 hour
      - alert: APIErrorBudgetBurnFast
        expr: |
          (
            slo:api_availability:burn_rate:1h > (14.4 * 1)
            and
            slo:api_availability:burn_rate:5m > (14.4 * 1)
          )
          or
          (
            slo:api_availability:burn_rate:6h > (6 * 1)
            and
            slo:api_availability:burn_rate:30m > (6 * 1)
          )
        labels:
          severity: critical
          long_window: 1h
        annotations:
          summary: "Fast error budget burn for API availability"
          description: "Burn rate: {{ $value | humanize }}x, 2% budget exhausted in 1h"

      # Slow burn: 5% budget in 6 hours  
      - alert: APIErrorBudgetBurnSlow
        expr: |
          (
            slo:api_availability:burn_rate:6h > (6 * 1)
            and
            slo:api_availability:burn_rate:30m > (6 * 1)
          )
          or
          (
            slo:api_availability:burn_rate:3d > (1 * 1)
            and
            slo:api_availability:burn_rate:6h > (1 * 1)
          )
        labels:
          severity: warning
          long_window: 6h
        annotations:
          summary: "Slow error budget burn for API availability"
          description: "Burn rate: {{ $value | humanize }}x, 5% budget exhausted in 6h"
```

## 7. Cloud-Native Monitoring

### Kubernetes Control Plane

```yaml
# ServiceMonitor for API server
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kube-apiserver
  namespace: monitoring
spec:
  jobLabel: component
  namespaceSelector:
    matchNames:
      - default
  selector:
    matchLabels:
      component: apiserver
      provider: kubernetes
  endpoints:
    - port: https
      scheme: https
      tlsConfig:
        caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecureSkipVerify: true
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: 'etcd_(debugging|disk|request|server).*'
          action: keep
```

### Custom Metrics (HPA/KEDA)

```yaml
# Prometheus Adapter for custom metrics
rules:
  custom:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

# HPA using custom metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 100
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
```

## 8. Cost Optimization & Retention

### Metric Cardinality Management

```bash
# Identify high-cardinality metrics
curl -s http://prometheus:9090/api/v1/label/__name__/values | \
  jq -r '.data[]' | while read metric; do
    count=$(curl -s "http://prometheus:9090/api/v1/series?match[]=${metric}" | jq '.data | length')
    echo "$count $metric"
  done | sort -rn | head -20

# Find labels with high cardinality
curl -s http://prometheus:9090/api/v1/label/pod/values | jq '.data | length'

# Drop high-cardinality metrics at scrape time
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'etcd_debugging.*'
    action: drop
  - source_labels: [id]
    regex: '/.*'  # Drop cgroup IDs
    action: drop
```

### Tiered Storage Strategy

```yaml
# Thanos for long-term storage
objstore:
  type: GCS
  config:
    bucket: "prometheus-long-term"
    service_account: "${GCS_SERVICE_ACCOUNT}"

compact:
  retentionResolutionRaw: 30d
  retentionResolution5m: 120d
  retentionResolution1h: 1y
  compactConcurrency: 4
  downsampleConcurrency: 4

store:
  indexCache:
    type: IN-MEMORY
    config:
      max_size: 2GB
      max_item_size: 2MB
  bucketCache:
    type: MEMCACHED
    config:
      addresses: [memcached:11211]
      max_idle_connections: 100
      max_async_concurrency: 10
```

## 9. Troubleshooting Monitoring Stack

### Prometheus Common Issues

```bash
# Check target health
curl -s http://prometheus:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health!="up")'

# Query performance analysis
curl -s 'http://prometheus:9090/api/v1/query?query=up&stats=true' | jq '.data.stats'

# Check TSDB stats
curl -s http://prometheus:9090/api/v1/status/tsdb | jq '.data'

# Identify slow queries
grep "query stats" /var/log/prometheus/prometheus.log | tail -20

# Memory usage by head chunks
curl -s http://prometheus:9090/metrics | grep prometheus_tsdb_head_chunks
```

### Grafana Debugging

```bash
# Check datasource health
curl -H "Authorization: Bearer ${API_KEY}" \
  http://grafana:3000/api/datasources/uid/prometheus/health

# Query inspector logs
tail -f /var/log/grafana/grafana.log | grep "level=error"

# Dashboard JSON validation
curl -X POST \
  http://grafana:3000/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @dashboard.json
```

### Alertmanager Debugging

```bash
# Check alert status
curl -s http://alertmanager:9093/api/v1/alerts | jq '.data[] | {labels: .labels, status: .status}'

# Test routing
curl -X POST http://alertmanager:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "critical",
      "service": "api"
    },
    "annotations": {
      "summary": "Test alert"
    },
    "generatorURL": "http://prometheus"
  }]'

# Check silences
curl -s http://alertmanager:9093/api/v1/silences | jq '.data[] | {id: .id, matchers: .matchers, status: .status}'
```

## 10. Integration with Claude Code

When working with monitoring stack issues, provide Claude Code with:
1. Query Results: Output of PromQL queries, not just descriptions
2. Configuration: Relevant scrape configs, recording rules, alert definitions
3. Context: Time ranges when issues occurred, recent deployments
4. Goals: Specific SLO targets or performance requirements

### Example Prompts for Claude Code

```text
"Analyze this PromQL query performance. Why is it slow and how to optimize?
[query: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, pod))]"

"Help me design SLOs for a payment processing service with 99.95% availability requirement.
What SLIs should I use and what are the multi-window alert thresholds?"

"Debug why this alert isn't firing. Here are the recording rule and the alert rule configs..."

"Create a Grafana dashboard JSON for RED metrics of a microservices architecture
with 15 services. Include variables for cluster and service selection."

"Optimize this Loki query that's timing out: {app=\"api\"} |= \"error\" | json | line_format \"{{.msg}}\""
```
