# 17 — Cheat Sheet: Quick Reference Guide

## Pattern Decision Matrix

### When to Use Each Pattern

| Pattern | Best For | Team Size | Latency | Complexity | Cost |
|---------|----------|-----------|---------|-----------|------|
| **Single Agent** | Simple Q&A, classification | 1 | <500ms | Low | Low |
| **Router** | Multi-domain dispatch | 2-3 | <1s | Low | Low-Med |
| **Sequential** | Ordered pipeline tasks | 3-4 | <5s | Medium | Medium |
| **Parallel Workers** | Independent parallel tasks | 2-3 | <2s | Medium | Medium |
| **Orchestrator** | Dynamic workflows | 4-5 | <3s | High | High |
| **Network Agents** | Distributed, loosely coupled | 5+ | <5s | High | Medium |

---

## Problem → Solution Map

### Your Problem: "I have one question to answer"
→ **Single Agent**
- Example: "Is this email spam?"
- Latency: <100ms
- Cost: Lowest

---

### Your Problem: "I need to route to specialist agents"
→ **Router Pattern**
- Example: "Route support ticket to correct team"
- Latency: <1s
- Cost: Low

---

### Your Problem: "I need to execute ordered stages"
→ **Sequential Workflow**
- Example: "Analyze → Summarize → Translate"
- Latency: <5s
- Cost: Medium

---

### Your Problem: "I need to run parallel tasks"
→ **Parallel Workers**
- Example: "Check 4 fraud signals in parallel"
- Latency: <2s (not 8s)
- Cost: Medium

---

### Your Problem: "I need dynamic workflow routing"
→ **Orchestrator**
- Example: "Inventory → Fulfillment → Shipping (depends on decisions)"
- Latency: <3s
- Cost: High

---

### Your Problem: "I need distributed agents"
→ **Network Agents**
- Example: "Payment service talks to inventory service"
- Latency: <5s
- Cost: Medium-High

---

## Pattern Comparison Table

| Dimension | Single | Router | Sequential | Parallel | Orchestrator | Network |
|-----------|--------|--------|-----------|----------|--------------|---------|
| **Latency** | <100ms | <1s | <5s | <2s | <3s | <5s |
| **Concurrency** | 1 | 1 | Serial | N parallel | 1 coordinator | M distributed |
| **State mgmt** | Simple | Simple | Medium | Complex | Complex | Very complex |
| **Debugging** | Easy | Easy | Medium | Hard | Hard | Very hard |
| **Scaling** | Horizontal | Horizontal | Vertical | Horizontal | Horizontal | Horizontal |
| **Testing** | Easy | Easy | Medium | Hard | Hard | Very hard |
| **Cost/1M ops** | $1k | $1.5k | $3k | $2.5k | $5k | $6k |

---

## Failure Mode Quick Reference

### Single Agent
**Most likely failure:** External API timeout
**Fix:** Add timeout, circuit breaker, fallback

### Router
**Most likely failure:** Classifier sends to wrong specialist
**Fix:** Escalation fallback, manual review queue

### Sequential
**Most likely failure:** Token growth → context window exceeded
**Fix:** Checkpoint and compress at 70% threshold

### Parallel
**Most likely failure:** One worker slow, holds up result merging
**Fix:** Timeout per worker, partial results

### Orchestrator
**Most likely failure:** State corruption, workers don't follow plan
**Fix:** Immutable state, versioning, idempotent operations

### Network Agents
**Most likely failure:** Cascading failures (one down brings down others)
**Fix:** Circuit breaker per service, bulkhead, async communication

---

## Cost Quick Calculator

### Per 1 Million Requests

| Pattern | Model Cost | Infrastructure | Total |
|---------|-----------|-----------------|-------|
| Single Agent | $0.30 | $0.70 | **$1.00** |
| Router | $0.50 | $1.00 | **$1.50** |
| Sequential | $1.00 | $2.00 | **$3.00** |
| Parallel (4x) | $1.00 | $1.50 | **$2.50** |
| Orchestrator | $2.00 | $3.00 | **$5.00** |
| Network (3 agents) | $2.00 | $4.00 | **$6.00** |

**Example:** 10M requests/day with Router pattern
- 10M × $1.50 / 1M = $15/day = $450/month

---

## SLA (Service Level Agreement) Reference

### Production Targets

| Metric | Target | Warning | Critical |
|--------|--------|---------|----------|
| **Availability** | 99.9% | <99.8% | <99% |
| **P50 Latency** | <100ms | >150ms | >500ms |
| **P95 Latency** | <500ms | >800ms | >2s |
| **P99 Latency** | <2s | >3s | >5s |
| **Error Rate** | <0.1% | >0.5% | >1% |
| **Cache Hit** | >80% | <70% | <50% |

