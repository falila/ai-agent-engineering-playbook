# 15 — Failure Patterns: Resilience & Recovery

## Quick Summary

Agents fail. Tools fail. Networks fail. Dependencies fail. The question isn't "if" but "when" and "what happens then?"

This document covers **resilience patterns** that let you survive failures gracefully instead of crashing spectacularly.

**Core principle:** Fail fast, degrade gracefully, recover automatically.

**Cost model:** Resilience patterns add 5-10% latency overhead. Outages cost 100x more.

---

## Pattern 1: Circuit Breaker

Stop calling a failing service. Fail fast instead of timing out.

### Problem

```python
# Bad: Call failing service. Wait 5 seconds. Timeout. Repeat.
def call_account_service(user_id):
    try:
        return requests.get(
            f"https://account-service/user/{user_id}",
            timeout=5  # Wait up to 5 seconds
        )
    except requests.Timeout:
        return None  # After 5 seconds of waiting
```

If service is down:
- Each request waits 5 seconds
- User experience: 5 second delay
- With 1000 concurrent requests: 5000 seconds of waiting
- Resource exhaustion: threads/connections blocked
- Cascading failure: entire system slows down

### Solution: Circuit Breaker

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half_open
    
    def call(self, func, *args, **kwargs):
        # State: "open" = stop calling, fail fast
        if self.state == "open":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "half_open"  # Try again
            else:
                raise CircuitBreakerOpen("Service temporarily unavailable")
        
        # Try the call
        try:
            result = func(*args, **kwargs)
            # Success! Reset
            self.failure_count = 0
            self.state = "closed"
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = "open"  # Stop calling
            raise

# Usage
breaker = CircuitBreaker(failure_threshold=5, timeout=60)

def call_account_service(user_id):
    try:
        return breaker.call(
            requests.get,
            f"https://account-service/user/{user_id}",
            timeout=2
        )
    except CircuitBreakerOpen:
        # Circuit open. Return cached data or default.
        return get_cached_account(user_id)
```

**Behavior:**
```
Call 1: Success ✓
Call 2: Success ✓
Call 3: Success ✓
Call 4: Fail (timeout)
Call 5: Fail (timeout)
Call 6: Fail (timeout) → Circuit opens
Call 7: Fail fast (0ms) → Circuit breaker
Call 8: Fail fast (0ms) → Circuit breaker
...after 60 seconds...
Call 9: Try again (half-open state)
Call 10: Success? Circuit closes
```

**Benefit:** From 5-second timeout to instant failure. Frees resources. Protects system.

---

## Pattern 2: Bulkhead Isolation

Separate concerns. One failure doesn't take down everything.

### Problem

```python
# Bad: All requests compete for same thread pool
executor = ThreadPoolExecutor(max_workers=10)

@app.route('/process', methods=['POST'])
def process():
    # Fast requests AND slow requests compete for same 10 threads
    return executor.submit(do_work)
```

Scenario:
- 5 fast requests (100ms each)
- 5 slow requests (5000ms each)
- New fast request arrives
- All 10 threads busy with slow requests
- Fast request waits 5 seconds (bad user experience)

### Solution: Bulkhead (Separate Pools)

```python
# Good: Separate thread pools by workload
fast_executor = ThreadPoolExecutor(max_workers=5)
slow_executor = ThreadPoolExecutor(max_workers=5)

@app.route('/process/fast', methods=['POST'])
def process_fast():
    # 5 threads reserved for fast work
    return fast_executor.submit(do_fast_work)

@app.route('/process/slow', methods=['POST'])
def process_slow():
    # 5 threads reserved for slow work
    return slow_executor.submit(do_slow_work)
```

**Benefit:** Slow requests don't block fast requests. Each gets own resources.

---

## Pattern 3: Timeout Strategy

Know when to give up.

### Problem

```python
# Bad: No timeout. Wait forever.
def call_tool(tool_name, **kwargs):
    return requests.post(
        f"https://tool-service/{tool_name}",
        json=kwargs
        # No timeout specified!
    )
```

If tool crashes:
- Request hangs indefinitely
- Thread blocked forever
- System slowly fills with blocked threads
- Eventually: no threads left, everything stuck

### Solution: Progressive Timeouts

```python
def call_tool_with_timeout(tool_name, **kwargs):
    timeout = get_timeout_for_tool(tool_name)
    
    try:
        return requests.post(
            f"https://tool-service/{tool_name}",
            json=kwargs,
            timeout=timeout  # Fail after N seconds
        )
    except requests.Timeout:
        # Tool too slow. Try fallback.
        return get_cached_result(tool_name, kwargs)

