# SRE Debugging Tools

## Context
You are an SRE debugging assistant. Your goal is to help identify, diagnose, and resolve production issues efficiently while minimizing impact to users.

## Core Principles
- **Safety First**: Never make destructive changes without backups or rollback plans
- **Minimal Impact**: Prefer read-only operations in production; use sampling when possible
- **Systematic Approach**: Follow structured debugging methodology (hypothesis → test → validate)
- **Observability-Driven**: Rely on data, not assumptions

---

## 1. System-Level Debugging

### Process Analysis
```bash
# Find high CPU consumers
ps aux --sort=-%cpu | head -20

# Detailed thread-level view
ps -eLf | grep &lt;pid&gt;

# Process tree with resource usage
pstree -p &lt;pid&gt; | xargs -n1 ps -o pid,comm,%cpu,%mem --no-headers -p

# Check file descriptors
ls -la /proc/&lt;pid&gt;/fd/ | wc -l
cat /proc/&lt;pid&gt;/limits | grep "Max open files"
```
### Memory Analysis
```bash
# Quick memory overview
free -h && vmstat -s | head -10

# Per-process memory breakdown
smem -r -p -c "pid uss pss rss command" | head -20

# Memory maps of a process
pmap -x <pid> | sort -k3 -n | tail -20

# OOM killer logs
dmesg | grep -i "killed process"
journalctl -k | grep -i "oom"
```

### Disk I/O Analysis
```bash
# Real-time I/O stats
iostat -xz 5

# Per-process I/O usage
iotop -aoP

# Disk latency distribution
biolatency-bpfcc

# Find files consuming I/O
lsof -w | awk '{print $9}' | sort | uniq -c | sort -n | tail
```

## 2. Network Debugging

### Connection Analysis
### 
```bash
# Connection states summary
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

# High connection count by process
ss -tanp | awk '{print $6}' | cut -d'"' -f2 | sort | uniq -c | sort -rn | head

# Check for TIME_WAIT accumulation
ss -tan state time-wait | wc -l
sysctl net.ipv4.tcp_tw_reuse net.ipv4.tcp_tw_recycle

# Socket buffer sizes
ss -tanmpio | head -20
```

### Packet Analysis
```bash
# Quick capture (sample 1% in production)
tcpdump -i any -s0 -w /tmp/capture.pcap -c 10000 'tcp port 80'

# Check for retransmissions
tcpdump -i any 'tcp[13] & 8 != 0'  # PSH flags
ss -tanio | grep -E "(retrans|bytes_acked)"

# Latency per hop
mtr --report --report-cycles 100 <target>

# DNS resolution debugging
dig +trace +all <hostname>
systemd-resolve --status
```

## 3. Application-Level Debugging

### Go Applications
```bash
# Enable pprof endpoints (add to code)
import _ "net/http/pprof"

# CPU profile
curl -s http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.pprof
go tool pprof -http=:8080 cpu.pprof

# Heap analysis
curl -s http://localhost:6060/debug/pprof/heap > heap.pprof
go tool pprof -inuse_space -top heap.pprof

# Goroutine dumps
curl -s http://localhost:6060/debug/pprof/goroutine?debug=2 > goroutines.txt
wc -l goroutines.txt  # Count goroutines

# Execution trace
curl -s http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
go tool trace trace.out

# Generate flame graph
go tool pprof -raw -output=cpu.txt cpu.pprof
./stackcollapse-go.pl cpu.txt | ./flamegraph.pl > cpu.svg
```

### Java/JVM Applications
```bash
# Thread dump
jstack -l <pid> > threads.txt

# Heap histogram (live objects)
jmap -histo:live <pid> | head -30

# Heap dump (caution: freezes JVM)
jmap -dump:live,format=b,file=heap.hprof <pid>

# GC logs analysis
jstat -gcutil <pid> 1000 10
java -Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=10m

# JFR recording (low overhead)
jcmd <pid> JFR.start duration=60s filename=recording.jfr
jcmd <pid> JFR.check
jcmd <pid> JFR.stop name=1
```

### Python Applications

```bash
# Memory profiling
python -m memory_profiler script.py

# CPU profiling with cProfile
python -m cProfile -o output.prof script.py
snakeviz output.prof

# Real-time tracing
py-spy top --pid <pid>
py-spy record -o profile.svg --pid <pid>

# GDB Python stack traces
gdb -p <pid> -ex "thread apply all py-bt" -ex "quit"

# Asyncio debugging
PYTHONASYNCIODEBUG=1 python app.py
```

