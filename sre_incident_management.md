# SRE Incident Management

## Context
You are an SRE incident commander. Your goal is to coordinate effective incident response, minimize business impact, restore service quickly, and ensure organizational learning through structured post-incident analysis.

## Core Principles
- **People &gt; Process &gt; Tools**: Human safety and wellbeing come first
- **Blameless Culture**: Focus on systemic failures, not individual mistakes
- **Communication is Critical**: Clear, timely updates to stakeholders at all levels
- **Preserve Evidence**: Capture logs, metrics, and timeline before they disappear
- **Learn and Improve**: Every incident is an opportunity to make the system more resilient

---

## 1. Incident Severity Classification

### Severity Matrix

| Severity  | Impact                                                              | Response Time     | Examples                                                                   | Communication                                              |
| --------- | ------------------------------------------------------------------- | ----------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------- |
| **SEV-1** | Complete service outage; critical business function down; data loss | Immediate (5 min) | Payment system down; all users cannot login; database corruption           | Immediate executive notification; war room; 15-min updates |
| **SEV-2** | Major functionality degraded; significant customer impact           | &lt; 30 minutes   | Core API latency &gt; 10s; 50% of requests failing; checkout broken        | Manager notification; dedicated channel; 30-min updates    |
| **SEV-3** | Partial degradation; workaround available; limited customer impact  | &lt; 2 hours      | Non-critical feature broken; single region issues; performance degradation | Team notification; async updates; hourly status            |
| **SEV-4** | Minor issue; no customer impact; preventive/fix work                | &lt; 24 hours     | Monitoring gaps; certificate expiring in 30 days; code cleanup             | Ticket tracking; daily standup mention                     |
| **SEV-5** | Post-incident follow-up; process improvements                       | Best effort       | Postmortem action items; runbook updates; automation                       | Weekly sprint planning                                     |

### Severity Escalation Criteria

```yaml
# Automated severity escalation rules
escalation_rules:
  sev1_triggers:
    - error_rate &gt; 99% for 2 minutes
    - checkout_completion_rate &lt; 10%
    - payment_processing_rate == 0
    - data_loss_detected: true
    - security_breach_confirmed: true
    
  auto_escalate:
    - if: sev2_unresolved_for &gt; 30_minutes
      then: escalate_to_sev1
    - if: customer_facing_and_no_update_for &gt; 15_minutes
      then: page_incident_commander
    - if: multiple_sev2_same_service_within_24h
      then: escalate_to_sev1_and_initiate_rollback_review
```

## 2. Incident Response Lifecycle
### Phase 1: Detection & Declaration (0-5 minutes)
#### Automated Detection

```python
# Incident detection automation example
def evaluate_incident_conditions(metrics):
    """
    Auto-detect and declare incidents based on SLO violations
    """
    conditions = {
        'availability_breach': metrics.error_rate > 0.01,  # 1% error rate
        'latency_spike': metrics.p99_latency > 2.0,  # 2 seconds
        'saturation_critical': metrics.cpu > 95 or metrics.memory > 95,
        'business_metric_drop': metrics.checkout_rate < baseline * 0.5
    }
    
    if sum(conditions.values()) >= 2:
        return declare_incident(
            severity=infer_severity(conditions),
            channels=['pagerduty', 'slack', 'webex'],
            auto_page=True
        )
```

### Manual Declaration Template

## Incident Declaration

**Incident ID**: INC-2024-001234
**Declared**: 2024-03-15 14:32:05 UTC
**Severity**: SEV-2
**Reporter**: @sre-oncall
**Commander**: TBD (awaiting response)

**Symptoms**:
- API latency p99 &gt; 5s (normal: 200ms)
- Error rate 15% on /payment endpoints
- Affected regions: us-east-1, us-west-2

**Impact**:
- ~50% of checkout attempts failing
- Estimated 1000 customers affected in last 10 minutes

**Initial Hypothesis**:
- Database connection pool exhaustion (observed 500/500 connections in use)

**Status Page**: Investigating
**War Room**: #incident-2024-001234
**Bridge**: https://meet.example.com/inc-2024-001234

### Phase 2: Response & Coordination (5-30 minutes)
#### Incident Commander Checklist

#### Immediate Actions (First 5 Minutes)