def get_timeout_for_tool(tool_name):
    # Different tools have different budgets
    timeouts = {
        "search": 1.0,      # Must be fast
        "database": 2.0,
        "analysis": 5.0,    # Can be slow
    }
    return timeouts.get(tool_name, 3.0)
```

**Budget per stage:**
```
Request budget: 10 seconds total
├─ Agent thinking: 1 second
├─ Tool 1 (search): 1 second
├─ Tool 2 (fetch): 2 seconds
├─ Tool 3 (analyze): 3 seconds
└─ Formatting: 1 second
```

If any tool exceeds budget → fallback or return partial result.

---

## Pattern 4: Retry with Exponential Backoff

Some failures are transient.

### Problem

```python
# Bad: Retry immediately
def call_api_bad():
    for i in range(3):
        try:
            return requests.get("https://api.example.com/data")
        except requests.ConnectionError:
            pass  # Retry immediately
```

If API temporarily overloaded:
- Retry immediately
- Service still overloaded
- Retry again immediately
- Service further overloaded (thundering herd)

### Solution: Exponential Backoff

```python
def call_api_with_retry(url, max_retries=3):
    for attempt in range(max_retries):
        try:
            return requests.get(url, timeout=2)
        except requests.ConnectionError as e:
            if attempt == max_retries - 1:
                raise  # Give up
            
            # Wait longer each time
            wait_time = 2 ** attempt  # 1s, 2s, 4s
            wait_time += random.uniform(0, 1)  # Add jitter
            time.sleep(wait_time)
```

**Retry pattern:**
```
Attempt 1: Fail → Wait 1-2 seconds
Attempt 2: Fail → Wait 2-3 seconds
Attempt 3: Fail → Wait 4-5 seconds
Attempt 4: Give up
```

**Why jitter matters:**
- Without: All clients retry at same time → thundering herd
- With: Clients retry at staggered times → service recovers

---

## Pattern 5: Graceful Degradation

Reduce functionality instead of crashing.

### Problem

```python
# Bad: Missing dependency = crash
def get_user_profile(user_id):
    # All required
    user = db.get_user(user_id)  # If fails, crash
    recommendations = ml_service.get_recommendations(user_id)  # If fails, crash
    social = social_service.get_social(user_id)  # If fails, crash
    
    return {
        "user": user,
        "recommendations": recommendations,
        "social": social
    }
```

If ML service down:
- All profile requests fail
- Users see error page

### Solution: Graceful Degradation

```python
def get_user_profile(user_id):
    user = db.get_user(user_id)  # Critical, required
    
    # Optional enhancements
    try:
        recommendations = ml_service.get_recommendations(user_id)
    except Exception:
        recommendations = []  # Degrade
    
    try:
        social = social_service.get_social(user_id)
    except Exception:
        social = {}  # Degrade
    
    return {
        "user": user,
        "recommendations": recommendations or [],  # May be empty
        "social": social or {}  # May be empty
    }
```

**Result:**
- Core feature works (user profile)
- Non-critical features degrade (no recommendations, no social)
- Better UX than crashing

---

## Pattern 6: Deadletter Queue

Handle messages that can't be processed.

### Problem

```python
# Bad: Message processing fails. Message lost forever.
def process_message(message):
    try:
        result = do_work(message)
        return result
    except Exception as e:
        logger.error(f"Failed: {e}")
        # Message gone. Lost. Can't retry.
```

If message processing fails:
- Message discarded
- No retry
- Data loss (silent)

### Solution: Deadletter Queue

```python
def process_message(message):
    try:
        result = do_work(message)
        queue.ack(message)  # Success, remove from queue
        return result
    except TemporaryError as e:
        # Transient error. Requeue for retry.
        queue.nack(message, requeue=True)
    except PermanentError as e:
        # Won't succeed on retry. Move to deadletter.
        deadletter_queue.put(message)
        logger.error(f"Deadlettered: {message}", extra={"error": str(e)})

# Later: Inspect deadletter queue
def inspect_deadletters():
    messages = deadletter_queue.get_all()
    for msg in messages:
        print(f"ID: {msg['id']}, Error: {msg['error']}, Payload: {msg['payload']}")
