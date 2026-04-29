# Field Guide Relay Proxy Patterns

Extracted from: `LDFieldGuide/LaunchDarkly Field Guide.pdf` (242 pages)
Source: https://launchdarkly-labs.github.io/ps-flag-book/print.html

## Key Sections and Page References

| Pages | Section | What it covers |
|-------|---------|----------------|
| 24-27 | SDK Architecture / Network Requirements | SDK connection model, network endpoints, first Relay mention |
| 35-45 | Init/Config, Resilient SDK, Fallback Values | SDK config for Relay, resilience patterns, caching layer |
| 48-55 | Projects and Environments | Environment design impact on Relay connections and memory |
| 78-83 | Relay Proxy: Overview, When to Use, When NOT to Use | Core decision criteria, use cases, anti-patterns |
| 84-88 | Architecture, Modes (Proxy, Daemon, Offline) | Three operating modes with selection criteria |
| 89-97 | Sizing and Scaling | Resource requirements, sizing guidelines, scaling strategies |
| 98-110 | Deploying, Configuration, High Availability | Deployment options, config settings, HA patterns |
| 111-118 | Monitoring, Alerting | Metrics, endpoints, alerting thresholds |
| 119-126 | Caching | Cache layers, TTL configuration, persistent store setup |
| 127-135 | Offline Mode / Air-Gapped | Offline patterns, data sync, air-gapped considerations |
| 143-144 | Reverse Proxies | Reverse proxy config table for nginx/HAProxy/ALB |
| 145-148 | AutoConfig | Auto-configuration with custom roles and environment policies |

---

## Decision Criteria (for rules.py)

### Reasons FOR Relay
- Multiple server-side SDK instances (reduces outbound connections)
- Network restrictions (firewalls, on-prem, limited internet)
- Air-gapped / offline environments
- Client-side proxy mode (ad-blockers, hiding LD endpoints)
- Security/compliance (traffic inspection, TLS termination, restricted SaaS)
- High availability caching layer (persistent store survives restarts)

### Reasons AGAINST Relay (anti-patterns)
- Fewer than 10 server-side SDK instances (connection savings minimal)
- SDKs can reach LD directly with no restrictions
- Client-side SDKs only (CDN-backed endpoints handle scale already)
- Goal is latency reduction (Relay adds a hop, LD CDN may be faster)
- Goal is redundancy only (SDKs already cache in memory)

### Mode Selection
- **Proxy mode (default):** Standard deployment, browser apps present, centralized preference
- **Daemon mode:** Large server-side fleet + existing shared data store (Redis/DynamoDB/Consul), SDKs read store directly, no streaming connections to Relay
- **Offline mode:** Air-gapped, no internet, compliance prohibits external SaaS

### Daemon Mode Triggers (expanded)
- Large number of server-side SDK instances AND shared data store exists
- PHP apps (cannot maintain streaming connections)
- Embedded/sidecar preference where SDKs read from store directly

---

## Sizing Numbers (verified from Field Guide)

| Metric | Value | Source |
|--------|-------|--------|
| Tested instance | m4.xlarge (4 vCPU, 16 GiB RAM) | Official |
| Minimum instances | 3 across 2+ AZs | Official |
| Idle memory | ~11 MiB | Official |
| Connection provisioning | 2x expected | Official |
| Client-side streaming | "not efficient" at scale | Official |
| Default cache TTL | 30s | Official |
| Recommended cache TTL | -1 (infinite) | Official |
| Default port | 8030 | Official |
| Heartbeat interval | 3 minutes | Official |

### Scaling Signals
- CPU sustained > 70% for 10 min: scale horizontally
- Memory growth > 20% in 1 hour: investigate segments/environments
- Connection count drops > 50%: investigate Relay health
- Event forwarding delayed > 5 min: check backlog
- Any environment disconnected > 5 min: investigate connectivity
- Status endpoint unreachable: critical, process may be down

### Key Resources (priority order)
1. Network bandwidth (most important)
2. Memory (grows with environments, flags, segments, connections)
3. CPU (connection management, serialization)
4. File descriptors (one per connected SDK, check OS limits)

---

## Reverse Proxy Configuration Checklist

From Field Guide pages 143-144. This is a ready-made checklist for Module 3.

| Setting | Configuration | Notes |
|---------|--------------|-------|
| Response buffering for SSE | Disable | Relay sends `X-Accel-Response-Buffering: no` for nginx automatically |
| Forced gzip for SSE | Disable if proxy not SSE-aware | Gzip buffers responses |
| Response timeout / max connection time for SSE | Minimum 10 minutes | Avoid too-long timeouts (waste resources on disconnected clients) |
| Upstream / proxy read timeout | 5 minutes | In nginx: `proxy_read_timeout` |
| CORS headers | Restrict to your domains only | Required when using browser SDK endpoints |
| Status endpoint | Restrict from public access | Internal monitoring only |
| Metrics / Prometheus port | Restrict access | Internal monitoring only |

