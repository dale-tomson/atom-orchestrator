# Section 4: Core Mechanisms

**How scaling, updates, and health management work**

**Author:** Dale Tomson

---

## Overview

This section details the three core operational mechanisms that define how Atom Orchestrator manages the container lifecycle:

1. **Horizontal Auto-Scaling** - Adding and removing capacity based on utilization
2. **Zero-Downtime Rolling Updates** - Deploying new versions without service interruption
3. **Graceful Shutdown** - Clean termination with connection draining

---

## Mechanism 1: Horizontal Auto-Scaling

### Utilization-Based Decision Making

Scaling decisions are based on **aggregate resource utilization** across all atoms of an element.

#### CPU Utilization Calculation

```
totalAllocatedCpu = atomCount × spec.containerCpu

If an element has:
  - 5 atoms
  - 0.2 CPU each
  Then: totalAllocatedCpu = 1.0 CPU

At any given time:
  - actual usage might be 0.85 CPU
  - utilization = 0.85 / 1.0 = 85%
```

#### Memory Utilization Calculation

```
totalAllocatedMem = atomCount × spec.containerMem

If an element has:
  - 5 atoms
  - 256 MB each
  Then: totalAllocatedMem = 1.28 GB

At any given time:
  - actual usage might be 1.0 GB
  - utilization = 1.0 / 1.28 = 78%
```

### Scaling-Up Algorithm

#### Trigger Condition
```
if cpuUtilization >= spec.scaleAtCpuUsage:
  AND currentAllocatedCpu < spec.maxCpuToUse:
    spawnAtom()
```

#### Process

```
1. Identify template atom
   - Select any healthy, running atom
   - Use docker inspect to get full config

2. Create new atom
   - docker create with identical config
   - Apply resource limits (CPU, memory)
   - Add orchestrator tracking label

3. Start new atom
   - docker start {new-atom-id}

4. Register with load balancer
   - Mark unhealthy initially (warming up)

5. Health checks begin
   - Every 5 seconds, probe /health endpoint
   - Once 2xx response: mark HEALTHY

6. Load balancer includes in rotation
   - New atom now receives traffic

7. Orchestrator state updated
   - In-memory registry reflects new atom
   - Metrics updated
```

#### Timing

```
Scale-up is IMMEDIATE:

T=0:00   CPU utilization exceeds threshold
T=0:00   Spawn decision made
T=0:01   New atom created and started
T=0:05   First health check passes
T=0:05   Load balancer begins routing traffic
         ↑
         System is ready to handle load
```

### Scaling-Down Algorithm

#### Trigger Condition
```
if cpuUtilization <= spec.killAtCpuUsage:
  AND atomCount > 1:
    killNewestAtom()
```

#### Key Design Decision: Conservative Scale-Down

Scale-down is **intentionally conservative** to prevent oscillation:

```
Conservative vs. Aggressive:

Aggressive (BAD):
  T=0:00  Utilization drops below threshold
  T=0:00  Kill 3 atoms immediately
  T=0:05  New requests arrive
  T=0:05  Kill atoms is now too aggressive
  T=0:05  Need to scale back up again
  Result: Thrashing, wasted resources

Conservative (GOOD):
  T=0:00  Utilization drops below threshold
  T=0:00  Kill 1 atom only
  T=0:10  Metrics stabilize, still below threshold
  T=0:10  Kill 1 more atom
  T=0:20  Metrics stabilize, still below threshold
  T=0:20  Kill 1 more atom
  Result: Smooth scale-down, no oscillation
```

#### Process

```
1. Select atom to kill
   - Choose NEWEST atom (preserve warm caches in older atoms)
   - Reason: Newer atoms are more likely to have cold data

2. Open circuit breaker
   - Force open for this specific atom
   - Load balancer stops routing new requests

3. Drain active connections
   - Wait for in-flight requests to complete
   - Maximum wait: 30 seconds

4. Remove container
   - docker stop {atom-id}
   - docker remove {atom-id}

5. Update state
   - Remove from in-memory registry
   - Update Prometheus metrics

6. Wait for stabilization
   - Before killing next atom (if needed)
   - Prevents oscillation
```