- [ ] Acknowledge pagerduty alert
- [ ] Join war room (voice + chat)
- [ ] Assess personal safety (if on-call fatigue, escalate)
- [ ] Confirm severity level
- [ ] Identify required roles:
  - [ ] Incident Commander (IC)
  - [ ] Technical Lead (TL)
  - [ ] Communications Lead (CL)
  - [ ] Scribe/Timer
- [ ] Create incident document
- [ ] Notify stakeholders based on severity
  
#### Role Definitions

| Role                         | Responsibilities                                                                                   | Can Also Perform                               |
| ---------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| **Incident Commander (IC)**  | Overall coordination; decision authority; stakeholder communication; resource allocation           | Technical tasks if needed, but should delegate |
| **Technical Lead (TL)**      | Technical investigation; hypothesis testing; implementation of fixes; system expertise             | Communication if CL unavailable                |
| **Communications Lead (CL)** | Status page updates; customer communication; internal stakeholder updates; social media monitoring | Technical documentation                        |
| **Scribe**                   | Timeline documentation; decision log; action tracking; evidence preservation                       | Technical tasks if not busy                    |
| **Operations Lead**          | Infrastructure changes; scaling decisions; rollback execution; vendor coordination                 | Technical investigation                        |

### Communication Templates

## Initial Status Update (T+5 minutes)

🚨 **SEV-2 Incident Declared: Payment Processing Degradation**

**Status**: Investigating
**Impact**: ~50% of checkout attempts failing, ~1000 customers affected
**Regions**: us-east-1, us-west-2
**Started**: 14:32 UTC (10 minutes ago)

**What we're doing**:
- Database connection pool identified as likely cause
- Scaling database connections in progress
- Engaging database team

**Next update**: 15:00 UTC (in 15 minutes)

**War Room**: #incident-2026-001234

## Progress Update Template (Every 15-30 min)

⏳ **Update: Payment Processing Degradation (INC-2026-001234)**

**Status**: Mitigating
**Duration**: 45 minutes
**Current Impact**: Error rate reduced to 5% (from 15%)

**Actions Taken**:
- 14:45: Increased DB connection pool from 500 to 1000
- 15:00: Restarted affected API pods
- 15:10: Error rate dropping, latency improving

**Next Steps**:
- Monitoring for full recovery
- Root cause analysis pending

**ETA**: 15:30 UTC

### Phase 3: Mitigation & Resolution (30 min - hours)
### Decision Framework

#### Decision Log Template

| Time  | Decision                 | Rationale                               | Risk                          | Decision Maker |
| ----- | ------------------------ | --------------------------------------- | ----------------------------- | -------------- |
| 14:45 | Increase DB pool to 1000 | Connections exhausted, queue building   | Memory pressure on DB         | TL: @engineer  |
| 15:00 | Restart API pods         | Memory leak suspected in v2.3.1         | Brief availability hit        | IC: @commander |
| 15:15 | Enable circuit breaker   | Prevent cascade to inventory service    | Feature degradation           | TL: @engineer  |
| 15:30 | Rollback to v2.3.0       | Root cause identified in new deployment | Data consistency check needed | IC: @commander |

### Mitigation Strategies Priority
#### Fastest to Implement (minutes):
- Scale horizontally (add instances)
- Circuit breaker / feature flags
- Rate limiting / load shedding
- Failover to secondary region
#### Medium Effort (10-30 minutes):
- Configuration changes (timeouts, pool sizes)
- Restart/redeploy services
- Database query optimization
- CDN/cache rule changes
#### Significant Changes (30+ minutes):
- Code rollback
- Database schema changes
- Infrastructure provisioning
- Data fixes/migrations

### Phase 4: Recovery & Verification

## Recovery Checklist

- [ ] Error rates returned to baseline (&lt; 0.1%)
- [ ] Latency p99 &lt; 500ms sustained for 10 minutes
- [ ] All health checks passing
- [ ] Business metrics normalized (checkout rate, conversion)
- [ ] No new error patterns in logs
- [ ] Monitoring alerts cleared
- [ ] Status page updated to "Resolved"
- [ ] Final stakeholder communication sent
- [ ] Incident formally closed in tracking system
- [ ] Postmortem scheduled within 24 hours (SEV-1/2) or 1 week (SEV-3)

