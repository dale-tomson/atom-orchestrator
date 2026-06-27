# Section 9: Security Model

**Defense in depth through minimalism**

**Author:** Dale Tomson

---

## The Security of Not Managing Secrets

### The Core Principle

The most significant security feature of Atom Orchestrator is what it does not do. By design:

It doesn't store secrets. It doesn't store environment variables. It doesn't manage configuration. It doesn't encrypt/decrypt data. It doesn't maintain persistent state.

By eliminating these responsibilities, the orchestrator eliminates an entire class of security vulnerabilities.

### Traditional Orchestrator Vulnerabilities

Kubernetes stores secrets in etcd, and there are a lot of ways that can go wrong. The API can expose secrets over the network. Secrets sit in memory as plaintext after decryption. The etcd database needs encryption at rest. Backups need to be encrypted. Audit logs might contain secrets. Network traffic needs TLS.

That's at least 7 separate exposure points where secrets can leak.

### Atom Orchestrator Approach

Atom doesn't manage secrets at all. There's no secret storage. No secret API. No secret encryption logic. No backups of secrets. No audit logs containing secrets. Secrets only exist in the container runtime, managed by Docker.

Secret-related attack surface: zero.

---

## Configuration Immutability

### How It Works

Configuration is **embedded in the container image** by the deployment pipeline:

```bash
# CI/CD Pipeline creates container with config baked in
docker run \
  --name api-template \
  -e DB_URL="postgres://prod.db" \
  -e API_KEY="secret123" \
  myimage:v1

# Orchestrator clones from this container
# Configuration is replicated exactly as-is
# Orchestrator never sees, stores, or modifies it
```

### Security Implications

No injection attacks: The orchestrator can't be exploited to change environment variables or secrets because it doesn't manage them. There's no API endpoint to modify secrets. Changes require a code rebuild, which is explicit and trackable.

Audit trail in CI/CD: All configuration changes live in source control. Git history shows who changed what and when. Code review happens before deployment. Deployment logs show exactly what went out.

No accidental exposure: Secrets aren't logged by the orchestrator. They're not exposed via any API. They don't appear in orchestrator state dumps.

---

## Docker Socket Access

### The Necessary Trade-Off

Atom Orchestrator requires access to the Docker Engine API, typically via `/var/run/docker.sock`.

```
Important: This is equivalent to root access on the host
```

### Mitigations

**Run as non-root user.** Create a dedicated user and add it to the docker group.

**Use Docker rootless mode.** Docker can run without root privileges, which significantly reduces the attack surface.

**Restrict file system access.** Use AppArmor or SELinux profiles to limit what the orchestrator can access. It should only have access to docker.sock, the configuration file, and a few system files.

**Never expose the API directly.** Always run behind a reverse proxy (nginx, Caddy, Traefik) with TLS and authentication.

**Network isolation.** The orchestrator shouldn't be accessible from untrusted networks. Run it on an internal network. If you need remote access, use a VPN or SSH tunnel.

---

## Network Isolation

### Docker Networking

Atom Orchestrator uses Docker's default bridge networking:

```
Each atom gets:
  ✓ Internal IP on bridge network (172.17.0.x)
  ✓ Isolated from host network
  ✓ Port mapping from host port to container port

Access patterns:
  External → Host port 8080
          ↓
  Atom port 8080 (mapped internally)
  
Within containers:
  Atom 1 → Atom 2: Via load balancer (localhost:8080)
                    NOT direct IP communication
```

### Limitations

```
No network policies:
  ✗ Can't restrict container-to-container traffic
  ✗ Can't enforce ingress/egress rules
  
Solution: External tools
  - Use host iptables for port restrictions
  - Use firewall for network segmentation
  - Consider service mesh if needed (external)
```

---

## Resource Denial-of-Service Protection

### Hard Resource Caps

The YAML specification includes hard caps on resource consumption:

```yaml
elements:
  - name: api-service
    maxCpuToUse: 1.6 CPU
    maxMemToUse: 2 GB
```

### Behavior

Even if a buggy or malicious workload tries to consume unlimited resources, it hits the configured cap. Say an application has a memory leak. It starts at 100 MB and grows: 200 MB, 300 MB, 400 MB. Eventually it reaches the configured maxMemToUse (2 GB). At that point:

No new atoms are spawned because the budget is exhausted. The existing container keeps running but can't grow beyond the limit. Other services are protected. The host system won't experience out-of-memory crashes.

