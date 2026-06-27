# Section 3: System Architecture

**Component diagram and interaction model**

**Author:** Dale Tomson

---

## Overview

Atom Orchestrator consists of **five tightly integrated components**, all running within a **single process**:

```
┌─────────────────────────────────────────────────┐
│    ATOM ORCHESTRATOR (Single Go Binary)         │
├─────────────────────────────────────────────────┤
│                                                  │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ ORCHESTRATOR │  │ LOAD BALANCER│  ┌────────┐│
│  │   (Core)     │  │ (HTTP Rev    │  │HEALTH  ││
│  │              │  │  Proxy + CB) │  │CHECKER ││
│  │ - YAML spec  │  │              │  │(Active)││
│  │ - Metrics    │  │ - Round      │  │Probing ││
│  │ - Scaling    │  │   Robin      │  │        ││
│  │ - Updates    │  │ - Least      │  │        ││
│  │              │  │   Conn       │  │        ││
│  └──────┬───────┘  └──────┬───────┘  └───┬────┘│
│         │                  │               │     │
│         └──────────────────┼───────────────┘     │
│                            │                     │
│         ┌──────────────────┴────────────────┐    │
│         │                                   │    │
│  ┌──────┴──────┐              ┌────────────┴──┐ │
│  │  DOCKER API │              │ STATE STORE   │ │
│  │  (Runtime)  │              │ (In-Memory)   │ │
│  │             │              │               │ │
│  │ - Inspect   │              │ - Atom list   │ │
│  │ - Create    │              │ - Metrics     │ │
│  │ - Start     │              │ - CB state    │ │
│  │ - Stop      │              │ - Health      │ │
│  │ - Stats     │              │   status      │ │
│  └──────┬──────┘              └───────────────┘ │
│         │                                       │
│  ┌──────┴──────────────────────────────────┐    │
│  │                                          │    │
│  │    RUNNING CONTAINERS (Atoms)           │    │
│  │                                          │    │
│  └──────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

---

## Component 1: The Orchestrator Core

### Purpose
The Orchestrator Core is the **central control loop**. It reads element specifications from YAML files, maintains an in-memory registry of all atoms, and executes scaling decisions.

### Responsibilities

| Responsibility | Frequency | Details |
|----------------|-----------|---------|
| **Read YAML specifications** | On startup + file watch | Load element definitions |
| **Scaling decisions** | Every 10 seconds | Evaluate utilization, scale up/down |
| **Image change detection** | Every 10 seconds | Detect new image, trigger rolling update |
| **Metrics collection** | Every 10 seconds | Pull CPU/memory stats from Docker |
| **Health check coordination** | Continuous | Integrate health check results |
| **Circuit breaker state** | Continuous | Monitor per-backend failures |

### Key Algorithm: Scaling Decision

```
Every 10 seconds:
  1. Collect CPU and memory stats from all atoms
  2. Calculate aggregate utilization:
     totalActualCpu = sum(atom.cpuUsage for all atoms)
     totalAllocatedCpu = atomCount * spec.containerCpu
     cpuUtilization = totalActualCpu / totalAllocatedCpu
  
  3. If cpuUtilization >= spec.scaleAtCpuUsage:
       - If atomCount * containerCpu < maxCpuToUse:
           SpawnAtom()
       - Else: Do nothing (budget reached)
  
  4. If cpuUtilization <= spec.killAtCpuUsage AND atomCount > 1:
       KillNewestAtom()  # Preserve warm caches in older atoms
       Wait for metrics to stabilize
```

### State Management

**Critical Design Point:** The Orchestrator Core does **not maintain persistent state**.

All state is **derived from the Docker runtime**:

```
On Startup:
  1. List all running containers
  2. Filter by atom.orchestrator label
  3. Rebuild in-memory registry
  4. Resume scaling/updating operations

Result: Self-healing system with no etcd, no state database
```

### API Endpoints

The Orchestrator Core exposes:

```
GET /status              - Current orchestrator state
GET /elements            - List all elements and atoms
GET /elements/{name}     - Details for specific element
POST /elements/{name}/scale         - Manual scaling
POST /elements/{name}/update        - Trigger update
GET /metrics            - Prometheus-format metrics
```

---

## Component 2: The Load Balancer

### Purpose
The Load Balancer is an **HTTP reverse proxy** that:
- Routes requests to healthy backends
- Implements per-backend circuit breaker
- Provides two selection strategies
- Handles graceful connection draining

### Implementation

Built using Go's standard library:
```go
import "net/http/httputil"
```

### Selection Strategies

#### Round Robin
- Equal distribution across healthy backends
- Simple, predictable behavior
- Ideal for stateless workloads

#### Least Connections
- Routes to backend with fewest active requests
- Better for variable request processing times
- Prevents overloading slower backends

### Circuit Breaker Integration

```
Request arrives
    ↓