#### Timing

```
Scale-down is CONSERVATIVE:

T=0:00   CPU utilization below threshold
T=0:00   Kill decision for 1 atom
T=0:01   Circuit breaker opened for this atom
T=0:05   In-flight requests drained
T=0:05   Atom removed
T=0:10   Metrics re-evaluated
         If still below threshold, kill another atom
         Otherwise, stop and wait
```

### Protection Mechanisms

#### Budget Enforcement

```
spec:
  maxCpuToUse: 1.6 CPU
  maxMemToUse: 2 GB

Scale-up decision:
  if totalAllocatedCpu + containerCpu <= maxCpuToUse:
    spawnAtom()
  else:
    # Reached budget, cannot scale further
    log("CPU budget exhausted")
    # Next scale cycle, still under budget?
    # Scale-down may be needed
```

#### Minimum Atom Requirement

```
NEVER scale down to zero atoms:

if atomCount > spec.minAtoms:
  Can scale down

Result: Always maintain at least minAtoms
        Prevents total service unavailability
```

---

## Mechanism 2: Zero-Downtime Rolling Updates

### Trigger Condition

```
if spec.image != runningImage:
  initiateRollingUpdate()
```

### Rolling Update Process

#### Step 1: Spawn New Atom

```
1. Check if any atom already running new image
   if YES:
     use existing atom as template
   else:
     pull new image, create new atom

2. Mark new atom UNHEALTHY initially
   (warmup period while starting)

3. Add to state store
```

#### Step 2: Health Checks

```
1. Begin active health probing
   every 5 seconds: GET /health

2. Configurable warmup timeout
   (allow time for application startup)

3. Once 2xx response received:
   Mark new atom HEALTHY
```

#### Step 3: Transfer Traffic

```
1. Register new atom with load balancer

2. Open circuit breaker on OLD atom
   (prevents new traffic routing to it)

3. New traffic routes to new atom
```

#### Step 4: Drain Connections

```
1. Load balancer identifies active connections
   to old atom

2. Wait for connections to complete
   Maximum wait time: 3 seconds (configurable)

3. Gracefully close remaining connections
   Don't abruptly terminate
```

#### Step 5: Remove Old Atom

```
1. docker stop {old-atom-id}

2. docker remove {old-atom-id}

3. Update state store
```

#### Step 6: Repeat (If Needed)

```
if more atoms still running old image:
  goto Step 1
else:
  Rolling update complete!
```

### Zero-Downtime Guarantee

```
At no point is there ZERO capacity:

Timeline:
  T=0:00  Old: 5 atoms, New: 0 atoms
          │ → Spawning new atom
  
  T=0:05  Old: 5 atoms, New: 1 atom (warming up)
          │ → New atom passes health checks
  
  T=0:05  Old: 5 atoms, New: 1 atom (HEALTHY)
          │ → New traffic routes to new
  
  T=0:10  Old: 5 atoms, New: 1 atom
          │ → Old connections drained
          │ → Remove 1 old atom
  
  T=0:10  Old: 4 atoms, New: 1 atom
          │ → Repeat process
  ...
  T=2:00  Old: 0 atoms, New: 5 atoms
          └ → Update complete, full capacity maintained
```

### No Dropped Requests

```
Scenario: Request in-flight during update

1. Request arrives at load balancer (T=0:05)
2. Load balancer selects old atom (still healthy)
3. Request routed to old atom
4. Old atom processes request
5. At T=0:10, old atom marked for removal
   BUT connection still active
6. Orchestrator waits 3 seconds for request to complete
7. Request completes at T=0:12
8. Connection closed gracefully
9. Atom removed

Result: Request completed, no 503, no timeout
```

---

## Mechanism 3: Graceful Shutdown

### Trigger Events

Graceful shutdown is triggered by:

```
- SIGTERM (kill signal from container orchestrator)
- SIGINT (Ctrl+C from terminal)
- Explicit API request (if implemented)
```

### Graceful Shutdown Sequence

#### Phase 1: Stop Accepting New Connections

