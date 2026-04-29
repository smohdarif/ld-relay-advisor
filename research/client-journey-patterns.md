# Client Journey Patterns

Extracted from real client engagements: ARC Airlines, ACI, Island, NinjaTrader, Tricon Residential.

## The Universal Client Journey

Every client follows roughly the same path, regardless of industry or size:

### Phase 1: "Do I need a Relay Proxy?"

Clients ask this when they have one or more of these situations:
- Applications running on-prem or in private data centers
- Strict firewall/proxy rules, no direct outbound internet from apps
- Security team wants to control what talks to external services
- Want to reduce the number of outbound connections to LD
- Need resilience during LD outages
- PHP or serverless environments that can't maintain streaming connections
- Browser SDK traffic being blocked by ad-blockers (NinjaTrader pattern)
- Need to prevent TLS proxy manipulation of flag data (Island pattern)

Decision tree inputs:
- Where are apps hosted? (on-prem, cloud, hybrid, customer-hosted)
- Do apps have direct internet access?
- How many services will use LD?
- Are there security/compliance requirements (PCI, SOC2, etc.)?
- Is there a serverless or PHP component?
- Are ad-blockers causing event delivery issues?
- Is there a need for air-gapped/offline deployment?

### Phase 2: "Which mode: proxy or daemon?"

This is almost always decided by one question: **do you have browser/mobile/client-side apps?**

- If yes: proxy mode (daemon mode can't serve browser SDKs)
- If server-side only + want maximum resilience: daemon mode is an option
- If server-side only + PHP: daemon mode often preferred
- If air-gapped/offline: offline mode (Enterprise only, separate from proxy/daemon)

Other factors:
- Does the client want centralized infrastructure? (proxy mode)
- Do they already have Redis/DynamoDB they want SDKs to read from directly? (daemon mode)
- Do they need experimentation/A/B testing? (requires events to flow back, rules out pure offline mode)

### Phase 3: "What does deployment look like?"

Common questions:
- How many Relay Proxy instances? (LD recommends minimum 3)
- Do I need a load balancer? (yes, with SSE support)
- Single cluster or separate for frontend/backend?
- Where does Redis fit?
- How do browser SDKs reach the Relay Proxy?
- Do I need separate Relay Proxies for different deployment models?

The answer depends on:
- Number of environments (prod, staging, dev)
- Frontend vs backend split
- Network topology (DMZ, reverse proxy, Cloudflare, etc.)
- Existing infrastructure (Redis, load balancers, container orchestration)
- Customer-hosted vs vendor-hosted deployments (ACI pattern)
- Scale: number of pods, HPA config (Island pattern: 20-200 pods)

### Phase 4: "How do I size it?"

Every client asks for:
- CPU, memory, disk, network per Relay Proxy instance
- Number of instances
- Redis sizing
- Load balancer requirements

The answer is always based on:
- LD official benchmark: m4.xlarge (4 vCPU, 16 GiB) tested instance
- 3 minimum instances across 2+ AZs (LD recommendation)
- Network bandwidth is the primary constraint
- Redis sizing depends on flag/segment complexity, not just flag count
- Plan for 2x expected concurrent connections

### Phase 5: "What about Redis?"

Sub-questions every client has:
- Do I need Redis at all?
- Can I share my existing Redis? (ARC pattern: PCI concern with session Redis)
- Disk persistence needed?
- Separate logical DB or separate instance?
- What eviction policy? (must be noeviction)
- How much memory?
- What about cache TTL? (LD recommends infinite)

### Phase 6: "What about security?"

Common security team questions:
- What data leaves our network?
- What ports need to be opened?
- Is data encrypted in transit/at rest?
- What if the Relay Proxy is compromised?
- PCI/compliance implications? (ARC pattern)
- Outbound-only or bidirectional? (SSE confusion is common)
- Can flag data be manipulated in transit? (Island pattern: TLS proxy attacks)
- Does LD use our data for AI training? (ARC explicitly asked this)

## Patterns by Client

| Client | Industry | Hosting | Mode | Redis | Key Concern |
|--------|----------|---------|------|-------|-------------|
| ARC Airlines | Airlines/Payments | On-prem + Cloudflare UI | Proxy | Yes (separate, PCI) | Zero-downtime, security team buy-in, PCI |
| ACI | Payments Infrastructure | Multi-deployment (ACI-hosted + customer-hosted) | Proxy + Offline | TBD | Air-gapped deployments, 4-nines, kill switches |
| Island | Enterprise Browser | EKS/Kubernetes | Proxy | No | TLS proxy manipulation, browser security, scale (20-200 pods) |
| NinjaTrader | Trading Platform | Direct (no Relay) | N/A | N/A | Ad-blockers blocking events, missing 20% of contexts |
| Tricon Residential | Real Estate | Greenfield | TBD | TBD | Org structure, flag design, new to LD |

## Key Insights for the App

1. **80% of decisions follow a predictable flowchart.** The remaining 20% is client-specific context.
2. **The SSE connection model confuses every security team.** Need a clear, visual explanation every time.
3. **Redis questions always come up** even when Redis is optional. Clients need clear guidance on when/why.
4. **Browser vs server-side is the single biggest architectural driver.** It determines proxy vs daemon immediately.
5. **Every client wants a PDF they can hand to their DevOps/security team.** The app should generate this.
6. **Offline/air-gapped is a separate track** that only applies to Enterprise customers with customer-hosted deployments.
7. **Ad-blocker issues** can be a driver for Relay Proxy adoption even when the client has direct connectivity (NinjaTrader).
8. **Scale varies wildly.** From a handful of services (Tricon) to 200 pods with HPA (Island). Sizing guidance must be flexible.
9. **Multi-deployment models** (ACI: vendor-hosted + customer-hosted) need separate architecture tracks within the same engagement.
10. **Experimentation/A/B testing breaks in offline mode.** This is a critical gotcha that clients discover late.
