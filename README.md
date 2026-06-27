# Atom Orchestrator

<div style="background-color: #ffffff; padding: 30px 0; text-align: center; margin-bottom: 30px;">
  <img src="logo.svg" alt="Atom Orchestrator" width="200" />
</div>

A lightweight, clone-based container orchestrator for minimal-infrastructure environments.

## Current Status

**Design Phase Complete** — Comprehensive white paper (v1.0, June 28, 2026)  
**Development Begins:** July 2026

> The complete design specification is available on the [`white-paper`](https://github.com/atom-orchestrator/atom/tree/white-paper) branch.

## Read the White Paper

The full design specification is available here:

- **[White Paper Overview](https://github.com/atom-orchestrator/atom/tree/white-paper)** — Complete specification with all sections
- **[Problem Statement](https://github.com/atom-orchestrator/atom/tree/white-paper/white-paper/01-problem-with-modern-orchestration)** — Why Kubernetes is overkill
- **[System Architecture](https://github.com/atom-orchestrator/atom/tree/white-paper/white-paper/03-system-architecture)** — Core components and design
- **[Performance Benchmarks](https://github.com/atom-orchestrator/atom/tree/white-paper/white-paper/08-performance)** — Spawn latency, throughput, scaling limits
- **[Use Cases](https://github.com/atom-orchestrator/atom/tree/white-paper/white-paper/10-use-cases)** — When to choose Atom
- **[Implementation Roadmap](https://github.com/atom-orchestrator/atom/tree/white-paper/white-paper/11-roadmap)** — Development timeline

## Key Principles

1. **No configuration management** — Secrets embedded in containers by deployment pipeline
2. **Running container is source of truth** — Eliminates registry latency
3. **Fail fast, recover automatically** — Circuit breaker pattern for resilience
4. **Simplicity is a feature** — Single Go binary, no extensions, single developer comprehensible

## Target Specs

- **Binary size:** ~15 MB
- **Overhead:** 128 MB RAM hard cap
- **Spawn latency:** 0.5-2 seconds (clone-based)
- **Throughput:** 10,000-20,000 req/s per core
- **Minimum hardware:** 2 CPU / 4 GB RAM

## License

GNU Affero General Public License v3.0 with supplementary attribution terms. See [LICENSE](/LICENSE) for details.

---

**Project Status:** Design specification complete (v1.0, June 28, 2026)  
**Development Begins:** July 2026 (week 2-3)  
**Language:** Go  
**Author:** Dale Tomson
