# 14 — Observability: Monitoring & Debugging

## Quick Summary

You can't fix what you can't see. Observability is your window into what agents are actually doing in production.

This document covers **metrics, logs, traces, and alerting** — the infrastructure that lets you diagnose problems before customers notice them.

**Cost model:** Observability infrastructure adds 10-20% overhead. But it saves 10x that in debugging time.

**When to focus on this:** Multi-agent systems, distributed deployment, or high-reliability requirements.

---

## Three Pillars of Observability

### Pillar 1: Metrics (Quantitative)

Numeric measurements over time.

```
Example metrics:
- Requests per second (throughput)
- Latency (P50, P95, P99)
- Error rate (% of requests that failed)
- Token usage (cost per request)
- Model latency (time for LLM response)
```

**Tool:** Prometheus (time-series database)

**Use for:** Capacity planning, alerting, performance trends

---

### Pillar 2: Logs (Events)

Human-readable record of what happened.

```
Example logs:
2024-06-29T10:23:45.123Z [INFO] RequestID=abc123 Agent=fraud_detector Input="user_id=456"
2024-06-29T10:23:46.234Z [INFO] RequestID=abc123 Agent=fraud_detector ToolCall="search_account" Duration=1.1s
2024-06-29T10:23:47.567Z [INFO] RequestID=abc123 Agent=fraud_detector Output="fraud_score=0.75"
```

**Tool:** ELK Stack (Elasticsearch, Logstash, Kibana) or Datadog

**Use for:** Debugging, understanding what happened in a specific request

---

### Pillar 3: Traces (Causality)

Distributed tracing shows request flow across services.

```
Request enters system
├─ Agent 1 (100ms)
│  ├─ Tool A (50ms)
│  └─ Tool B (40ms)
├─ Agent 2 (80ms)
│  ├─ Database query (60ms)
│  └─ API call (15ms)
└─ Total: 180ms
```

**Tool:** Jaeger, Zipkin, DataDog

**Use for:** Understanding performance bottlenecks, debugging distributed systems

---

## Key Metrics to Track

### 1. Throughput Metrics

| Metric | Definition | Alert If |
|--------|-----------|----------|
| **Requests/sec** | Total requests entering system | < 100 (drop in traffic) or > 5000 (surge) |
| **Success rate** | % of requests that completed successfully | < 95% |
| **Error rate** | % of requests that failed | > 5% |

---

### 2. Latency Metrics

| Metric | Definition | Alert If |
|--------|-----------|----------|
| **P50 latency** | Median request time | > 100ms (degradation) |
| **P95 latency** | 95th percentile (slow customers) | > 500ms |
| **P99 latency** | 99th percentile (tail latency) | > 2000ms |

**Why percentiles matter:**
- P50 = typical user experience
- P99 = worst-case user experience
- P99 > P50 by 10x = system has tail latency problem

---

### 3. Resource Metrics

| Metric | Definition | Alert If |
|--------|-----------|----------|
| **CPU usage** | % of CPU being used | > 70% (approaching limit) |
| **Memory usage** | % of RAM being used | > 80% (risk of OOM) |
| **Disk usage** | % of disk space used | > 85% (may fill up) |
| **Network I/O** | Bytes sent/received per second | > 1Gbps (approaching network limit) |

---

### 4. AI-Specific Metrics

| Metric | Definition | Alert If |
|--------|-----------|----------|
| **Tokens/request** | Average tokens used per request | > 10k (expensive) |
| **LLM latency** | Time for model to respond | > 5000ms (slow) |
| **Model cost** | $ per request | > $1 (expensive) |
| **Cache hit rate** | % of requests hitting cache | < 70% (poor caching) |
| **Tool success rate** | % of tool calls that succeeded | < 95% (tool reliability issue) |

---

## Logging Best Practices

### 1. Structured Logging

Use JSON format, not free-form text.

```python
# Bad: free-form text
logger.info("User 123 requested analysis")

# Good: structured JSON
logger.info("user_analysis_requested", extra={
    "user_id": "123",
    "request_id": "abc456",
    "timestamp": "2024-06-29T10:23:45.123Z"
})
```

**Why it matters:** Can query and filter logs easily ("show me all requests for user 123")

---

### 2. Include Request ID

Track a request through entire system.

```python
import uuid

request_id = str(uuid.uuid4())

# In Agent class
logger.info("agent_starting", extra={
    "request_id": request_id,
    "agent_name": "fraud_detector",
    "input": input_data
})

# In Tool call
logger.info("tool_call", extra={
    "request_id": request_id,
    "tool_name": "search_account",
    "duration_ms": 150
})
```

**Benefit:** Grep logs by request_id, see complete request flow

---

### 3. Log at Right Levels

| Level | Use For | Example |
|-------|---------|---------|
| **DEBUG** | Development/troubleshooting | Variable values, function entry/exit |
| **INFO** | Important events | Request started, tool called, result ready |
| **WARNING** | Something unexpected | Retry attempt, cache miss, slow latency |
| **ERROR** | Something failed | API call failed, tool timeout |
| **CRITICAL** | System broken | Database unreachable, out of memory |

