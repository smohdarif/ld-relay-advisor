# Module Specifications

## Development Order and Dependencies

```
Phase 1 (MVP - Core Flow):
  Module 1: Discovery ──> Module 2: Decision Engine
  (no external dependencies, pure Python)

Phase 2 (Sizing + Architecture):
  Module 3: Architecture Generator (depends on Module 2)
  Module 4: Sizing Calculator (depends on Module 2)
  (templates + math, no external API)

Phase 3 (Report):
  Module 5: Report Generator (depends on Modules 2, 3, 4)
  (requires Typst installed)

Phase 4 (AI Chat):
  Module 6: AI Q&A (independent, sidebar)
  (requires Claude API key, RAG index)
```

## Module 1: Discovery

**File:** `modules/discovery.py`
**Dependencies:** Streamlit
**Estimated effort:** 2-3 days

**What it does:**
- Renders a multi-step form in Streamlit
- 5 pages: Infrastructure, Resilience, Preferences, Security, Scale
- Progress bar showing which step the user is on
- "Back" and "Next" buttons
- Saves all answers to Streamlit session state as a ClientProfile object
- Review page at the end showing all answers before proceeding

**UI components:**
- `st.radio` for single-choice questions (hosting type, internet access)
- `st.multiselect` for multi-choice (compliance types, languages)
- `st.checkbox` for yes/no (has browser apps, air-gapped)
- `st.slider` for numeric (environment count)
- `st.selectbox` for dropdowns (service count ranges, deployment platform, reverse proxy)
- `st.text_input` for company name

**Discovery pages (6 pages, updated):**
1. **Company and Infrastructure** - company name, hosting, internet access, deployment platform (K8s/ECS/Docker/VM), reverse proxy (nginx/HAProxy/ALB/none)
2. **Applications** - browser apps, server apps, server languages, service count
3. **Resilience** - outage tolerance, frequent deployments, air-gapped
4. **Preferences** - infra preference, existing datastore, preferred persistent store (Redis/DynamoDB/Consul), Redis sharing and compliance
5. **Security** - compliance frameworks, security review, ad-blocker concern
6. **Scale** - server-side connection scale, client-side connection scale (separate), container orchestration, multi-deployment, segment complexity, Big Segments

**New questions added (from Field Guide analysis):**
- Deployment platform: drives K8s-specific guidance (Deployment vs DaemonSet, readiness probes, PDB) vs ECS auto-scaling vs VM/systemd
- Reverse proxy/LB in front of Relay: drives SSE configuration checklist (buffering, gzip, timeouts, CORS)
- Client-side vs server-side connection scale (split): Field Guide warns client-side streaming through Relay is "not efficient" at scale
- Preferred persistent store: DynamoDB and Consul are equal options to Redis per Field Guide

**Learnings from client work:**
- Keep questions simple. Clients don't always know their exact connection count.
- Use ranges instead of exact numbers (e.g., "<10, 10-50, 50-200, 200+").
- Add help tooltips explaining why each question matters.
- Allow skipping non-critical questions with sensible defaults.

---

## Module 2: Decision Engine

**File:** `modules/decision_engine.py` + `engine/rules.py`
**Dependencies:** None (pure Python)
**Estimated effort:** 2-3 days

**What it does:**
- Takes a ClientProfile, runs it through rules, produces ArchitectureRecommendation
- Three main decisions: needs relay? which mode? needs Redis?
- Each decision comes with a list of reasons
- Confidence level (high/medium/low) based on how many questions were answered
- Caveats list for edge cases

**Display:**
- Four cards/panels showing each decision
- Green/yellow/red color coding for confidence
- Expandable reasoning for each decision (both FOR and AGAINST)
- "Edit answers" button to go back to discovery

**Decisions (updated):**
1. Do you need a Relay Proxy? (yes/no with reasons FOR and reasons AGAINST)
2. Recommended mode (proxy/daemon/offline)
3. Persistent store recommendation (Redis/DynamoDB/Consul with reasoning)
4. Trade-offs and caveats (what you gain vs what complexity you add)

**Anti-pattern detection (from Field Guide pp.82-83):**
The engine now actively recommends AGAINST relay when:
- Fewer than 10 server-side SDK instances (connection savings minimal)
- Client-side SDKs only (CDN-backed endpoints handle scale)
- Goal is latency reduction (Relay adds a hop)
- Goal is redundancy only (SDKs already cache in memory)
- Direct internet access with no compliance/firewall restrictions