### Protection

You're protected from memory leaks cascading to crash the host, CPU-hogging loops consuming all cores, and disk I/O exhaustion. You're not protected if an application exhauses resources within its own container—that's the Docker runtime's job to handle.

---

## Secrets Management (Best Practices)

### Recommended Approach

```
┌──────────────────────────────────────────────┐
│           CI/CD Pipeline (GitHub Actions)    │
├──────────────────────────────────────────────┤
│ 1. Retrieve secrets from vault               │
│    (GitHub Secrets, HashiCorp Vault, etc.)   │
│ 2. Build container image with secrets baked │
│ 3. Push image to registry                    │
│ 4. Trigger rolling update in Atom            │
└──────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────┐
│      Container Image (at rest in registry)   │
│  - Secret built into image                   │
│  - Only accessible to image owner            │
│  - Can be signed/encrypted                   │
└──────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────┐
│     Atom Orchestrator (running)              │
│  - Does NOT see secrets                      │
│  - Clones container verbatim                 │
│  - Secret stays in container runtime         │
└──────────────────────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────┐
│      Running Container                       │
│  - Secret available to application           │
│  - Stays within container namespace          │
│  - Not exposed to orchestrator               │
└──────────────────────────────────────────────┘
```

### Alternative: External Secret Manager

```
If CI/CD pipeline doesn't support secret injection:

1. Application reads secrets from:
   - HashiCorp Vault
   - AWS Secrets Manager
   - Azure Key Vault
   - GCP Secret Manager

2. At container startup:
   - App authenticates to secret manager
   - App retrieves secrets
   - App loads into memory

3. Orchestrator:
   - Never sees the secrets
   - Just manages container lifecycle
```

---

## Compliance Considerations

### What Atom Doesn't Provide

Atom was not designed for regulated environments. It doesn't provide audit logging by design. There's no RBAC (role-based access control). No encryption key management. No compliance reporting features.

If you're working with HIPAA, SOC2 Type II, PCI-DSS, or GDPR requirements, you need:

For audit logging: Use an external logging aggregator outside the orchestrator.
For encryption: Set up TLS at the reverse proxy layer.
For RBAC: Use Kubernetes with compliance operators, or managed services like GKE, EKS, or AKS.
For compliance reporting: Use a specialized compliance platform.

**Recommendation:** Atom is not suitable for regulated environments. If compliance is the primary concern, use Kubernetes with compliance operators or purpose-built compliance platforms.

---

## Security Summary

Atom's main security strength is simplicity. There's no secret storage to breach, no complex authentication to misconfigure, no encryption keys to lose, no persistent state to be compromised. 

The tradeoff is that you're responsible for the Docker socket access. Run as a non-root user, use rootless Docker if you can, restrict network access, and always use a reverse proxy. Don't expose the API directly to the internet.

Atom is best for:
- Small teams
- Internal deployments
- Edge devices
- Development/testing environments
- Single-operator systems

Atom is not for:
- Enterprise environments
- Regulated industries
- Multi-tenant systems
- Highly sensitive data
- Systems exposed to the public internet without a proxy layer  

---

## Production Deployment Checklist

Before running Atom in production:

**Security hardening:**
Run the orchestrator as a non-root user. Use Docker rootless mode if available on your platform. Apply AppArmor or SELinux profiles if you want defense in depth. Put a reverse proxy (nginx or Caddy) in front for TLS and basic authentication. Restrict network access so only your VPN or internal network can reach the orchestrator. Enable Docker daemon logging and check logs regularly for suspicious activity. Keep Docker and container images up to date.

**Network security:**
Use firewall rules to open only the ports you need. Don't expose the orchestrator API to the public internet. Put everything behind a reverse proxy with TLS termination. Consider basic authentication or OAuth on the reverse proxy. Run the orchestrator on an internal network, separate from publicly routable infrastructure.

**Data security:**
Manage secrets outside the orchestrator, in your CI/CD pipeline. Never log secrets or include them in debug output. Scan container images for vulnerabilities before deploying. Keep base images up to date. Run regular backups of important containers (not just images).

---

## Next Steps

- **[Section 10: Use Cases →](../10-use-cases/README.md)**
- **[Section 11: Roadmap →](../11-roadmap/README.md)**
- **[Back to White Paper Index](../README.md)**

---

**Author:** Dale Tomson | **Date:** June 2026 | **License:** GNU AGPL v3.0
