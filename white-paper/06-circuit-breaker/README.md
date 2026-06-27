# Section 6: Circuit Breaker Load Balancing

**Preventing cascading failures through fast failure**

**Author:** Dale Tomson

---

## The Problem: Cascading Failures

### In Traditional Systems

When a single backend begins failing, a cascade of failures can propagate across the entire system:

```
T=0:00  Backend starts returning 500 errors or timing out

T=0:00-0:30  Load balancer continues routing requests
             Client A: retries failed request
             Client B: retries failed request
             Client C: retries failed request
             
             Retries add MORE LOAD to already-failing backend

T=0:05  Failing backend is now completely overwhelmed
        Other healthy backends get redirected traffic
        Other backends become overloaded

T=0:10  Healthy backends start timing out too
        Cascading failure across system
        
T=0:30  Health check finally times out
        Backend removed from rotation
        System enters "death spiral"

Result: 30+ seconds of degradation and potential complete failure
```

### Health Checks Are Not Enough

Traditional health checks have a fatal flaw:

```
Health Check Interval: 5-30 seconds

During the gap:
  - Backend fails at T=0:05
  - Hundreds or thousands of requests sent to failing backend
  - Clients experience errors and timeouts
  - Retries compound the problem
  - Damage is done before health check can react

Example:
  1000 requests/second
  30-second health check interval
  → 30,000 requests sent to failing backend before it's detected
```

---

## The Circuit Breaker Pattern

### Electrical Engineering Inspiration

In electrical engineering, a circuit breaker:

1. **Allows normal current flow** (CLOSED state)
2. **Detects overload or fault** (counts failures)
3. **Immediately cuts power** (OPEN state - fast failure)
4. **Tests if problem is resolved** (HALF-OPEN - probing)

Atom Orchestrator applies the same pattern to **per-backend request routing**.

### Three States

#### State 1: CLOSED (Normal Operation)

```
Behavior:
  ✅ Requests pass through to backend
  ✅ Consecutive failures are counted
  ⚠️  When failures reach threshold (default: 5)
     → Transition to OPEN

Example timeline:
  Request 1: 200 OK        → failureCount = 0
  Request 2: 500 Error     → failureCount = 1
  Request 3: 200 OK        → failureCount = 0 (reset on success)
  Request 4: 500 Error     → failureCount = 1
  Request 5: 500 Error     → failureCount = 2
  Request 6: 500 Error     → failureCount = 3
  Request 7: 500 Error     → failureCount = 4
  Request 8: 500 Error     → failureCount = 5
  Request 9: ⚠️ CIRCUIT OPENS
```

#### State 2: OPEN (Failing Fast)

```
Behavior:
  ❌ ALL requests immediately rejected with HTTP 503
  ❌ No connection attempted to backend
  ❌ No timeout waited for
  ❌ Backend is ELECTRICALLY ISOLATED

Duration: 30 seconds (default cooldown)

Example:
  Request 1: ❌ 503 Service Unavailable (immediate)
  Request 2: ❌ 503 Service Unavailable (immediate)
  Request 3: ❌ 503 Service Unavailable (immediate)
  
  All requests fail instantly without hitting backend
  → Healthy backends can handle traffic instead
  → Prevents cascading failure
```

#### State 3: HALF-OPEN (Probing)

```
Behavior:
  ⚠️ Limited number of requests allowed through (default: 3)
  🔍 Used to probe if backend has recovered
  
  Success condition: 2 consecutive successful responses
    → Circuit CLOSES, backend reintegrated
  
  Failure condition: Any request fails
    → Circuit OPENS again, retry after cooldown

Example:
  OPEN state expires after 30s
  → Transition to HALF-OPEN
  
  Probe 1: 200 OK           → success count = 1
  Probe 2: 200 OK           → success count = 2
  ✅ Circuit CLOSES
  → Backend reintegrated into load balancer
  
  OR
  
  Probe 1: 200 OK           → success count = 1
  Probe 2: 500 Error        → ❌ immediate REOPEN
  → Return to OPEN for another 30s
```

### State Diagram