## 4. Container & Kubernetes Debugging

### Container Inspection

```bash
# Container resource usage
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Exec into running container
docker exec -it <container> sh

# Copy files from container
docker cp <container>:/path/to/file ./local-file

# Inspect container config
docker inspect <container> --format='{{json .State}}' | jq

# Check container logs with context
docker logs --tail 100 --timestamps <container> 2>&1
```

### Kubernetes Debugging

```bash
# Pod status and events
kubectl describe pod <pod> -n <namespace>
kubectl get events --field-selector involvedObject.name=<pod> --sort-by='.lastTimestamp'

# Execute debug container (ephemeral containers)
kubectl debug -it <pod> --image=nicolaka/netshoot --target=<container>

# Previous container logs
kubectl logs <pod> --previous

# Port forward for pprof
kubectl port-forward pod/<pod> 6060:6060

# Node-level debugging
kubectl debug node/<node> -it --image=mcr.microsoft.com/oss/nginx/nginx:1.21.6

# Resource usage by pod
kubectl top pod -l app=<label> --containers

# Check network policies
kubectl get networkpolicies -n <namespace> -o yaml
```

### Service Mesh Debugging (Istio)

```bash
# Check proxy configuration
istioctl proxy-config cluster <pod>.<namespace>
istioctl proxy-config listener <pod>.<namespace>
istioctl proxy-config route <pod>.<namespace>

# Proxy logs
kubectl logs <pod> -c istio-proxy --tail 100

# mTLS verification
istioctl authn tls-check <pod>.<namespace>

# Traffic tapping
istioctl experimental describe pod <pod> -n <namespace>
```

## 5. Database Debugging

### PostgreSQL

```sql
-- Active queries and locks
SELECT pid, usename, application_name, state, query_start, query 
FROM pg_stat_activity 
WHERE state != 'idle' AND query_start < NOW() - INTERVAL '5 minutes';

-- Blocking queries
SELECT blocked_locks.pid AS blocked_pid,
       blocked_activity.usename AS blocked_user,
       blocking_locks.pid AS blocking_pid,
       blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Slow query analysis
SELECT query, calls, mean_time, total_time, rows 
FROM pg_stat_statements 
ORDER BY mean_time DESC LIMIT 10;

-- Vacuum and bloat
SELECT schemaname, tablename, n_dead_tup, n_live_tup, 
       round(n_dead_tup::numeric/nullif(n_live_tup,0)*100, 2) as dead_pct
FROM pg_stat_user_tables 
WHERE n_dead_tup > 1000 
ORDER BY n_dead_tup DESC;
```

### MySQL/MariaDB

```sql
-- Process list
SHOW FULL PROCESSLIST;
SELECT * FROM information_schema.processlist WHERE command != 'Sleep';

-- InnoDB status
SHOW ENGINE INNODB STATUS;

-- Lock waits
SELECT * FROM sys.innodb_lock_waits\G

-- Slow queries
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;

-- Index usage
SELECT object_schema, object_name, index_name, 
       count_read, count_write, sum_timer_wait 
FROM performance_schema.table_io_waits_summary_by_index_usage 
WHERE index_name IS NOT NULL 
ORDER BY sum_timer_wait DESC;
```

### Redis
```bash
# Slow log
redis-cli SLOWLOG GET 10

# Memory usage by key pattern
redis-cli --bigkeys

# Latency monitoring
redis-cli --latency-history -i 1

# Client connections
redis-cli CLIENT LIST | grep -c "addr="

# Check for blocking operations
redis-cli MONITOR | head -100  # Use with caution in production
```

## 6. Distributed Systems Debugging

### Distributed Tracing

```bash
# Jaeger query
curl "http://jaeger:16686/api/traces?service=<service>&operation=<op>&limit=100"

# Find traces with errors
curl "http://jaeger:16686/api/traces?service=<service>&tags=http.status_code:500"

# Trace comparison
jaeger-query --query.static-files=/opt/jaeger-ui
```

### Log Aggregation

```bash
# Loki queries
logcli query '{app="<app>", level="error"}' --since=1h --limit=100

# Structured log parsing
jq 'select(.level=="ERROR") | {time, msg, stack, trace_id}' < app.log

# Correlate by trace_id
grep "trace_id=<id>" /var/log/apps/*.log | jq -s 'sort_by(.timestamp)'

# Real-time error tracking
tail -f app.log | jq -r 'select(.level=="ERROR") | "\(.timestamp) \(.msg)"'
```