**Daemon mode triggers (expanded from Field Guide pp.86-88):**
- PHP apps (cannot maintain streaming connections)
- Large server-side fleet (50-200+) with existing shared data store and embedded preference
- SDKs that can read directly from Redis/DynamoDB/Consul

**Learnings from client work:**
- The browser app question is the single most important input. It decides proxy vs daemon immediately.
- Air-gapped is a special case that overrides everything else.
- Most clients land on "proxy mode + Redis" because they want zero downtime.
- Always show the reasoning. Clients need to justify the decision to their teams.
- Showing reasons AGAINST is just as valuable. It builds trust when the tool says "you don't need this."

---

## Module 3: Architecture Generator

**File:** `modules/architecture.py` + `engine/config_generator.py`
**Dependencies:** Jinja2 (for template rendering)
**Estimated effort:** 3-4 days

**What it does:**
- Generates a text-based architecture diagram based on the recommendation
- Produces a network requirements table (endpoints, ports, purpose)
- Generates a sample Relay Proxy config file (TOML)
- Shows deployment topology (how many instances, LB, Redis layout)

**Templates needed:**
- Proxy mode, on-prem, with Redis
- Proxy mode, on-prem, without Redis
- Proxy mode, cloud, with Redis
- Proxy mode, cloud, without Redis
- Daemon mode with Redis/DynamoDB/Consul
- Offline mode with persistent store
- Offline mode with file-based data source
- Each template has placeholders for environment names, prefixes, proxy URLs

**New sections (from Field Guide analysis):**

**Reverse proxy/LB configuration checklist** (Field Guide pp.143-144):
Generated when reverse_proxy != "none". Platform-specific settings:
- Disable response buffering for SSE endpoints
- Disable forced gzip if proxy not SSE-aware
- Set SSE response timeout to minimum 10 minutes
- Set upstream/proxy read timeout to 5 minutes (nginx: `proxy_read_timeout`)
- Restrict CORS headers to client domains only (when browser SDKs use Relay)
- Restrict `/status` endpoint and Prometheus port from public access

**Deployment platform guidance** (Field Guide pp.98-100):
- Kubernetes: Deployment (centralized) vs DaemonSet (per-node). Readiness probes on `/status`. PodDisruptionBudgets. Resource requests/limits.
- ECS/Fargate: ECS service with auto-scaling on CPU or connection count.
- Docker: Official image `launchdarkly/ld-relay` with env var config.
- VM/bare metal: Binary + systemd process management.

**AutoConfig policy examples** (Field Guide pp.145-148):
When environment_count >= 3, generate sample AutoConfig policies:
- Production only across all projects
- Specific project scoping
- Tag-based filtering
- Deny patterns for sensitive environments

**Rolling deployment runbook** (Field Guide pp.106-110):
1. Deploy new instances alongside existing
2. Wait for initialization and flag data sync
3. Shift traffic via load balancer
4. Drain connections from old instances
5. Terminate old instances

**Learnings from client work:**
- Every client asks for a sample config file. Having one ready saves a meeting.
- The network requirements table is what security teams actually care about.
- Architecture diagrams don't need to be fancy. Text-based is fine for technical audiences.
- Always include the corporate proxy config section if internet_access is proxy-firewall.
- The reverse proxy checklist alone saves clients hours of debugging SSE issues.

---

## Module 4: Sizing Calculator

**File:** `modules/sizing.py` + `engine/sizing_calculator.py`
**Dependencies:** None (pure Python)
**Estimated effort:** 2-3 days

**What it does:**
- Interactive sliders and inputs for fine-tuning
- Calculates Relay Proxy instances, CPU, memory, network
- Calculates Redis instances, memory, persistence
- Calculates total bill of materials
- Clearly labels what's from LD official docs vs estimates

**Interactive inputs:**
- Number of environments (slider: 1-10)
- Expected server-side peak connections (dropdown: ranges)
- Expected client-side peak connections (dropdown: ranges, with warning about efficiency)
- Flag/segment complexity (radio: simple, moderate, complex)
- Big Segments planned (checkbox)
- Adjust memory per instance (slider: 4-16 GB, default 8)

**Output:**
- Relay Proxy sizing table
- Persistent store sizing table (Redis, DynamoDB, or Consul depending on recommendation)
- Total BOM table
- Notes/disclaimers on what's official vs estimated
- Scaling signals and alerting thresholds table (from Field Guide pp.111-118)
- Environment partitioning recommendation (when environment_count >= 5)
- File descriptor note (each SDK connection = 1 fd, check OS limits)

