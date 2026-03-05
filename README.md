# SRE skills pack for Claude Code

## Cтруктура SRE-скиллов для Claude Code

### Core Operations

| Файл                               | Описание                                                                 |
| ---------------------------------- | ------------------------------------------------------------------------ |
| [`sre_incident_management.md`](sre_incident_management.md)       | Управление инцидентами: классификация, эскалация, постмортемы, runbooks  |
| [`sre_debugging_tools.md`](sre_debugging_tools.md)           | Дебаггинг в продакшене: трассировка, логи, core dumps, remote debugging  |
| `sre_troubleshooting_playbooks.md` | Готовые сценарии для типичных проблем (OOM, deadlock, network partition) |

### Observability & Monitoring

| Файл                            | Описание                                                            |
| ------------------------------- | ------------------------------------------------------------------- |
| [`sre_monitoring_stack.md`](sre_monitoring_stack.md)       | Prometheus, Grafana, Datadog, New Relic — запросы, алерты, дашборды |
| `sre_logging_best_practices.md` | Структурированные логи, агрегация, корреляция trace\_id             |
| `sre_distributed_tracing.md`    | OpenTelemetry, Jaeger, Zipkin — анализ latency между сервисами      |
| `sre_slo_sli_sla_framework.md`  | Определение и расчёт error budgets, burn rate alerts                |

### Performance & Reliability

| Файл                       | Описание                                                           |
| -------------------------- | ------------------------------------------------------------------ |
| `sre_performance_tools.md` | Профилирование: CPU, memory, I/O — pprof, perf, eBPF, flame graphs |
| `sre_load_testing.md`      | k6, Locust, JMeter — проектирование и анализ нагрузочных тестов    |
| `sre_capacity_planning.md` | Прогнозирование роста, right-sizing, cost optimization             |
| `sre_chaos_engineering.md` | Gremlin, Litmus, Chaos Mesh — принципы и практики хаос-инжиниринга |

### Deployment & Infrastructure

| Файл                               | Описание                                                           |
| ---------------------------------- | ------------------------------------------------------------------ |
| `sre_cicd_pipeline_reliability.md` | Надёжные деплои: blue-green, canary, feature flags, rollback       |
| `sre_kubernetes_operations.md`     | K8s troubleshooting: pod lifecycle, networking, storage, HPA/VPA   |
| `sre_infrastructure_as_code.md`    | Terraform, Pulumi, Ansible — безопасные изменения, drift detection |
| `sre_cloud_platforms.md`           | AWS/GCP/Azure — специфика отказов, лимиты, best practices          |


### Security & Compliance

| Файл                                | Описание                                                 |
| ----------------------------------- | -------------------------------------------------------- |
| `sre_security_incident_response.md` | Реагирование на угрозы: forensics, containment, recovery |
| `sre_secrets_management.md`         | Vault, AWS Secrets Manager — ротация, аудит, zero-trust  |


### Architecture & Design

| Файл                              | Описание                                                          |
| --------------------------------- | ----------------------------------------------------------------- |
| `sre_production_readiness.md`     | Production Readiness Review (PRR) — чеклисты, критерии готовности |
| `sre_disaster_recovery.md`        | RTO/RPO, backup/restore, multi-region failover, DR drills         |
| `sre_circuit_breaker_patterns.md` | Устойчивость: retry, timeout, bulkhead, rate limiting             |
| `sre_database_reliability.md`     | Репликация, шардирование, резервное копирование, миграции         |


### Automation & Platform

| Файл                          | Описание                                                   |
| ----------------------------- | ---------------------------------------------------------- |
| `sre_automation_scripting.md` | Python/Go/Bash для рутинных операций, self-healing системы |
| `sre_platform_engineering.md` | Внутренние платформы, developer experience, golden paths   |
| `sre_gitops_practices.md`     | ArgoCD, Flux — drift, sync waves, progressive delivery     |


### Process & Culture

| Файл                           | Описание                                                         |
| ------------------------------ | ---------------------------------------------------------------- |
| `sre_postmortem_culture.md`    | Blameless postmortems, 5 Whys, корректирующие действия           |
| `sre_oncall_best_practices.md` | Организация дежурств: ротации, alert fatigue, runbook automation |