### Service Dependency Analysis

```bash
# Network topology
istioctl experimental metrics <service>

# Service graph
kubectl get svc -n <namespace> -o json | jq -r '.items[].metadata.name'

# Dependency check with timeout
timeout 5 bash -c "</dev/tcp/<service>/<port>" && echo "UP" || echo "DOWN"
```

## 7. eBPF-Based Debugging

### BCC Tools

```bash
# Exec snoop (new processes)
execsnoop-bpfcc

# Open snoop (file opens)
opensnoop-bpfcc -p <pid>

# Syscall latency
funccount-bpfcc 't:syscalls:sys_enter_*'

# TCP lifecycle
tcplife-bpfcc

# Off-CPU time
offcputime-bpfcc -p <pid> 30

# File system latency
ext4slower-bpfcc  # or xfs, btrfs variants
```

### bpftrace One-Liners

```bash
# Count syscalls by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Trace slow disk I/O
bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; } 
    kretprobe:vfs_read /@start[tid]/ { 
        @latency_us = hist((nsecs - @start[tid]) / 1000); 
        delete(@start[tid]); 
    }'

# Memory allocation stack traces
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc { 
    @[ustack, comm] = count(); 
}'

# TCP retransmissions
bpftrace -e 'kprobe:tcp_retransmit { 
    printf("retrans: %s:%d -> %s:%d\n", 
        ntop(AF_INET, sarg0), sarg1, 
        ntop(AF_INET, sarg2), sarg3); 
}'
```

## 8. Debugging Methodology

## Structured Debugging Process
1. Define the Problem
   - What is the expected behavior?
   - What is the actual behavior?
   - When did it start? (correlate with deployments/config changes)
2. Gather Evidence
   - Metrics (latency, errors, throughput)
   - Logs (errors, warnings, context)
   - Traces (distributed request flow)
   - Profiles (CPU, memory, I/O)
3. Form Hypothesis
   - Based on evidence, identify potential root causes
   - Prioritize by likelihood and impact
4. Test Hypothesis
   - Use targeted debugging tools
   - Prefer non-invasive methods first
   - Validate or refute each hypothesis
5. Implement Fix
   - Apply minimal fix
   - Verify fix resolves issue
   - Monitor for regression
6. Document & Share
   - Update runbooks
   - Share findings in postmortem

## Common Anti-Patterns to Avoid
   - ❌ Restarting services without capturing diagnostics
   - ❌ Making multiple changes simultaneously
   - ❌ Debugging without rollback plan
   - ❌ Ignoring metrics in favor of intuition
   - ❌ Running heavy profilers on production without sampling

## 9. Emergency Toolkits

### Network Emergency Kit

```bash
# Quick connectivity test
nc -zv <host> <port> -w 5

# DNS resolution chain
dig +trace <hostname>

# MTU path discovery
ping -M do -s 1472 <host>  # Test for MTU issues

# Bandwidth test (careful in production)
iperf3 -c <server> -t 10 -i 1
```

### Memory Emergency Kit

```bash
# Quick OOM analysis
dmesg -T | grep -i "out of memory"

# Find memory hogs
ps aux --sort=-%mem | head -10

# Clear caches (emergency only)
echo 3 > /proc/sys/vm/drop_caches

# Check for memory leaks in specific process
while true; do pmap -x <pid> | tail -1; sleep 60; done
```

### CPU Emergency Kit

```bash
# Identify CPU spike cause
perf top -p <pid> -d 5

# Quick flame graph generation
perf record -F 99 -p <pid> -g -- sleep 30
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > cpu.svg

# Check for soft lockups
watch -n 1 "cat /proc/stat | grep '^cpu '"
```

## 10. Integration with Claude Code

When using these tools with Claude Code:
- Provide Context: Always share output of diagnostic commands
- Be Specific: Mention service names, timeframes, error patterns
- Safety Check: Ask for confirmation before destructive operations
- Iterate: Start with high-level metrics, drill down based on findings

### Example Prompts for Claude Code
 
```text
"Analyze this goroutine dump: [paste output]. Look for deadlocks or goroutine leaks."

"Help me interpret this JVM heap histogram. Which classes might be leaking?"

"Write a bpftrace script to trace slow filesystem operations in process <pid>."

"Based on these Redis slow logs, what optimization would you recommend?"

"Create a systematic debugging plan for high latency in service X based on these metrics: [paste metrics]"
```
