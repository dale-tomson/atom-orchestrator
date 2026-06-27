# Section 10: Ideal Use Cases and Anti-Patterns

**Where Atom Orchestrator succeeds and where it should not be used**

**Author:** Dale Tomson

---

## Ideal Use Cases

### 1. Edge Computing and IoT Gateways

**The Problem:**
Edge devices (Raspberry Pi, NVIDIA Jetson) run on ARM boards with 2-4 GB of RAM. Kubernetes is physically infeasible.

**How Atom Solves It:**

```
50-150 MB footprint on constrained devices. Clone-based scaling works offline. No registry dependency (cache local images). Auto-scaling within device limits. Circuit breaker prevents cascading failures. Single binary: easy to deploy to ARM/edge.

Example:
  Device: Raspberry Pi 4
  RAM: 4 GB
  Available for workloads with Atom: ~3.85 GB
  Max atoms: 12-15

  Available for workloads with K3s: ~3.5 GB
  Overhead increase: 350 MB
```

Ideal for IoT telemetry aggregation, edge video processing, 5G cellular gateways, industrial edge computing, and vehicle edge nodes.

### 2. Development and Staging Environments

**The Problem:**
Developers need environments that mirror production behavior without production complexity. Docker Compose lacks scaling and health checks.

**How Atom Solves It:**

```
Laptop-local orchestration with prod-like features. Same YAML spec as production. Auto-scaling works on laptop. Circuit breaker teaches resilience patterns. 5-minute setup (no minikube, kind, or Docker Desktop K8s). One developer can operate it alone.

Developer Workflow:
  1. Create element in atom.yaml
  2. Run ./atom -config atom.yaml
  3. Test auto-scaling behavior
  4. Verify circuit breaker behavior
  5. Push same config to production
```

Ideal for local development with auto-scaling, staging environment simulation, testing scaling policies, load testing setup, and training and education.

### 3. Small SaaS and Single-Server Deployments

**The Problem:**
Bootstrapped SaaS running on a single $20/month VPS doesn't need a 3-node Kubernetes cluster but needs auto-scaling and high availability.

**How Atom Solves It:**

```
Horizontal scaling within single server. Automatic recovery from container crashes. Zero-downtime deployments. No complex infrastructure. Single binary deployment. Operational simplicity (one person can manage).

Example SaaS Stack:
  1 VPS (2 CPU / 4 GB)
  ├─ Atom Orchestrator (150 MB)
  ├─ API service (3 atoms × 256 MB = 768 MB)
  ├─ Worker service (2 atoms × 256 MB = 512 MB)
  ├─ Cache (512 MB)
  └─ Available buffer: ~1.5 GB

Auto-scaling behavior:
  At 75% CPU utilization: Add new atom
  At 25% CPU utilization: Remove atom
  During traffic spike: New capacity in <2 seconds
```

Ideal for early-stage startups, bootstrapped SaaS, indie hacker projects, freelancer services, and internal company tools.

### 4. CI/CD Build Agents and Ephemeral Workloads

**The Problem:**
CI/CD pipelines that spawn containers for testing/building benefit from fast scaling, but image-pull adds 30-120 seconds per agent.

**How Atom Solves It:**

```
Clone-based scaling: 0.5-2 seconds per agent. No registry dependency during scaling. Build queue response time: sub-second. Automatic cleanup (scale-down as queue empties). Zero-downtime deployment of new agent versions.

Workflow:
  Build Queue = 10 jobs waiting
  
  T=0:00   Utilization spike detected
  T=0:01   Clone 5 new agents (0.5s each = 2.5s total)
  T=0:02   Agents ready to process jobs
  
  Traditional (image-pull):
  T=0:00   Utilization spike detected
  T=0:30   First new agent ready
           10 jobs still waiting
           Build latency: 30+ seconds
```

Ideal for CI/CD worker nodes, test runners with parallel execution, build agents, batch job processing, and data pipeline workers.

### 5. Legacy Application Modernization

**The Problem:**
Organizations with legacy monoliths wrapped in containers want auto-scaling and health checking without rewriting code or learning Kubernetes.

**How Atom Solves It:**

```
✓ Orchestrator treats container as black box
✓ Works with any application (no special configuration needed)
✓ No code changes required
✓ Team doesn't need to learn Kubernetes
✓ Minimal operational overhead

Workflow:
  1. Team has legacy Java monolith
  2. Wrap in Docker container
  3. Deploy to Atom (no code changes)
  4. Get auto-scaling, health checks, rolling updates
  5. No new infrastructure complexity
```

Ideal for legacy monolith modernization, migration from VM deployments, application lift-and-shift, gradual modernization paths, and quick wins before full Kubernetes migration.

---

## Anti-Patterns: Where NOT to Use Atom

### 1. High-Availability Production Systems