## 3. Communication Protocols

### Escalation Matrix
| Level | Trigger                                    | Action                   | Timeframe   |
| ----- | ------------------------------------------ | ------------------------ | ----------- |
| L1    | Auto-detected issue                        | Page on-call engineer    | Immediate   |
| L2    | No acknowledgment in 5 min                 | Page secondary on-call   | T+5 min     |
| L3    | SEV-1 declared or SEV-2 unresolved 30 min  | Page engineering manager | T+5 or T+30 |
| L4    | Customer data at risk or regulatory impact | Page director + legal    | Immediate   |
| L5    | Media attention or brand risk              | Page VP Engineering + PR | Immediate   |

### Stakeholder Notification Map

SEV-1:
├── Immediate (0 min)
│   ├── On-call engineer (pager)
│   ├── Incident Commander (pager)
│   ├── Engineering Manager (SMS)
│   └── CTO (email + SMS)
├── 15 minutes
│   ├── Customer Support (slack #support-leads)
│   ├── Sales Leadership (email)
│   └── Account Managers (for affected enterprise customers)
├── 30 minutes
│   ├── CEO (if > 1 hour duration)
│   ├── Legal (if data/privacy involved)
│   └── PR/Communications (if external visibility)
└── Every 15 minutes until resolved
    └── All above channels

SEV-2:
├── Immediate (0 min)
│   ├── On-call engineer (pager)
│   └── Team lead (slack)
├── 30 minutes (if unresolved)
│   └── Engineering Manager (slack)
└── Every 30 minutes until resolved

SEV-3:
├── Immediate (0 min)
│   └── On-call engineer (slack)
└── Hourly updates in team channel

### External Communication
#### Status Page Updates

#### Status Page Templates

**Investigating**:
We are currently investigating reports of [service] issues. 
Some users may experience [symptom]. We will provide updates as more information becomes available.

**Identified**:
We have identified the cause of [service] issues. 
The problem is related to [brief technical cause]. 
We are working to implement a fix. Next update in [timeframe].

**Monitoring**:
We have implemented a fix for [service] and are monitoring the results. 
Services should be returning to normal. We will confirm resolution shortly.

**Resolved**:
[Service] has been fully restored. 
The issue was caused by [root cause summary]. 
We are conducting a full postmortem and will implement preventive measures. 
We apologize for any inconvenience caused.

#### Customer Communication

## Enterprise Customer Notification

Subject: Service Impact Notification - [Company] [Service]

Dear [Customer] Team,

We are writing to inform you of a service impact affecting [specific functionality] 
between [start time] and [end time] UTC.

**Impact on Your Account**:
- [Specific data if available: e.g., "3 API requests failed during this window"]

**Root Cause**:
[Brief, technical but clear explanation]

**Preventive Actions**:
[Specific steps being taken to prevent recurrence]

Your Customer Success Manager will follow up within 24 hours. 
For immediate questions, contact support@example.com or your CSM directly.

Incident ID: INC-2026-001234

## 4. Technical Response Procedures
### Database Incident Response

#### Database Connection Pool Exhaustion

**Detection**:
- Alert: `database_connections_used / database_connections_max &gt; 0.9`
- Symptom: API latency spike, connection timeout errors

**Immediate Actions**:
1. Verify pool saturation: `SELECT count(*) FROM pg_stat_activity;`
2. Check for idle connections: `SELECT * FROM pg_stat_activity WHERE state = 'idle in transaction';`
3. Kill long-running idle transactions (if safe): `SELECT pg_terminate_backend(pid);`
4. Increase pool size temporarily: `ALTER SYSTEM SET max_connections = 1000; SELECT pg_reload_conf();`

**Investigation**:
- Identify connection leaks: Look for apps not releasing connections
- Check connection lifetime settings
- Review recent deployments for connection handling changes

**Resolution**:
- Scale application instances down (reduce total connections)
- Fix connection leak in code
- Implement connection pooling at application level (PgBouncer)

### Cascading Failure Response
## Circuit Breaker Activation

When: Service A calls Service B, and B is degraded

**Detection**:
- Error rate from Service B &gt; 50%
- Latency from Service B &gt; 5s
- Service A threads/connections backing up

**Immediate Actions**:
1. Activate circuit breaker (fail fast)
2. Return degraded response or cached data
3. Shed load if necessary (429 responses)
4. Scale Service B if resource-constrained

**Configuration**:
```yaml
circuit_breaker:
  failure_threshold: 50%  # Open after 50% failure rate
  slow_call_threshold: 5s  # Consider &gt;5s as failure
  wait_duration_open: 60s  # Try half-open after 1 min
  permitted_calls_half_open: 10  # Test with 10 requests
```

### Security Incident Response

## Suspected Security Breach

**Immediate (First 5 minutes)**:
1. Preserve evidence: Snapshot logs, don't restart services yet
2. Isolate affected systems (network segmentation)
3. Page security team and legal
4. Document everything (who accessed what, when)

**Containment (5-30 minutes)**:
1. Revoke compromised credentials
2. Enable additional logging/monitoring
3. Block suspicious IPs at edge
4. Take affected systems offline if necessary

**Investigation**:
- Preserve chain of custody for logs
- Identify scope of data access
- Determine attack vector
- Check for persistence mechanisms

**Recovery**:
- Rebuild affected systems from known-good images
- Rotate all potentially compromised secrets
- Implement additional monitoring
- Regulatory notification if required (72h for GDPR)

**Never**:
- Delete logs
- Restart without preserving memory dumps
- Communicate specifics externally without legal review
- Delay containment to preserve "evidence" if active breach


## 5. Post-Incident Analysis
### Timeline Reconstruction
## Incident Timeline Template (INC-2024-001234)

| Time (UTC) | Event                                 | Source                      | Notes                        |
| ---------- | ------------------------------------- | --------------------------- | ---------------------------- |
| 14:28:15   | Deployment v2.3.1 completed           | CI/CD logs                  | Deployed to 10% of traffic   |
| 14:29:30   | Database connection errors spike      | DB logs                     | First errors observed        |
| 14:32:05   | **Incident Declared**                 | PagerDuty                   | Auto-triggered by error rate |
| 14:32:30   | On-call engineer acknowledged         | PagerDuty                   | Response time: 25s           |
| 14:35:00   | War room established                  | Slack #incident-2024-001234 | IC: @alice, TL: @bob         |
| 14:38:00   | Hypothesis: Connection leak in v2.3.1 | Discussion                  | New connection pooling code  |
| 14:40:00   | DB connection pool increased          | TL action                   | Temporary mitigation         |
| 14:45:00   | Traffic shifted away from v2.3.1      | IC decision                 | 100% to v2.3.0               |
| 14:50:00   | Error rates dropping                  | Monitoring                  | Down to 5% from 15%          |
| 15:00:00   | Full rollback completed               | CI/CD                       | All pods on v2.3.0           |
| 15:15:00   | Error rates normalized                | Monitoring                  | < 0.1% sustained             |
| 15:20:00   | **Incident Resolved**                 | IC declaration              | Duration: 48 minutes         |
| 15:30:00   | Postmortem scheduled                  | Calendar                    | 2024-03-16 10:00 UTC         |

**Time to Detect**: 4 minutes (from first error to alert)
**Time to Acknowledge**: 25 seconds
**Time to Mitigate**: 13 minutes (to traffic shift)
**Time to Resolve**: 48 minutes

### Postmortem Structure

## Postmortem: Payment Processing Degradation (INC-2024-001234)

### Executive Summary
On March 15, 2024, between 14:28 and 15:20 UTC, approximately 50% of payment 
processing requests failed due to a database connection pool exhaustion caused 
by a connection leak in version 2.3.1. 1,247 customers were unable to complete 
purchases, with estimated revenue impact of $45,000.

### Impact
- **Duration**: 52 minutes
- **Severity**: SEV-2
- **Customers Affected**: 1,247 unique users
- **Failed Transactions**: 2,156
- **Revenue Impact**: $45,000 (estimated)
- **SLA Impact**: 0.02% monthly error budget consumed

### Root Cause Analysis

**5 Whys**:
1. Why did payments fail? → Database connection pool exhausted
2. Why did the pool exhaust? → Connections not being released after payment processing
3. Why weren't connections released? → New async payment code missing `finally` block
4. Why was this missed? → Code review focused on business logic, not resource management
5. Why wasn't this caught in testing? → Load test didn't simulate sustained high connection usage

**Root Cause**: Missing connection cleanup in async payment handler introduced in v2.3.1

### Contributing Factors
- Connection pool size (500) was near capacity during peak traffic
- No connection leak detection in staging environment
- Alert threshold (90% pool usage) too close to failure point

### What Went Well
- Automated detection triggered within 4 minutes
- Rollback procedure executed cleanly
- Communication to customers was timely and transparent

### What Went Wrong
- Connection leak not caught in code review
- Load testing gap for sustained high-load scenarios
- On-call engineer initially investigated wrong service

### Action Items

| ID  | Action                                                | Owner    | Due Date   | Priority |
| --- | ----------------------------------------------------- | -------- | ---------- | -------- |
| A1  | Add connection leak detection to staging              | @charlie | 2024-03-22 | P0       |
| A2  | Implement try-with-resources for all DB connections   | @bob     | 2024-03-20 | P0       |
| A3  | Increase pool size to 1000 with alerting at 70%       | @dave    | 2024-03-18 | P1       |
| A4  | Add connection usage metrics to payment dashboard     | @alice   | 2024-03-25 | P1       |
| A5  | Update code review checklist with resource management | @emily   | 2024-03-30 | P2       |

### Lessons Learned
- Resource cleanup patterns need explicit review
- Load tests must include sustained peak scenarios
- Connection pool metrics should be on primary dashboards

### Attachments
- [Full timeline CSV](link)
- [Grafana dashboard snapshot](link)
- [Deployment diff](link)

## 6. Automation & Tooling
### Incident Bot Commands

```yaml
# Slack bot integration for incident management
commands:
  declare:
    pattern: "/incident declare <severity> <description>"
    actions:
      - create_incident_channel: "incident-{timestamp}"
      - page_oncall: "{severity}"
      - create_incident_doc: from_template
      - start_meeting_bridge: auto
      - post_status_page: "investigating"
  
  update:
    pattern: "/incident update <status> [message]"
    actions:
      - update_status_page: "{status}"
      - notify_stakeholders: "{severity}_channels"
      - log_to_timeline: "{timestamp} - {message}"
  
  page:
    pattern: "/incident page <role> [message]"
    actions:
      - send_page: "{role}"
      - log_escalation: "{timestamp} - {role} paged: {message}"
  
  resolve:
    pattern: "/incident resolve [summary]"
    actions:
      - update_status_page: "resolved"
      - close_incident_channel: "archive_after=7d"
      - schedule_postmortem: "within 24h for sev1/2"
      - send_customer_comms: if_sev1_or_sev2
      - generate_timeline_export: for_postmortem
```

### Automated Evidence Collection

```python
# Incident evidence collector
def collect_incident_evidence(incident_id, start_time, end_time):
    """
    Automatically gather all relevant data for incident analysis
    """
    evidence = {
        'metrics': {
            'prometheus': query_prometheus_range(start_time, end_time),
            'grafana_snapshots': capture_dashboard_snapshots(incident_id),
        },
        'logs': {
            'loki': query_loki_range(start_time, end_time, labels={'severity': 'error'}),
            'cloudwatch': download_cloudwatch_logs(start_time, end_time),
        },
        'traces': {
            'jaeger': export_traces_with_errors(start_time, end_time),
        },
        'infrastructure': {
            'k8s_events': get_kubernetes_events(start_time, end_time),
            'deployments': get_deployment_changes(start_time, end_time),
            'config_changes': get_config_changes(start_time, end_time),
        },
        'communications': {
            'slack_export': export_channel_logs(incident_id),
            'pagerduty_timeline': get_pagerduty_logs(incident_id),
        }
    }
    
    # Store in incident-specific S3 bucket
    s3.upload(f"incidents/{incident_id}/evidence.json", evidence)
    return evidence
```

### Runbook Automation

```yaml
# Automated runbook execution
runbooks:
  database_connection_exhaustion:
    detection:
      query: "database_connections_used / database_connections_max > 0.9"
      duration: "2m"
    
    auto_mitigation:
      - action: increase_pool_size
        target: "current * 1.5"
        max: 2000
        approval: auto if sev >= 2
      
      - action: kill_idle_transactions
        condition: "idle_in_transaction > 30s"
        approval: auto
    
    human_required:
      - action: identify_connection_leak
        assign_to: "database_team"
      
      - action: deploy_fix
        assign_to: "on_call_engineer"
    
    rollback:
      - action: revert_pool_size
        when: "incident_resolved"
```
## 7. On-Call & Rotation Management
### Rotation Design

```yaml
# On-call rotation configuration
rotations:
  primary:
    schedule: "1 week rotations"
    team_size: 6
    timezone_coverage: "follow-the-sun"
    regions:
      - apac: "09:00-17:00 UTC+8"
      - emea: "09:00-17:00 UTC+1"
      - americas: "09:00-17:00 UTC-5"
    handoff: "mandatory_30min_overlap"
    
  secondary:
    schedule: "backup for primary"
    notification_delay: "5 minutes"
    
  shadow:
    schedule: "optional for training"
    permissions: "read_only_alerts"
```

### Health & Sustainability

## On-Call Wellbeing Guidelines

**Maximum Limits**:
- No more than 1 SEV-1 per 24 hours without mandatory rest
- No more than 3 pages per night (auto-escalate after)
- Maximum 7 consecutive days on-call
- Minimum 2 weeks between primary rotations

**Post-Incident Recovery**:
- Day off after SEV-1 that lasted &gt; 4 hours
- No non-urgent work assigned for 24h after major incident
- Optional debrief with manager if incident was traumatic

**Escalation for Fatigue**:
If on-call engineer feels unable to respond effectively:
1. Acknowledge and escalate immediately (no blame)
2. Secondary on-call takes over
3. Manager notified for coverage adjustment
4. Incident reviewed for process improvement (not personal performance)

## 8. Metrics & Improvement
### Incident Metrics Dashboard
```promql
# Mean Time To Detect (MTTD)
avg(
  timestamp(
    vector(1) and on() (
      ALERTS{alertname=~"IncidentDeclared",alertstate="firing"}
    )
  ) 
  - 
  timestamp(
    vector(1) and on() (
      ALERTS{alertname=~"ErrorRateHigh",alertstate="firing"}
    )
  )
)

# Mean Time To Acknowledge (MTTA)
avg(
  alertmanager_notifications_latency_seconds_sum 
  / 
  alertmanager_notifications_latency_seconds_count
)

# Mean Time To Resolution (MTTR)
avg(
  incident_resolution_timestamp 
  - 
  incident_start_timestamp
)

# Incident Frequency by Service
sum by (service) (
  increase(incidents_declared_total[30d])
)

# Error Budget Consumption
sum by (service) (
  increase(error_budget_burned_total[30d])
) 
/ 
sum by (service) (
  monthly_error_budget_total
)
```

### Continuous Improvement Process

## Quarterly Incident Review

**Review Meeting Agenda**:
1. Metrics review (MTTD, MTTA, MTTR trends)
2. Repeat incident analysis (same root cause &gt; 1 time?)
3. Action item completion review
4. Process pain points discussion
5. Tooling improvement proposals

**Outputs**:
- Updated runbooks
- Revised alert thresholds
- Training priorities
- Automation backlog
- Architecture improvement proposals

## 9. Integration with Claude Code
When managing incidents with Claude Code:
1. Provide Context: Share incident ID, severity, affected services
2. Share Evidence: Paste error logs, metrics snapshots, traces
3. Define Constraints: Budget, time, safety requirements
4. Request Specific Output: Decision templates, communication drafts, timeline analysis

### Example Prompts for Claude Code

```text
"Draft a SEV-2 incident declaration for API latency degradation. 
Symptoms: p99 latency > 5s on /checkout, error rate 8%, started 10 min ago. 
Service: payment-api, regions: us-east-1."

"Analyze this incident timeline and identify where we could have detected faster 
or mitigated sooner: [paste timeline]"

"Create a postmortem template for a database failover incident that lasted 45 minutes. 
Include 5 Whys section and action items table."

"Draft executive summary for CTO about SEV-1 incident. 
Impact: 2 hours downtime, $100K revenue loss, 5000 affected customers. 
Technical cause: DNS misconfiguration during migration."

"Review this communication draft for customer notification. 
Check tone, technical accuracy, and appropriateness for SEV-2 incident."

"Generate runbook for 'Database Primary Failure' scenario. 
Include detection, immediate actions, failover steps, verification, and rollback."
```