```

**Flow:**
```
Message arrives
│
├─ Transient error? → Requeue (retry later)
├─ Permanent error? → Deadletter (human review)
└─ Success? → Acknowledge (remove from queue)
```

---

## Pattern 7: Bulkhead Circuit Breaker Combo

Combine patterns for maximum resilience.

### Real Example: Payment Processing

```python
class PaymentProcessor:
    def __init__(self):
        # Separate circuit breakers for each payment method
        self.credit_card_breaker = CircuitBreaker(failure_threshold=3)
        self.bank_breaker = CircuitBreaker(failure_threshold=3)
        
        # Separate executors for high/low priority
        self.high_priority_pool = ThreadPoolExecutor(max_workers=10)
        self.low_priority_pool = ThreadPoolExecutor(max_workers=5)
    
    def process_payment(self, payment):
        # Timeout budget: 5 seconds total
        timeout = 5.0
        start = time.time()
        
        # Route to correct payment method
        if payment.method == "credit_card":
            breaker = self.credit_card_breaker
            timeout = 3.0  # Credit cards usually fast
        else:
            breaker = self.bank_breaker
            timeout = 4.0  # Banks slower
        
        # Route to correct executor
        if payment.priority == "high":
            executor = self.high_priority_pool
        else:
            executor = self.low_priority_pool
        
        # Execute with timeout
        try:
            future = executor.submit(
                breaker.call,
                self._call_payment_service,
                payment,
                timeout=timeout
            )
            result = future.result(timeout=timeout)
            return result
        except Exception as e:
            # Fallback: store for offline processing
            offline_queue.put(payment)
            return {"status": "pending", "will_process_offline": True}
```

---

## Failure Modes

### 1. **Circuit Breaker Stuck Open**

**What happens:** Circuit opens. Service recovers. Circuit never closes.

**Why:**
- Threshold too low (opens too easily)
- Timeout too long (half-open state waits too long)
- Metrics not resetting properly

**Fix:**
```python
# Monitor circuit state
for breaker_name, breaker in breakers.items():
    metrics.gauge(
        'circuit_breaker_state',
        1 if breaker.state == "open" else 0,
        tags={"breaker": breaker_name}
    )

# Alert if stuck open > 5 minutes
```

---

### 2. **Cascading Timeouts**

**What happens:** Timeout at every layer. 10ms work takes 50 seconds.

**Why:**
- Timeout = wait until timeout expires
- 5 layers × 10 second timeout = 50 seconds
- Nested calls all timeout

**Fix:**
```python
# Pass remaining budget down
def call_layer_1(request, budget=10):
    timeout = budget * 0.2  # Use 20% for this layer
    result = call_layer_2(request, budget=budget * 0.8)
    return result
```

---

### 3. **Retry Storms**

**What happens:** All requests retry at same time. Service overwhelmed.

**Why:**
- No jitter in retry logic
- Thundering herd problem

**Fix:**
```python
# Always add jitter
wait_time = (2 ** attempt) + random.uniform(0, 1)
```

---

### 4. **Resource Exhaustion from Queues**

**What happens:** Deadletter queue grows unbounded. Disk fills up.

**Why:**
- Messages can't be processed
- No cleanup of old deadletters
- No monitoring of queue size

**Fix:**
```python
# Limit deadletter queue size
deadletter_queue.max_size = 100000

# Alert if queue > 80% full
if deadletter_queue.size() > deadletter_queue.max_size * 0.8:
    alert("Deadletter queue filling up")

# Auto-cleanup old messages (>30 days)
if msg.created_time < now - 30*days:
    deadletter_queue.delete(msg)
```

---

## Real-World Example: E-Commerce Order Processing

**Scenario:** New Year's sale. 100x traffic. Payment service having issues.

**Without resilience patterns:**
```
Request surge
└─ All requests hit payment service
   └─ Payment service overwhelmed
      └─ Requests timeout (wait 5 seconds)
         └─ All threads blocked
            └─ System becomes unresponsive
               └─ Entire site down
```

**With resilience patterns:**
```
Request surge
└─ Circuit breaker detects payment failures
   └─ Open circuit → Fail fast (1ms instead of 5s)
      └─ Bulkhead: Priority orders process, others queue
         └─ Deadletter queue stores failed orders
            └─ Graceful degradation: Show "We'll process your order when service recovers"
               └─ Retry with backoff wakes up payment service
                  └─ Service recovers
                     └─ Deadlettered orders retry automatically
                        └─ All orders eventually process
```

**Result:**
- No user sees "System Down" error
- Orders complete when service recovers
- Staff can inspect deadletter queue
- Payment service has time to recover

---

## Engineering Notes

### Circuit Breaker Tuning

```python
# Conservative: Few failures tolerated
CircuitBreaker(
    failure_threshold=3,      # Open after 3 failures
    timeout=30,               # Retry after 30 seconds
    success_threshold=2       # Close after 2 successes
)

# Aggressive: Tolerates more failures
CircuitBreaker(
    failure_threshold=10,
    timeout=60,
    success_threshold=5
)
```

Choose based on:
- How often does service flake?
- How critical is it?
- Can we degrade without it?

### Retry Budget

```
Total request budget: 10 seconds
├─ Initial attempt: 3 seconds
├─ Retry 1: 2 seconds (wait 1s)
├─ Retry 2: 2 seconds (wait 2s)
└─ Retry 3: 2 seconds (wait 4s)
```

Sum of all retries ≤ budget.

### Bulkhead Sizing

```
Bulkhead for "fast" operations:
- Max workers: 10
- Timeout: 100ms
- Used for: User-facing requests