---

### 4. Avoid Logging Sensitive Data

Don't log passwords, tokens, PII.

```python
# Bad: logs sensitive data
logger.info(f"User login: {email} with password {password}")

# Good: redact sensitive data
logger.info(f"User login: {redact_email(email)}")
```

---

## Tracing Best Practices

### 1. Instrument Entry Points

Mark where requests enter system.

```python
from jaeger_client import Config

tracer = Config(
    config={
        'sampler': {'type': 'const', 'param': 1},
        'logging': True,
    },
    service_name='ai-agent',
).initialize_tracer()

@app.route('/process', methods=['POST'])
def process_request():
    span = tracer.start_span('process_request')
    try:
        # Process request
        return result
    finally:
        span.finish()
```

---

### 2. Trace Tool Calls

Show where time is spent.

```python
def call_tool(tool_name, **kwargs):
    span = tracer.start_span(
        'tool_call',
        tags={'tool': tool_name}
    )
    try:
        start = time.time()
        result = tool(**kwargs)
        span.set_tag('duration_ms', (time.time() - start) * 1000)
        span.set_tag('success', True)
        return result
    except Exception as e:
        span.set_tag('error', True)
        span.set_tag('error_message', str(e))
        raise
    finally:
        span.finish()
```

---

### 3. Trace State Changes

Important for debugging.

```python
def update_state(state_id, new_status):
    span = tracer.start_span(
        'state_transition',
        tags={
            'state_id': state_id,
            'old_status': current_status,
            'new_status': new_status
        }
    )
    try:
        # Update state
        return updated_state
    finally:
        span.finish()
```

---

## Alerting Strategy

### Alert Rule: High Error Rate

```prometheus
alert: HighErrorRate
expr: rate(requests_total{status="error"}[5m]) > 0.05
for: 5m
labels:
  severity: critical
annotations:
  summary: "High error rate detected"
  description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"
```

**What it means:** If 5% of requests fail for 5 minutes, page on-call engineer.

---

### Alert Rule: High Latency

```prometheus
alert: HighLatency
expr: histogram_quantile(0.95, rate(request_duration_seconds_bucket[5m])) > 1.0
for: 5m
labels:
  severity: warning
annotations:
  summary: "P95 latency is high"
```

---

### Alert Rule: Resource Exhaustion

```prometheus
alert: HighMemoryUsage
expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.15
for: 10m
labels:
  severity: warning
```

**What it means:** If available memory drops below 15%, warn ops team to investigate.

---

## Debugging Strategies

### Strategy 1: Trace a Specific Request

```bash
# Find request ID for specific user
curl -s http://logs.company.com/search?user_id=456 | grep request_id
# Output: "request_id": "abc123def456"

# Get all logs for that request
curl http://logs.company.com/search?request_id=abc123def456
# Shows:
# - Request entry
# - Agent reasoning steps
# - Tool calls and results
# - Final output
# - How long each step took
```

---

### Strategy 2: Compare Performance

```
Request A (fast):
├─ Agent init: 10ms
├─ Tool 1: 50ms (cache hit)
├─ Tool 2: 40ms (parallel)
└─ Total: 100ms

Request B (slow):
├─ Agent init: 10ms
├─ Tool 1: 2000ms (cache miss, slow API)
├─ Tool 2: 40ms (parallel)
└─ Total: 2050ms

Insight: Tool 1 is slow when cache misses. Need to investigate API.
```

---

### Strategy 3: Distribution Analysis

```
Latency distribution (histogram):
- 0-100ms: 50% of requests
- 100-500ms: 40% of requests
- 500-2000ms: 8% of requests
- >2000ms: 2% of requests

Insight: Bimodal distribution suggests two classes of requests.
- Fast path (cache hit): < 100ms
- Slow path (no cache): > 500ms
```

---

## Failure Modes

### 1. **Observability Blind Spot**

**What happens:** Outage occurs but you don't notice until customers complain.

**Why it occurs:**
- No alerting set up
- Metrics weren't recording
- Logs went to /dev/null

**Recovery:**
- Set up alerting for key metrics before pushing to production
- Test alerting (simulate failures, verify alert fires)
- Page on-call team immediately, don't wait for customers

---

### 2. **Monitoring Itself is Down**

**What happens:** Monitoring system crashes. Can't see what's broken.

**Why it occurs:**
- Single monitoring node
- No redundancy
- Monitoring overwhelmed by volume