```
                 ┌─────────────────┐
                 │     CLOSED      │◄─────────────┐
                 │   (Normal)      │              │
                 └────────┬────────┘              │
                          │                       │ 2 successes
                   5 failures                     │ in HALF-OPEN
                          │                       │
                          v                       │
                 ┌─────────────────┐              │
                 │      OPEN       │              │
                 │  (Failing Fast) │              │
                 │    503 replies  │              │
                 └────────┬────────┘              │
                          │                       │
                   30s cooldown                   │
                          │                       │
                          v                       │
                 ┌─────────────────┐              │
                 │  HALF-OPEN      │──────────────┘
                 │   (Probing)     │
                 │ 3 test requests │
                 └─────────────────┘
                          │
                          │ Any failure
                          v
                  OPEN (restart cooldown)
```

---

## Integration with Load Balancer

### Selection Algorithm

```
function SelectBackend(element):
  1. Get all atoms for element
  
  2. Filter by health status
     unhealthy ← filter atom.health == UNHEALTHY
     healthy ← filter atom.health == HEALTHY
  
  3. Filter by circuit breaker state
     closed_circuits ← filter circuit[atom].state == CLOSED
     half_open_circuits ← filter circuit[atom].state == HALF_OPEN
     open_circuits ← filter circuit[atom].state == OPEN
  
  4. Primary selection pool
     if closed_circuits.length > 0:
       pool ← healthy ∩ closed_circuits
     else if half_open_circuits.length > 0:
       pool ← healthy ∩ half_open_circuits
     else:
       pool ← healthy  (all available, including OPEN)
  
  5. Select from pool
     if pool.length == 0:
       return NULL (no available backends)
     else:
       return SelectByStrategy(pool)  # Round-robin or Least-Conn
```

### Two-Layer Defense

```
Layer 1: Health Checker (5-second interval)
  - Detects slow degradation
  - Marks unhealthy over time
  - Example: Memory leak causing slowdown

Layer 2: Circuit Breaker (immediate)
  - Detects rapid failure
  - Rejects traffic immediately
  - Example: Crash loop, database connection loss

Combined effect: Fast response to both slow and fast failures
```

---

## Impact on System Behavior

### Without Circuit Breaker

```
Scenario: Backend database crashes

T=0:00  Database becomes unavailable
        Backend service tries to query database
        All queries timeout after 30 seconds

T=0:01-0:30  Load balancer routes requests to backend
             Every request waits 30 seconds
             Clients experience timeouts
             Clients retry
             Retries wait another 30 seconds

T=0:30  Health check finally detects timeout
        Backend marked unhealthy
        System recovers

Result: 30+ seconds of degradation
        Potential cascading failures
        Customer impact: Very high
```

### With Circuit Breaker

```
Scenario: Backend database crashes

T=0:00  Database becomes unavailable
        Backend service tries to query database
        Requests fail with 500 errors

T=0:01  1st error detected       → failureCount = 1
T=0:02  2nd error detected       → failureCount = 2
T=0:03  3rd error detected       → failureCount = 3
T=0:04  4th error detected       → failureCount = 4
T=0:05  5th error detected       → CIRCUIT OPENS

T=0:05  All new requests immediately rejected with 503
        Load balancer reroutes to healthy backends
        Other backends handle traffic
        System recovers instantly

T=0:35  (30s cooldown expires)
        HALF-OPEN: Allow probe requests

T=0:36  Probes succeed (database back online)
        Circuit CLOSES
        Backend reintegrated

Result: <5 seconds of fast-fail + 30s probing = graceful degradation
        Customer impact: Minimal
```

### Time Comparison

| Scenario | Without CB | With CB | Improvement |
|----------|-----------|---------|-------------|
| Detect failure | 5-30s | <1s | 5-30x faster |
| Load on failing backend | High (cascading) | Low (rejected) | 10-100x reduction |
| Healthy backend overload | Yes | Minimal | Prevented |
| System recovery | Slow | Fast | 5-10x faster |

---

## Configuration

Circuit breaker parameters are **configurable** via environment variables:

```bash
# Number of consecutive failures before opening circuit
export CB_FAILURE_THRESHOLD=5

# Number of consecutive successes in half-open to close
export CB_SUCCESS_THRESHOLD=2

# Cooldown duration before transitioning to half-open
export CB_TIMEOUT=30s

# Maximum probe requests in half-open state
export CB_HALF_OPEN_MAX=3
```

### Tuning for Different Workloads

#### High-Traffic Service

```
export CB_FAILURE_THRESHOLD=3      # Fail faster
export CB_SUCCESS_THRESHOLD=3      # Need more proof
export CB_TIMEOUT=10s              # Faster recovery probing
export CB_HALF_OPEN_MAX=5          # More probe requests
```

#### Batch Processing Service

```
export CB_FAILURE_THRESHOLD=10     # Tolerate more failures
export CB_SUCCESS_THRESHOLD=2      # Fewer probes needed
export CB_TIMEOUT=60s              # Longer recovery window
export CB_HALF_OPEN_MAX=2          # Fewer probe requests
```

#### Development/Testing

```
export CB_FAILURE_THRESHOLD=2      # Fail very quickly
export CB_SUCCESS_THRESHOLD=1      # Recover easily
export CB_TIMEOUT=5s               # Fast feedback
export CB_HALF_OPEN_MAX=3          # Standard probes
```

---

## Practical Examples

### Example 1: Memory Leak

```
Backend gradually consuming more memory over weeks

Week 1: Slow but handling
Week 2: Getting slower
Week 3: Increasingly timeouts

Scenario A (no circuit breaker):
  - Client requests accumulate
  - Timeouts cascade
  - Other backends overloaded
  - System fails gradually

Scenario B (with circuit breaker):
  - Failures detected after 5 consecutive timeouts
  - Circuit opens
  - Traffic redirected to healthy backends
  - Failing backend can be restarted
  - Recovery is immediate
```

### Example 2: Deployment Rollout

```
Deploying new version with a bug

Scenario A (no circuit breaker):
  - New instances deployed
  - Requests fail immediately
  - Load balancer doesn't know why
  - Clients retry, causing cascading failures
  - Takes manual intervention to fix

Scenario B (with circuit breaker):
  - New instances deployed and start failing
  - Circuit opens after 5 failures
  - Traffic stays on old (working) instances
  - No cascade, minimal customer impact
  - New version can be rolled back quickly
```

### Example 3: Thundering Herd

```
Coordinated attack or traffic spike

Scenario A (no circuit breaker):
  - All requests sent to backends
  - Backends overloaded
  - Response times spike to 10s+
  - Clients timeout and retry
  - Retry storm cascades

Scenario B (with circuit breaker):
  - Requests start failing
  - Circuit opens after 5 failures
  - Remaining traffic load-balanced across healthy backends
  - Some requests fail fast (503)
  - Other requests handled normally
  - Better user experience overall
```

---

## Failure vs. Slowness

### Important Distinction

```
Fast Failure (handled well):
  - Immediate 500 errors
  - Quick timeout (1-2 seconds)
  - Database unavailable
  → Circuit breaker triggers quickly

Slow Failure (harder to detect):
  - 20-second request processing
  - Memory leak gradual effect
  - Disk I/O contention
  → Circuit breaker takes longer (repeated slow requests)
  → Health checker picks it up sooner

Solution: Layered detection
  Layer 1: Health checks (detect slowness)
  Layer 2: Circuit breaker (detect failures)
```

---

## Summary

Circuit breaker load balancing provides:

| Benefit | Impact |
|---------|--------|
| **Fast failure detection** | <1 second vs. 5-30s |
| **Immediate rejection** | Prevents cascading |
| **Automatic recovery** | No manual intervention |
| **Graceful degradation** | Service continues at reduced capacity |
| **Customer experience** | Fast failures > slow timeouts |

---

## Next Steps

- **[Section 7: Comparison →](../07-comparison/README.md)**
- **[Section 8: Performance →](../08-performance/README.md)**
- **[Back to White Paper Index](../README.md)**

---

**Author:** Dale Tomson | **Date:** June 2026 | **License:** GNU AGPL v3.0