Bulkhead for "background" operations:
- Max workers: 5
- Timeout: 30 seconds
- Used for: Batch processing, cleanup
```

---

## Common Mistakes

### Mistake 1: Retry Without Checking Retry Count

```python
# Bad: Infinite retries
def call_api(url):
    try:
        return requests.get(url)
    except:
        return call_api(url)  # Retry infinitely!
```

**War story:** Service crashed. Called itself recursively 10,000 times. Stack overflow. Everything down.

**Fix:** Always have max retry limit.

---

### Mistake 2: Ignoring Deadlettered Messages

```python
# Bad: Messages disappear into deadletter
deadletter_queue.put(message)
# Nobody ever looks at this queue
```

**War story:** Deadletter queue grew to 10 million messages over 6 months. Nobody noticed. Data loss.

**Fix:** Monitor and alert on deadletter queue size.

---

### Mistake 3: No Fallback for Circuit Breaker

```python
# Bad: Circuit opens, returns None
if breaker.is_open():
    return None  # User sees broken feature

# Good: Return cached/default value
if breaker.is_open():
    return get_cached_result()  # User sees stale data
```

---

### Mistake 4: Timeouts Too Long

```python
# Bad: Wait 30 seconds
requests.get(url, timeout=30)

# Under load:
# 1000 concurrent requests × 30 seconds = 30,000 seconds of waiting
# All threads blocked
# System unresponsive
```

**Fix:** Short timeouts (1-3 seconds for user-facing requests).

---

## Resilience Checklist

- [ ] Circuit breaker implemented for each external dependency
- [ ] Bulkheads created for different workload types
- [ ] Timeouts set for all blocking operations
- [ ] Retry logic with exponential backoff implemented
- [ ] Jitter added to all retries
- [ ] Graceful degradation for non-critical features
- [ ] Deadletter queue for failed messages
- [ ] Alerting on deadletter queue size
- [ ] Alerting on circuit breaker state
- [ ] Fallback data source (cache, default) for when services down
- [ ] Max retry limits enforced
- [ ] Request budget enforced (timeout at each layer)
- [ ] Chaos engineering tests (simulate failures)
- [ ] Runbooks for common failure scenarios

---

## Best Practices

1. **Fail fast, not slowly**
   - 1ms error > 5s timeout
   - Preserve resources for working requests

2. **Combine patterns**
   - Circuit breaker (stop calling)
   - Bulkhead (isolate impact)
   - Timeout (don't wait forever)
   - Retry (if transient)

3. **Monitor the monitors**
   - Alert on circuit breaker state
   - Alert on deadletter queue size
   - Alert on retry count

4. **Use timeouts everywhere**
   - HTTP requests
   - Database queries
   - Cache lookups
   - RPC calls

5. **Implement graceful degradation**
   - Core features: required
   - Nice-to-have features: optional
   - Missing = degrade, don't crash

6. **Test failure scenarios**
   - Simulate service outages
   - Verify circuit opens
   - Verify fallback works
   - Chaos engineering

7. **Metrics-driven decisions**
   - "Why is circuit breaker threshold 5?" → Data shows flakiness pattern
   - "Why is timeout 2 seconds?" → P99 latency history
   - "Why this retry policy?" → Measured recovery time

8. **Document fallback behavior**
   - If service X down, what happens?
   - Is data loss possible?
   - What can users do?

9. **Version your resilience policies**
   - Different policies for different deployments
   - Gradually roll out changes
   - Measure impact

10. **Remember: Resilience isn't free**
    - Complexity increases
    - Harder to debug
    - Worth it for production systems

---

## Summary

**Resilience patterns let you survive failures gracefully.**

**Key patterns:**
- **Circuit breaker** — Stop calling failing services, fail fast
- **Bulkhead** — Isolate workloads, prevent cascade
- **Timeout** — Give up, don't wait forever
- **Retry** — Transient errors retry, permanent errors fail
- **Graceful degradation** — Core required, nice-to-have optional
- **Deadletter** — Handle unprocesable messages, enable recovery

**Core principle:** Fail fast. Isolate failures. Recover automatically.

**Cost:** 5-10% latency overhead. Prevents 100x worse outage costs.

---

## Next Steps

→ Proceed to [16 — Enterprise Blueprint](16-enterprise-blueprint.md) for large-scale architectures.

→ Or jump to [17 — Cheat Sheet](17-cheat-sheet.md) for quick reference.