---

## Decision Tree: Which Pattern to Use?

```
Start here
│
├─ Simple Q&A?
│  └─ YES → Single Agent ✓
│
├─ Route to different specialists?
│  └─ YES → Router Pattern ✓
│
├─ Ordered pipeline (Stage 1 → Stage 2 → Stage 3)?
│  └─ YES → Sequential Workflow ✓
│
├─ Independent parallel tasks?
│  ├─ YES + Can merge results easily?
│  │  └─ YES → Parallel Workers ✓
│  └─ Complex result merging?
│     └─ YES → Orchestrator ✓
│
├─ Distributed, loosely coupled?
│  └─ YES → Network Agents ✓
│
└─ Still not sure?
   └─ Combine patterns or ask platform team
```

---

## Resilience Patterns Quick Reference

### For Every External Service Call

```python
# Template: Resilience stack
circuit_breaker + timeout + retry + fallback + bulkhead

# Timeout: 1-3 seconds for user-facing
# Retry: 3x with exponential backoff
# Fallback: Cached data or degraded result
# Bulkhead: Separate thread pool for this service
```

---

### Circuit Breaker States

| State | Behavior | Transition |
|-------|----------|-----------|
| **Closed** | Accept calls | → Open after N failures |
| **Open** | Reject calls (fail fast) | → Half-open after timeout |
| **Half-open** | Try one call | → Closed if success, Open if fail |

**Tuning:**
- Aggressive: threshold=3, timeout=30s
- Conservative: threshold=10, timeout=60s

---

## Observability Checklist

### Minimum Viable Observability

**Metrics (Prometheus):**
- [ ] Request throughput (requests/sec)
- [ ] Error rate (%)
- [ ] Latency (P50, P95, P99)
- [ ] Model cost ($ per request)

**Logs (ELK Stack):**
- [ ] Request ID correlation
- [ ] Agent name and version
- [ ] Tool calls and results
- [ ] Duration of each step

**Traces (Jaeger):**
- [ ] Request entry point
- [ ] Agent execution time
- [ ] Tool call breakdown
- [ ] Final latency

**Alerts:**
- [ ] Error rate > 5%
- [ ] P95 latency > 2x baseline
- [ ] Cache hit rate < 70%

---

## Scaling Decision Table

### "We're hitting limits. What should we do?"

| Bottleneck | Symptom | Solution |
|-----------|---------|----------|
| **CPU** | Inference slow | Vertical scale (bigger instance) |
| **Memory** | OOM errors | Vertical scale + streaming |
| **Network** | High latency | Caching + reduce payload |
| **Database** | Query slow | Read replicas + indexing |
| **Concurrency** | Queue backing up | Horizontal scale (more instances) |
| **Cost** | Too expensive | Caching + async processing |

---

## Common Questions & Answers

### Q: "How do I choose token budget per stage?"
**A:** `Total budget / Number of stages`
```
Total: 10k tokens
Stages: 4 (init, think, tool, format)
Per stage: 2.5k tokens max
```

### Q: "What if one parallel worker is slow?"
**A:** Timeout per worker, return partial results, or retry with different worker

### Q: "How do I debug a distributed system?"
**A:** Use request ID, trace through logs, check latency breakdown

### Q: "Should I cache this result?"
**A:** Yes if:
1. Expensive (costs >$0.01)
2. Stable (valid >1 minute)
3. Frequent (same query >10x/day)

### Q: "Multi-region or single region?"
**A:** Single region until:
1. Customer in different geography (latency >200ms)
2. Availability requirement >99.9%
3. Regulatory requirements (data residency)

### Q: "How big should my thread pool be?"
**A:** `Number of CPU cores × 2 to 4`
- Conservative: cores × 2
- Aggressive: cores × 4

### Q: "What's a good timeout?"
**A:** Depends on use case:
- User-facing: 1-3 seconds
- Batch processing: 30-60 seconds
- Always set one

### Q: "When should I add another microservice?"
**A:** Only when:
1. Different team owns it
2. Deployed independently
3. Clear API boundary
4. Would be too large otherwise

---

## Common Mistakes Summary

| Mistake | Problem | Fix |
|---------|---------|-----|
| No timeout | Hangs forever | Add timeout to every call |
| Infinite retries | Stack overflow | Add max retry limit |
| No circuit breaker | Cascading failures | Add CB for each dependency |
| Observability late | Can't debug | Add metrics/logs from start |
| One big model | Slow & expensive | Use router for specialists |
| No caching | Expensive & slow | Cache >70% of requests |
| Stateful agents | Hard to scale | Use state store, keep agents stateless |
| No fallback | Crashes on failure | Graceful degradation |