```
1. All circuit breakers opened
   (all backends configured to reject new traffic)

2. Load balancer stops accepting requests
   (returns HTTP 503 if needed)

3. Signal logged to stdout
   "Graceful shutdown initiated"
```

#### Phase 2: Drain Active Connections

```
1. Identify all active connections
   (tracked by load balancer)

2. Wait for connections to complete
   Maximum wait: 10 seconds

3. Monitor connection count
   - If reaches 0 before timeout: proceed
   - If timeout: force close (Phase 3)
```

#### Phase 3: Cleanup

```
1. Force close any remaining connections

2. Stop health checking goroutines

3. Stop scaling decision goroutine

4. Remove all atoms (clean shutdown)
   or leave running (for external cleanup)

5. Exit gracefully
   Exit code: 0 (success)
```

### Timing

```
T=0:00   SIGTERM received
T=0:00   Circuit breakers open
T=0:01   Active connections being drained
T=0:05   All connections drained
T=0:05   Cleanup phase
T=0:06   Process exits

Total time: ~6 seconds (under normal circumstances)
```

### No In-Flight Request Loss

```
Scenario: Long-running request during shutdown

T=0:00   SIGTERM received, shutdown begins
T=0:00   New requests rejected (circuit open)
T=0:02   Long-running request still in progress
T=0:08   Long-running request completes
T=0:09   Timeout exceeded, force close begins
T=0:09   BUT long request already complete!

Result: Minimal request loss
        (only those in-flight when timeout expires)
```

---

## Mechanism Integration Example

### Scenario: Production Deployment

```
Current State:
  - api-service: 5 atoms running v1.5
  - Load: 60% CPU utilization

Action: Deploy v2.0

T=0:00  Developer pushes v2.0 tag
        Orchestrator detects image change

T=0:01  Rolling update starts
        New atom with v2.0 spawned

T=0:06  New atom passes health checks
        Circuit breaker opens on 1st old atom

T=0:06  New traffic routes to v2.0 atom
        Load distributed to other v1.5 atoms
        API response times: normal

T=0:11  1st old atom removed
        5 atoms → 4 v1.5 + 1 v2.0

T=0:12  2nd new atom with v2.0 spawned

T=0:17  2nd new atom healthy
        2nd old v1.5 circuit opens

T=0:22  2nd old v1.5 atom removed
        5 atoms → 3 v1.5 + 2 v2.0

... (repeat for remaining atoms)

T=4:00  All 5 atoms running v2.0
        Rolling update complete

Metrics:
  - Downtime: 0 seconds
  - Dropped requests: 0
  - Max latency increase: <5%
```

---

## Performance Characteristics

### Scaling Latency

```
Scale-up (under load, utilization at 85%):
  Detect: 0-10 seconds (next evaluation cycle)
  Create: 1 second
  Health check: 0-5 seconds (warmup)
  Registered: 5-6 seconds total
  
Under-provisioning time: 5-10 seconds
(worst case)
```

### Update Duration

```
For 5 atoms being updated from v1 to v2:
  - Per atom: 5-10 seconds
  - Total: 25-50 seconds (sequential update)
  - Parallel: Could be faster, but conservative
    approach ensures safety
```

### Drain Time

```
Graceful connection drain:
  - 3 seconds for rolling update
  - 10 seconds for shutdown
  - Varies by request processing time
```

---

## Summary

| Mechanism | Key Feature | Benefit |
|-----------|------------|---------|
| **Auto-Scaling** | Utilization-based | Responsive to load changes |
| **Rolling Updates** | One atom at a time | Zero downtime guaranteed |
| **Graceful Shutdown** | 10-second drain period | No dropped requests |
| **Circuit Breaker** | Per-backend | Prevents cascading failures |

---

## Next Steps

- **[Section 5: Clone-Based Scaling →](../05-clone-based-scaling/README.md)**
- **[Section 6: Circuit Breaker →](../06-circuit-breaker/README.md)**
- **[Back to White Paper Index](../README.md)**

---

**Author:** Dale Tomson | **Date:** June 2026 | **License:** GNU AGPL v3.0