[Check Circuit Breaker State]
    ├─ CLOSED: Route to backend, count failures
    ├─ OPEN: Reject with HTTP 503
    └─ HALF-OPEN: Route to backend, but limited
    ↓
[Backend processes request]
    ↓
[Response received]
    ├─ Success: Decrement failure count / increment success count
    └─ Failure: Increment failure count
    ↓
[If failures exceed threshold: Open circuit]
[If successes exceed threshold in half-open: Close circuit]
```

### Load Balancer Algorithm

```
SelectBackend():
  1. Get list of all atoms for element
  2. Filter: atom.health == HEALTHY
  3. Filter: circuitBreaker[atom].state != OPEN
     (or if no healthy closed circuits, allow HALF-OPEN)
  
  4. If using Round Robin:
       Select next in rotation
  
  5. If using Least Connections:
       Select atom with lowest activeConnections
  
  6. If no backends available:
       Return HTTP 503 (Service Unavailable)
```

### Graceful Shutdown

On SIGTERM or SIGINT:
```
1. Open circuit breaker on all backends
   (stop new requests)
2. Wait up to 10 seconds for in-flight requests to complete
3. Forcefully close connections
4. Stop accepting new requests
```

---

## Component 3: The Health Checker

### Purpose
The Health Checker performs **active probing** of all backends to determine their health status.

### Behavior

| Aspect | Details |
|--------|---------|
| **Probe Interval** | 5 seconds |
| **Endpoint** | `/health` (configurable) |
| **Timeout** | 3 seconds per probe |
| **Acceptable Status Codes** | 200-299 (2xx) |
| **Failure on** | Non-2xx status or timeout |

### Health Check States

```
Atom State Machine:

┌─────────────┐
│  UNHEALTHY  │ (Container just started)
│             │ • No traffic routed
│  Warming up │ • Health checks running
│             │ • When first 2xx response: HEALTHY
└─────┬───────┘
      │
      └──→ ┌─────────────┐
           │  HEALTHY    │ (Responding normally)
           │             │ • Traffic routed normally
           │  In service │ • Continuous monitoring
           │             │ • On 3+ failed probes: UNHEALTHY
           └─────┬───────┘
                 │
                 └──→ ┌─────────────┐
                      │  UNHEALTHY  │ (Container failing)
                      │             │ • No new traffic
                      │ Recovering  │ • Circuit breaker may open
                      │             │ • Health checks continue
                      └─────────────┘
```

### Health Check Reliability

```
Scenario: Transient network blip

Request 1: Timeout (network glitch) → failureCount = 1
Request 2: 200 OK                    → failureCount = 0
Request 3: 200 OK                    → still healthy

Result: Single failure doesn't trigger unhealthy state
        Persistent failures do (threshold-based)
```

---

## Component 4: Docker API Integration

### Purpose
The Docker API Interface provides the **runtime abstraction layer** for all container operations.

### Operations

| Operation | Purpose | Used By |
|-----------|---------|---------|
| `Inspect` | Get container config | Scaling, updates |
| `Create` | Create new container | Scaling, rolling update |
| `Start` | Start stopped container | Initial deploy |
| `Stop` | Stop running container | Scale-down, shutdown |
| `Stats` | Get CPU/memory metrics | Scaling decisions |
| `List` | Find containers with label | State recovery |
| `Remove` | Delete container | Scale-down cleanup |

### Key Integration Points

#### Golden Container Creation

```
Element has 0 atoms:
  1. Pull image from registry
  2. docker create --name atom-{element}-0 \
       --label atom.orchestrator={element} \
       -m 256Mi --cpus 0.2 \
       {image}
  3. docker start atom-{element}-0
  4. Wait for health check to pass
  5. This becomes the template!
```

#### Clone-Based Spawning

```
Element needs more atoms:
  1. Identify healthy running atom (template)
  2. docker inspect {template-container}
     Extract: image, env, cmd, entrypoint, workdir, user, etc.
  3. docker create --name atom-{element}-N \
       --label atom.orchestrator={element} \
       -m 256Mi --cpus 0.2 \
       [all extracted config] \
       {image}
  4. docker start atom-{element}-N
  5. Register with load balancer
```

### Error Handling

```
Scenario: docker pull times out

Action:
  1. Retry up to 3 times with exponential backoff
  2. If all retries fail, log error and skip spawn
  3. Continue trying in next 10-second cycle
  4. Do not crash orchestrator
  5. Do not cascade error to load balancer

Result: Orchestrator is resilient to transient Docker failures
```

---

## Component 5: In-Memory State Store

### Purpose
The State Store holds **ephemeral state** about atoms, metrics, and circuit breaker status. It is **rebuilt from Docker on startup**.

### State Stored

```
Elements:
  ├─ name: string
  ├─ image: string
  ├─ containerCpu: float
  ├─ containerMem: int
  ├─ minAtoms: int
  ├─ maxAtoms: int
  ├─ scaleAtCpuUsage: float
  ├─ killAtCpuUsage: float
  └─ atoms: []Atom