**New: Scaling signals panel** (Field Guide pp.92-94, 111-118):
Shows when to scale, with specific thresholds:
- CPU sustained > 70% for 10 min: add instances
- Memory growth > 20% in 1 hour: investigate segments
- Connection count drops > 50%: check Relay health
- Environment disconnected > 5 min: check LD connectivity
- Event forwarding delayed > 5 min: check backlog
- Status endpoint unreachable: critical alert

**New: Environment partitioning** (Field Guide pp.95-97):
When environment_count >= 5, recommend separating prod vs non-prod clusters to limit blast radius.

**Learnings from client work:**
- Sean Falzon's feedback: Redis memory depends on rules/segments, not just flag count. Make this clear.
- The 2x connection buffer is from official LD docs. Always mention it.
- m4.xlarge is the reference, not a hard requirement. Say "or equivalent."
- Clients love a total BOM table they can hand to procurement.
- The scaling signals table gives ops teams something actionable from day one.

---

## Module 5: Report Generator

**File:** `modules/report.py` + `templates/report.typ`
**Dependencies:** Typst (CLI), subprocess
**Estimated effort:** 3-4 days

**What it does:**
- Takes outputs from Modules 2, 3, 4
- Renders a Typst template with all the data
- Compiles to PDF using `typst compile`
- Returns PDF as bytes for Streamlit download button

**Report sections:**
1. Title page (company name, date)
2. Executive summary (auto-generated from decisions)
3. Your environment (table from ClientProfile answers)
4. Recommendation (relay, mode, persistent store with reasons FOR and AGAINST)
5. Architecture diagram
6. Sizing table (with official vs estimate labels)
7. Network requirements
8. Reverse proxy/LB configuration checklist (conditional: when reverse_proxy != "none")
9. Deployment guidance (platform-specific: K8s, ECS, Docker, VM)
10. Security FAQ (only questions relevant to their compliance profile)
11. Cache and resilience overview
12. Monitoring and alerting playbook (metrics, thresholds, actions)
13. Rolling deployment runbook (step-by-step upgrade procedure)
14. Sample configuration (TOML + AutoConfig policies when applicable)
15. Open items / suggested next steps
16. References (LD docs links + Field Guide citations)

**New sections added (from Field Guide analysis):**
- **Reverse proxy checklist** (section 8): SSE buffering, gzip, timeouts, CORS, endpoint restriction. Only included when client has a reverse proxy/LB.
- **Deployment guidance** (section 9): Platform-specific instructions. K8s gets Deployment vs DaemonSet, readiness probes, PDB. ECS gets auto-scaling config. VM gets systemd setup.
- **Monitoring playbook** (section 12): Key metrics table, alerting thresholds with severity levels, and specific actions for each condition.
- **Deployment runbook** (section 13): Step-by-step rolling deployment procedure.

**Learnings from client work:**
- The PDF is what clients actually share with their team. Make it look professional.
- Always include the references section. It builds trust.
- The executive summary should be 3-4 sentences max.
- Include "Open Items" so clients know what to figure out next.
- The monitoring playbook is the section ops teams actually bookmark.

---

## Module 6: AI Chat (RAG)

**File:** `modules/chat.py` + `knowledge/`
**Dependencies:** anthropic SDK, chromadb (or simple JSON index)
**Estimated effort:** 4-5 days

**What it does:**
- Sidebar chat that's available on every page
- Takes natural language questions about Relay Proxy
- Searches a knowledge base (RAG) for relevant context
- Calls Claude API with context + question
- Returns answer with source citations

**Knowledge base sources:**
- ld-relay docs/ folder (markdown files)
- ld-relay README.md (performance section)
- ld-relay config/config.go (defaults and descriptions)
- Official LD docs pages (relay proxy guidelines, persistent storage)
- LaunchDarkly Field Guide PDF (relay sections pp.78-148, extracted to research/field-guide-patterns.md)
- Anonymized client patterns (common questions and answers)

**System prompt requirements:**
- Only answer from provided context
- Always cite the source
- If can't verify, say "I cannot verify this"
- Label estimates as estimates
- Use simple language, avoid jargon

**Learnings from client work:**
- "What happens if X goes down?" is the most common question pattern.
- Clients ask about cache TTL, Big Segments, and Redis separation constantly.
- The whiteboard analogy for cache TTL was very effective. Include it in the knowledge base.
- The bouncer analogy for Big Segments worked well too.