**Recovery:**
- Replicate monitoring infrastructure
- Alerting separate from metrics storage
- Limit metric cardinality (don't track every user ID)

---

### 3. **Log Explosion**

**What happens:** Logging every detail. Logs grow 1TB/day. Can't query them.

**Why it occurs:**
- DEBUG level logging in production
- Logging large objects (full context, history)
- No log rotation/cleanup

**Recovery:**
- Sample logs (1% in production)
- Log only important data (not full request body)
- Implement retention policy (keep 30 days)

---

### 4. **Noisy Alerts**

**What happens:** Alerts fire constantly. Team ignores them. Real alert missed.

**Why it occurs:**
- Alerts too sensitive
- Temporary blips triggering alerts
- No alert deduplication

**Recovery:**
- Higher alert thresholds
- Longer evaluation window (5min not 1min)
- Alert deduplication (same alert within 5min = deduplicate)
- Alert fatigue management

---

## Real-World Example: Debugging a Production Issue

**Scenario:** Error rate suddenly jumps from 1% to 15%.

**Step 1: Alert fires**
```
Alert: HighErrorRate
Severity: Critical
Error rate: 15% (threshold: 5%)
Service: fraud_detector_agent
Time: 2024-06-29 14:23:15 UTC
```

**Step 2: Check metrics dashboard**
- Error rate: 15% ✓
- Latency: P95 = 800ms (elevated)
- Tool success rate: fraud_tool = 92%, account_tool = 60%
- → Insight: account_tool is failing

**Step 3: Check logs for account_tool**
```
curl http://logs.company.com/search?tool=account_tool&status=error
Results:
└─ Errors all have same pattern:
   "Connection timeout to auth service"
   Starting at: 2024-06-29 14:20:00 UTC
```

**Step 4: Check distributed traces**
```
Request flow:
Agent → account_tool (timeout after 5s) → retry → timeout → fail
(Tool has no circuit breaker, no cache)

vs. 

Pre-incident:
Agent → account_tool (50ms response, cache hit)
```

**Step 5: Root cause identified**
Auth service was updated at 14:20 UTC, responding slower now.

**Step 6: Immediate fix**
- Enable cache on account_tool (cache hits avoid timeout)
- Add circuit breaker (fail fast instead of timeout)
- Rollback auth service to previous version

**Step 7: Post-incident**
- Add alert for account_tool latency > 3 seconds
- Improve account_tool resilience (cache + circuit breaker)
- Incident review (why wasn't this caught in testing?)

---

## Observability Checklist

Before shipping to production:

- [ ] Prometheus configured (metrics collection)
- [ ] Grafana dashboards created (key metrics visible)
- [ ] Alerting rules defined (for business/technical metrics)
- [ ] On-call rotation established (who gets paged)
- [ ] Logging infrastructure deployed (ELK or equivalent)
- [ ] Request ID tracing implemented (correlate logs)
- [ ] Distributed tracing deployed (Jaeger or equivalent)
- [ ] Alert testing done (simulate failures, verify page)
- [ ] Runbooks written (what to do when alert fires)
- [ ] Dashboards accessible (no authentication barriers)
- [ ] Data retention set (how long to keep logs/metrics)
- [ ] PII redaction implemented (no leaking secrets)
- [ ] SLOs defined (what reliability target?)
- [ ] Error budget calculated (how much downtime allowed?)

---

## Best Practices

1. **Use percentiles not averages**
   - Average: 200ms (misleading if some requests take 5s)
   - P95: 500ms, P99: 2000ms (tells true story)

2. **Correlate metrics, logs, traces**
   - See metric spike → grep logs → trace request flow
   - All three together = powerful debugging

3. **Alert on user impact, not internals**
   - Bad: "CPU > 80%"
   - Good: "Error rate > 5%" or "P95 latency > 1s"

4. **Use semantic versioning for alerts**
   - Page on-call for critical issues (error rate, data loss)
   - Warn team for important issues (slow latency)
   - Notify for informational (cache miss, unusual pattern)

5. **Test alerting**
   - Deploy test alert
   - Trigger it intentionally
   - Verify page received
   - Document false positive handling

6. **Implement observability in code**
   - Don't add logging as afterthought
   - Use structured logging from start
   - Add tracing to important functions

7. **Monitor the monitors**
   - Is Prometheus scraping? Alerting system up?
   - What if alerting server down?
   - Have backup way to detect issues

8. **Use data-driven decisions**
   - "Should we scale?" → Check CPU/memory metrics
   - "Is feature working?" → Check error rate for that feature
   - "What's the bottleneck?" → Check trace breakdown

9. **Keep dashboards simple**
   - Don't put 100 graphs on one dashboard
   - Key metrics only (what decisions do we make based on this?)
   - Separate dashboards for different audiences

10. **Document your observations**
    - Why did we choose this metric?
    - What alert threshold and why?
    - What action should team take on alert?

---

## Summary

**Observability is your window into production.** Without it:
- Can't detect issues early
- Takes hours to debug problems
- Customers experience outages you don't know about

**Three pillars:**
- **Metrics** — Quantitative trends (Prometheus)
- **Logs** — Event records (ELK)
- **Traces** — Request flow (Jaeger)

**Key metrics:**
- Throughput (requests/sec, success rate)
- Latency (P50, P95, P99)
- Resources (CPU, memory, disk)
- AI-specific (tokens, cost, model latency)

**Key principle:** Observability isn't optional. Build it in from day 1, not as afterthought.

---

## Next Steps

→ Proceed to [15 — Failure Patterns](15-failure-patterns.md) for resilience strategies.

→ Or jump to [16 — Enterprise Blueprint](16-enterprise-blueprint.md) for large-scale architectures.

→ Continue to [17 — Cheat Sheet](17-cheat-sheet.md) for quick reference.