---

## Deployment Checklist

### Pre-Production
- [ ] Load testing passed (5x peak traffic)
- [ ] Error handling verified
- [ ] Timeouts configured
- [ ] Circuit breakers tested
- [ ] Monitoring dashboards live
- [ ] Alerting rules tested
- [ ] Runbooks written
- [ ] On-call team trained
- [ ] Rollback plan documented

### Day 1 Production
- [ ] Monitor latency P50/P95/P99
- [ ] Monitor error rate
- [ ] Check cache hit rate
- [ ] Verify regional replication (if multi-region)
- [ ] Check cost metrics

### Week 1
- [ ] No unexpected patterns
- [ ] Performance stable
- [ ] Cost in line with projections
- [ ] Alerting working as expected

---

## Technology Stack Recommendations

### Core Runtime
- **Python:** LangChain, LlamaIndex, Pydantic
- **Node.js:** LangChain, Vercel AI SDK
- **Go:** Custom (low latency needed)

### LLM Inference
- **Managed:** OpenAI, Anthropic, Gemini API
- **Self-hosted:** vLLM, TGI (HuggingFace Text Generation Inference)
- **Hybrid:** Local fast models + API slow models

### State Management
- **Database:** PostgreSQL (strong ACID), DynamoDB (eventual consistency)
- **Cache:** Redis (fast reads), Memcached (simple)
- **Message Queue:** RabbitMQ, Apache Kafka

### Observability
- **Metrics:** Prometheus
- **Logs:** ELK Stack, Datadog, New Relic
- **Traces:** Jaeger, DataDog
- **Combined:** Grafana (visualize all three)

### Deployment
- **Container:** Docker
- **Orchestration:** Kubernetes (complex), Docker Compose (simple)
- **Serverless:** AWS Lambda, Google Cloud Run
- **Managed:** Modal, Replicate

---

## Context Window & Token Budget

### Token Limits (Approximate)

| Model | Tokens | Cost/1M | Speed |
|-------|--------|---------|-------|
| GPT-4o | 128k | $5 | ~100ms |
| Claude 3.5 | 200k | $3 | ~150ms |
| Gemini 2.0 | 1M | $0.50 | ~200ms |
| Local LLaMA | 4k-8k | $0 (compute) | ~50ms |

### Budget Allocation Example (10k token budget)

```
System prompt: 500 tokens (5%)
Context/memory: 3000 tokens (30%)
User input: 1000 tokens (10%)
Agent reasoning: 3500 tokens (35%)
Tool results: 2000 tokens (20%)
Reserve: 500 tokens (5%)
```

---

## Pattern Combinations Cheat Sheet

### Support Agent (Router + Sequential)
```
Customer message
├─ Router: Classify intent
├─ Sequential:
│  ├─ Retrieve context
│  ├─ Generate response
│  └─ Format answer
└─ Response
```

### Fraud Detection (Parallel + Router)
```
Transaction
├─ Parallel workers:
│  ├─ Velocity check
│  ├─ Device check
│  ├─ Geographic check
│  └─ Payment method check
├─ Combine results → fraud_score
├─ Router: Route to action
│  ├─ Low risk: Approve
│  ├─ Medium risk: Rules engine
│  └─ High risk: Human review
└─ Result
```

### Order Processing (Orchestrator + Parallel)
```
Order arrives
├─ Orchestrator assigns tasks:
│  ├─ Parallel: Payment + Inventory
│  ├─ Sequential: Fulfillment → Shipping
│  └─ Monitors progress
└─ Orchestrator decides:
   ├─ All successful? → Complete
   ├─ Inventory issue? → Route to warehouse
   └─ Payment failed? → Escalate to human
```

---

## Key Metrics Dashboard

### For Engineering Leaders

| Metric | Good | Warning | Crisis |
|--------|------|---------|--------|
| Request throughput | Stable | ±20% | ±50% |
| Error rate | <0.1% | 0.1-1% | >1% |
| P95 latency | <500ms | 500ms-1s | >1s |
| Cache hit | >80% | 70-80% | <70% |
| Cost/request | Within budget | +5% | +20% |
| Availability | 99.9%+ | 99-99.9% | <99% |

---

## Troubleshooting Decision Tree

### "Something's wrong"