---

## Load Balancer Requirements

From Field Guide pages 106-110.

- Support for SSE / long-lived HTTP connections
- Session affinity NOT required (SDKs handle reconnection)
- Health checks against `/status` endpoint
- Response timeout for SSE: minimum 10 minutes
- Upstream read timeout: 5 minutes
- Disable response buffering for SSE endpoints
- Disable forced gzip if not SSE-aware

---

## Deployment Patterns

### By Platform (pages 98-100)
- **Docker:** Official image `launchdarkly/ld-relay`, env vars for SDK keys
- **Kubernetes:** Deployment (centralized, fewer instances) or DaemonSet (per-node, lowest latency). Readiness probes on `/status`. PodDisruptionBudgets. Resource requests/limits.
- **ECS/Fargate:** ECS service with auto-scaling on CPU or connection count
- **VM/bare metal:** Binary + systemd

### Rolling Deployment Steps (pages 106-110)
1. Deploy new instances alongside existing
2. Wait for new instances to initialize and sync flag data
3. Shift traffic via load balancer
4. Drain connections from old instances
5. Terminate old instances

SDKs handle reconnection automatically. Brief disconnections during deployment are expected and safe.

---

## Monitoring Reference

### Endpoints
- `/status` - environment connection state, data store state, last sync time
- Prometheus metrics endpoint

### Key Metrics
| Metric | Description |
|--------|-------------|
| connections | Active SDK connections |
| new_connections | Rate of new connections |
| requests | Request count by endpoint |
| events_posted | Event batches forwarded |
| event_bytes_posted | Event data volume forwarded |
| stream_connections | Active streaming connections |

### Alerting Thresholds
| Condition | Severity | Action |
|-----------|----------|--------|
| Environment disconnected > 5 min | High | Investigate Relay-to-LD connectivity |
| Connection count drops > 50% | High | Investigate Relay health and LB |
| CPU sustained > 70% for 10 min | Medium | Scale horizontally |
| Memory growth > 20% in 1 hour | Medium | Investigate environment/segment data |
| Event forwarding delayed > 5 min | Medium | Check Relay backlog |
| Status endpoint unreachable | Critical | Relay process may be down |

---

## AutoConfig Patterns

### Custom Role Policies (pages 145-148)

All instances (admin):
```json
[{"effect": "allow", "resources": ["relay-proxy-config/*"], "actions": ["*"]}]
```

Production in all projects:
```json
[{"actions": ["*"], "effect": "allow", "resources": ["proj/*:env/production"]}]
```

Production in specific project:
```json
[{"actions": ["*"], "effect": "allow", "resources": ["proj/foo:env/production"]}]
```

All environments in projects starting with prefix:
```json
[{"actions": ["*"], "effect": "allow", "resources": ["proj/foo-*:env/*"]}]
```

Exclude production and federal-tagged projects:
```json
[
  {"actions": ["*"], "effect": "allow", "resources": ["proj/*:env/*"]},
  {"actions": ["*"], "effect": "deny", "resources": ["proj/*:env/production", "proj/*;federal:env/*"]}
]
```

---

## Offline / Air-Gapped Patterns

### Data Sync Options
1. **Export/Import:** LD API export from connected env, transfer via approved mechanism, import to store
2. **File-based data source:** Relay reads JSON flag config from local file, watches for changes

### Operational Concerns
- Events: cannot forward to LD. Discard, store locally, or batch-transfer.
- Flag updates: must build a manual transfer process
- SDK keys: still required, generate in LD and include in deployment config
- Time sync: critical for scheduled flag changes
- Version tracking: know which flag data version is in each air-gapped env

---

## Caching Layers

### In-Memory Cache (always on)
- Populated from streaming connection
- Updated in real time
- Fastest path for serving SDKs
- Lost on process restart

### Persistent Store Cache (optional, recommended)
- Redis (most common), DynamoDB, or Consul
- Write-through from streaming connection
- Fallback on startup or streaming failure

### Cache TTL Behavior
- **Default (30s):** Relay re-reads from store every 30s. Adds read load. Safety net.
- **Recommended (-1/infinite):** Relay reads store only on startup. After init, trusts streaming. Reduces store load.

---

## SDK Configuration for Relay

All three endpoints must be pointed to Relay. If only some are set, the SDK connects directly to LD for the others.

```python
config = Config(
    sdk_key="sdk-key-123",
    stream_uri="https://relay.internal.example.com",
    base_uri="https://relay.internal.example.com",
    events_uri="https://relay.internal.example.com"
)
```

---

## Environment Design Impact

Each environment configured in Relay creates a separate streaming connection to LD. More environments = more connections + more memory.

Best practices:
- Only include environments that SDKs behind the proxy actually need
- Use AutoConfig to dynamically manage environments
- Monitor resource usage as environments are added