Don't use Atom when requiring 99.99%+ uptime, multi-zone redundancy, automatic failover across regions, or ability to handle data center outages. A single-node architecture means a single point of failure—if the server crashes, the entire service goes down.

Use Kubernetes, Nomad, or managed container services instead.

Example: Production payment processing requires 99.99% uptime SLA (52 minutes downtime/year). If your server crashes, everything stops. Atom can't guarantee that level of availability. You need multi-zone Kubernetes.

### 2. Stateful Applications (Databases, Caches)

Don't use Atom for databases, message queues, distributed caches, or anything requiring persistent storage. When you scale up with Atom, new containers have zero data—only the old containers have it, and scaling becomes data loss.

Use Kubernetes StatefulSets, managed database services, or dedicated orchestrators.

Example: Scaling PostgreSQL from 3 to 5 replicas with Atom means the new replicas are empty containers with no data. The data is stuck in the old containers. Kubernetes StatefulSets handle this correctly with persistent volumes and data replication.

### 3. Multi-Region or Multi-Cloud Deployments

Don't use Atom for multi-region deployments, multi-cloud strategy, disaster recovery across geographies, or global load balancing. Atom is single-node by design and can't coordinate across regions.

Use Kubernetes, Nomad, or cloud provider solutions.

Example: SaaS deployed in US, EU, and APAC regions. Atom can run in each region, but can't share state, can't load-balance traffic globally, and can't replicate data across regions. Kubernetes Federation handles multi-region natively.

### 4. Regulated Environments (HIPAA, SOC2, PCI-DSS)

Don't use Atom for compliance-heavy environments. HIPAA, SOC2, PCI-DSS, and GDPR all require audit logging, fine-grained access controls, and encryption—none of which Atom provides. It's a design decision to keep the system simple, but it means Atom is incompatible with regulated workloads.

Use Kubernetes with compliance operators or specialized healthcare/fintech platforms.

Example: A healthcare app handling PHI (Protected Health Info) requires encryption in transit and at rest, RBAC, audit logging, and vulnerability scanning. Atom provides none of these. You need to use Kubernetes with compliance operators or a specialized healthcare platform.

### 5. Complex Multi-Microservice Architecture

Don't use Atom for 20+ microservices with complex service-to-service communication, service mesh requirements, distributed tracing, or advanced traffic policies. A single-node design can't manage that topology. The HTTP load balancer is too simple, and there's no service mesh or distributed tracing.

Use Kubernetes with a service mesh (Istio, Linkerd) or Nomad with Consul.

For most projects, you won't need this complexity anyway. If you do, Atom isn't the right tool.

---

## Choosing the Right Tool

Here's the basic decision flow:

**Do you need container orchestration?** If not, use Docker Compose or manage containers manually.

**How many nodes?** If just one, continue. If 2-5 nodes, consider Nomad or Docker Swarm. If 5+, you probably want Kubernetes.

**Do you need persistent storage?** Stateful applications need Kubernetes or Nomad, not Atom.

**How much operational complexity can you tolerate?** If you want minimal complexity and your app is stateless, Atom might work. If you need enterprise features, go to Kubernetes.

**Is it mission-critical?** If you need 99.99% uptime, you need redundancy that Atom can't provide. Use Kubernetes or Nomad.

**What's your team size and skill level?** If you're one person running a small service, Atom could work. If you have a large team, Kubernetes is worth the investment.

---

## Quick Comparison

Here's a rough sense of which tool fits which scenario:

| Use Case | Atom | Swarm | Nomad | Kubernetes |
|----------|------|-------|-------|-----|
| Laptop development | Good | OK | OK | Overkill |
| Single VPS | Good | Good | OK | Overkill |
| Edge/IoT | Good | OK | Good | Not possible |
| Small SaaS | Good | Good | Good | OK |
| Startup MVP | Good | Good | Good | OK |
| CI/CD agents | Good | Good | Good | Good |
| Scale-up ready | Not ideal | Good | Good | Best |
| High availability | No | Maybe | Yes | Yes |
| Stateful apps | No | No | Yes | Yes |
| Multi-region | No | No | Yes | Yes |
| Compliance heavy | No | No | Maybe | Yes |

---

## Summary

Atom works best for simple, single-node deployments where operational simplicity matters more than enterprise features. It's perfect for bootstrapped SaaS, edge devices, development environments, and teams too small to justify a full Kubernetes operation.

It's not for mission-critical systems that need global redundancy, stateful workloads like databases, or heavily regulated environments. If you have those requirements, spend the time to learn Kubernetes—it's worth it.

For everyone else, Atom is worth considering. If your use case fits the profile above, it'll save you months of operational overhead.  

---

## Next Steps

- **[Section 11: Roadmap →](../11-roadmap/README.md)**
- **[Back to White Paper Index](../README.md)**

---

**Author:** Dale Tomson | **Date:** June 2026 | **License:** GNU AGPL v3.0