```
Check error rate
├─ High? (>1%)
│  ├─ Check logs for pattern
│  ├─ Check circuit breakers (open?)
│  └─ Check external services (down?)
│
Check latency
├─ P95 slow? (>1s)
│  ├─ Check trace breakdown
│  ├─ Which step is slow?
│  ├─ Tool timeouts?
│  └─ Check cache hit rate
│
Check cost
├─ Higher than expected?
│  ├─ Check token usage
│  ├─ Check cache hit rate
│  └─ Check model version (newer = more expensive?)
│
Check availability
├─ Intermittent errors?
│  ├─ Check for transient network issues
│  ├─ Check retry logic
│  └─ Check circuit breaker half-open behavior
```

---

## One-Pagers: Pattern at a Glance

### Single Agent
- **Use when:** Simple question → answer
- **Latency:** <100ms
- **Cost:** $1 per 1M ops
- **Key metric:** Model accuracy
- **Most common failure:** External API timeout
- **Fix:** Circuit breaker + timeout

### Router
- **Use when:** Route to specialist agents
- **Latency:** <1s
- **Cost:** $1.50 per 1M ops
- **Key metric:** Classification accuracy
- **Most common failure:** Wrong routing
- **Fix:** Escalation + manual review

### Sequential Workflow
- **Use when:** Ordered stages (A → B → C)
- **Latency:** <5s
- **Cost:** $3 per 1M ops
- **Key metric:** Context window usage
- **Most common failure:** Token explosion
- **Fix:** Checkpoint + compress at 70%

### Parallel Workers
- **Use when:** Independent parallel tasks
- **Latency:** <2s
- **Cost:** $2.50 per 1M ops
- **Key metric:** Worker speed uniformity
- **Most common failure:** One slow worker blocks merge
- **Fix:** Per-worker timeout

### Orchestrator
- **Use when:** Dynamic workflow (depends on decisions)
- **Latency:** <3s
- **Cost:** $5 per 1M ops
- **Key metric:** State consistency
- **Most common failure:** State corruption
- **Fix:** Immutable state + versioning

### Network Agents
- **Use when:** Distributed, loosely coupled
- **Latency:** <5s
- **Cost:** $6 per 1M ops
- **Key metric:** Message delivery reliability
- **Most common failure:** Cascading failures
- **Fix:** Circuit breaker per link

---

## References by Topic

### Foundational Concepts
- Doc 01: Introduction (agent loop, principles)
- Doc 02: How to Choose (decision framework)
- Doc 03: Agent Lifecycle (execution phases, stop conditions)

### Pattern Documentation
- Doc 04: Single Agent
- Doc 05: Router Pattern
- Doc 06: Sequential Workflow
- Doc 07: Parallel Workers
- Doc 08: Orchestrator Workers
- Doc 09: Network Agents

### Infrastructure & Operations
- Doc 10: Memory & Context (RAG, semantic caching)
- Doc 11: State Management (consistency models)
- Doc 12: Tool Calling (schema design, resilience)
- Doc 13: Production Runtime (deployment, scaling)

### Reliability & Scale
- Doc 14: Observability (metrics, logs, traces)
- Doc 15: Failure Patterns (circuit breaker, bulkhead, retry)
- Doc 16: Enterprise Blueprint (multi-region, teams, governance)
- Doc 17: Cheat Sheet (this document)

---

## Final Thoughts

**Remember:**
1. Start simple (Single Agent)
2. Add complexity only when needed (Router, Sequential)
3. Build for production from day 1 (observability, resilience)
4. Scale horizontally, not vertically
5. Monitor everything
6. Test failures before they happen

**Architecture is a journey, not a destination.**

Start here. Build. Learn. Scale.

---

## Quick Navigation

**Learning Path:**
1. New to agents? Start with Doc 01-03
2. Building first system? Pick pattern from Doc 04-09
3. Going to production? Read Doc 10-16
4. Need quick answers? You're here! (Doc 17)

**By Use Case:**
- **Customer Support:** Router + Sequential (Doc 05, 06)
- **Fraud Detection:** Parallel + Router (Doc 07, 05)
- **Content Creation:** Sequential + Router (Doc 06, 05)
- **Search/Retrieval:** Single + Memory (Doc 04, 10)
- **Complex Workflows:** Orchestrator (Doc 08)
- **Microservices:** Network Agents (Doc 09)

**By Problem:**
- "Agents too slow?" → Parallel Workers + Caching
- "Too expensive?" → Router + Sequential + Caching
- "Hard to debug?" → Better observability + tracing
- "Unreliable?" → Resilience patterns + circuit breakers
- "Multiple teams?" → Enterprise Blueprint

---

**The playbook is complete. Now go build something great.**
