# Service Connection Patterns

Extracted from: `ServiceConnection-RelayProxy/RelayProxy-ServiceConnections-Session 2 Core Product Tech Enablement - Architectural Patterns and Usage Trends.pptx` (71 slides)
Source: Internal LD Revenue Enablement, presented by Matthew Magne, John Winstead, Ruth Neligan

## What is a Service Connection?

"Philosophically speaking, a service connection is one unit of value that a customer gets from LaunchDarkly."

### How SCs are counted
- Once a minute, LD counts concurrent streaming connections
- 1 concurrent streaming connection = 1 minute's worth of SCs
- 1 polling request = 1 hour's worth of SCs
- 1 service connection = a month's worth of streaming connections

### Billing notes
- SDKs behind the Relay Proxy: billed (SCs are counted)
- SDKs using polling: billed (1 poll = 1 hour of SCs)
- SDKs in Daemon mode: currently CANNOT count SCs (requires custom billing)

---

## Architecture Component Categories

| Category | Components | Our responsibility |
|----------|-----------|-------------------|
| LaunchDarkly products | SDK, Relay Proxy, LD CLI | Full support |
| Supporting components | Customer's applications, persistent store | Support LD's interaction with them |
| Adjacent components | Host, proxy, firewall, load balancer | Not supported |

### Adjacent Component Issues (slides 13)
- **HTTP/Web Proxy:** Network timeouts. Root cause: proxy lacks SSE support. Fix: enable SSE support.
- **Firewall:** Timeouts and refused connections. Root cause: firewall limits egress. Fix: whitelist LaunchDarkly domains.
- **Load balancer:** 502 errors during pod scaling. Root cause: LB routes to terminating Relay instances. Fix: graceful shutdown with health check failure.

---

## Architecture Patterns and SC Formulas

### Pattern 1: Typical Server-Side (slides 15-26)
- One host, one application, one SDK instance
- **SC = 1 per host**
- Predictable
- Horizontally scaled: **SC = number of hosts**

### Pattern 2: Multiple SDK Instances per Host (slide 22) -- ANTI-PATTERN
- Application initializes SDK more than once
- This is a code-level implementation bug
- Symptoms:
  - High memory usage
  - High CPU usage
  - High network usage
  - Slower initialization
  - Stale targeting rules
  - EOF errors
  - Timeouts
  - Requires scaling earlier
- **Fix: Initialize SDK once, share the client instance**

### Pattern 3: Forking (slides 27-30)
- Ruby (Puma workers), Python (Gunicorn workers)
- Each fork creates its own SDK instance
- **SC = number_of_hosts * number_of_forks**
- Moderately predictable
- Horizontally scaled + forking: **SC = hosts * forks** (hard to predict if auto-scaled)

### Pattern 4: Containerized (slides 31-40)
- Docker containers, each with one SDK instance
- **SC = number_of_containers**
- Hard to predict if containers auto-scale
- With Relay sidecar: **SC still = containers, but only 1 outbound connection per Relay**
- Horizontally scaled: **SC = containers * hosts**
- Horizontally scaled + forking: **SC = containers * hosts * forks** (worst case)

### Pattern 5: Lambda / Serverless (slides 41-44)
- Each Lambda invocation with cold start creates new SDK instance
- **SC = concurrent Lambda instances**
- Very hard to predict due to Lambda scaling
- Recommendation: Use Relay Proxy on EC2 to reduce outbound connections

---

## Optional/Situational Configurations

### Relay for Redundancy (slides 46-47) -- ANTI-PATTERN

"In case LaunchDarkly goes down" is NOT a good standalone reason for Relay.

Problems:
- Relay requires uptime equal to or better than LD
- Requires periodic reassessment to scale
- Relay array must be warmed up before LD goes down (basically on all the time)

Correct approach:
- Keep SDKs connected directly to LaunchDarkly
- Use dynamic scaling to handle variable load
- Have a switchover plan in place
- Periodically test the plan
- Relay array requires being warmed up prior to LD going down

### Persistent Store (slides 48-50)

Use cases:
- Required for Big Segments feature
- Ensures Relay can cold-start when LD is down

Trade-offs:
- Introduces a partially-supported component
- Introduces a new point of failure
- Scaling the persistent store is harder than scaling Relay and less supported

With Relay + persistent store:
- Relay arrays can cold start when LD is down
- But introduces new point of failure
- Requires scaling the store

With SDK + Relay + persistent store (both connected):
- SDKs could already cold-start by connecting to their Relay array
- Requires scaling the store EVEN MORE
- Added complexity for marginal benefit

### Daemon Mode (slide 51)

Benefits:
- Reduces egress per host
- Egress can be reduced more using Relay sidecar pattern

Trade-offs:
- Introduces a critical point of failure
- Requires scaling the persistent store EVEN MORE
- SDKs could already cold-start by connecting to their Relay array
- Requires custom billing due to current inability to track SCs in Daemon mode

### LD CLI (slide 52)
- Allows devs to control different flag configurations locally
- Could theoretically abuse SC count, but loses visibility, reliability, auditability
- Prevents SCs from lower/dev environments

---

## Key Takeaways for the Advisor App

1. **Service connection estimation is critical.** Clients need to understand their SC count before sizing Relay. The forking and container multipliers are often surprising.

2. **Anti-pattern detection saves clients from mistakes.** Multiple SDK instances, Relay-for-redundancy-only, and over-engineered persistent store are common traps.

3. **Architecture pattern drives everything.** The discovery conversation needs to identify: forking? containerized? auto-scaled? serverless? These determine SC formula, Relay topology, and sizing.

4. **Daemon mode has hidden costs.** SC billing is currently broken for daemon mode. This needs to be surfaced prominently in the recommendation.

5. **Adjacent component issues (proxy, firewall, LB) cause most support tickets.** The app should generate specific configuration guidance for these.