Atoms:
  ├─ id: string
  ├─ name: string
  ├─ createdAt: time
  ├─ cpuUsage: float
  ├─ memUsage: int
  ├─ healthStatus: enum (healthy, unhealthy)
  ├─ circuitBreakerState: enum (closed, open, half-open)
  ├─ failureCount: int
  ├─ successCount: int
  └─ lastHealthCheck: time

Metrics:
  ├─ totalRequests: int
  ├─ totalErrors: int
  ├─ avgLatency: float
  └─ activeConnections: int per atom
```

### Consistency Model

**Write Pattern:**
```
Docker Event
  ↓
Update State Store
  ↓
Sync to Prometheus metrics
```

**Read Pattern:**
```
Scaling decision
  ↓
Read from State Store
  ↓
No consistency issues (single goroutine for updates)
```

---

## Interaction: A Typical Request

```
1. Client Request Arrives
   │
   ├─ → Load Balancer
   │
   ├─ Check circuit breaker state (CLOSED)
   │
   ├─ Consult state store for healthy atoms
   │
   ├─ Select backend using Round Robin
   │
   ├─ Forward request to backend
   │
   2. Backend Processing
   │
   ├─ Container processes request
   │
   ├─ Returns 200 OK
   │
   ├─ Load Balancer receives response
   │
   ├─ Decrement failure counter (or stay at 0)
   │
   ├─ Return response to client
   │
   3. Metrics Updated
   │
   ├─ Request count incremented
   │
   ├─ Latency recorded
   │
   └─ Health status remains HEALTHY
```

---

## Scaling Decision: A 10-Second Cycle

```
Orchestrator Core:

Every 10 seconds:
  1. Collect metrics from Docker Stats API
     cpuUsage = sum(container.CpuStats)
     memUsage = sum(container.MemStats)
  
  2. Calculate utilization
     cpuUtil = cpuUsage / totalAllocatedCpu
  
  3. If cpuUtil >= 75% (scaleAtCpuUsage):
       - Check budget: currentCpu < maxCpuToUse?
       - If yes: Call Health Checker for template
       - Get healthy atom, call Docker API Create
       - Load Balancer adds to rotation once healthy
  
  4. If cpuUtil <= 25% (killAtCpuUsage) AND atoms > 1:
       - Select newest atom
       - Call Load Balancer to drain connections
       - Call Docker API Stop/Remove
       - Update state store

  5. Update Prometheus metrics
  
  6. Check for image changes
     If spec.image != running container image:
       - Trigger rolling update (see Core Mechanisms)
```

---

## State Recovery: Orchestrator Restart

```
Orchestrator crashes at 2:00 PM

At 2:05 PM, orchestrator restarts:

1. Read atom.orchestrator labeled containers
   Found: 5 running atoms
  
2. For each atom:
     - Inspect container config
     - Restore to state store
     - Get last known health status (assume healthy)
  
3. Resume scaling decisions
   - Recompute utilization based on Docker stats
   - If scale-up or scale-down needed, resume
  
4. Resume health checks
   - All atoms begin health probing immediately
  
Result: System recovers within seconds, no manual intervention
```

---

## Concurrency Model

### Single Control Loop

```
Main goroutine:
  - Read YAML on startup
  - Start child goroutines
  
Child goroutines (one per component):
  - Health Checker: Probes at 5-second intervals
  - Scaling Loop: Decisions at 10-second intervals
  - Metrics Collector: Updates at 10-second intervals
  
Synchronization:
  - State store uses mutex for concurrent access
  - Channels used for signaling (graceful shutdown)
  - No distributed locks needed (single process)
```

### Ordering Guarantees

```
Event: Scale-up decision made

Timeline:
  1. Scaling loop creates new container (Docker API)
  2. Container started by Docker
  3. State store updated with new atom
  4. Load Balancer learns about new atom
  5. Health checks begin
  6. On first 2xx response: health = HEALTHY
  7. Load Balancer includes in rotation

Result: Strict ordering, no race conditions
```

---

## Summary

The architecture's key strengths:

✅ **Single process:** No IPC, no distributed consensus needed  
✅ **Self-healing:** State derived from Docker, not persistent store  
✅ **Resilient:** Circuit breakers prevent cascading failures  
✅ **Responsive:** 10-second scaling cycle vs. 30-60s for Kubernetes  
✅ **Observable:** All state visible in one place  

---

## Next Steps

- **[Section 4: Core Mechanisms →](../04-core-mechanisms/README.md)**
- **[Section 5: Clone-Based Scaling →](../05-clone-based-scaling/README.md)**
- **[Back to White Paper Index](../README.md)**

---

**Author:** Dale Tomson | **Date:** June 2026 | **License:** GNU AGPL v3.0
