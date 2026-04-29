# High-Level Design (HLD)

## LD Relay Advisor

**Version:** 1.0
**Author:** Arif Shaikh
**Date:** April 2026

---

## 1. System Overview

```
                    LD Relay Advisor - System Architecture

 ┌──────────────────────────────────────────────────────────────────┐
 │                        Streamlit App                             │
 │                                                                  │
 │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
 │  │ Module 1  │  │ Module 2  │  │ Module 3  │  │  Module 4     │   │
 │  │ Discovery │->│ Decision  │->│ Arch Gen  │->│  Sizing Calc  │   │
 │  │ (Q&A)    │  │ Engine    │  │           │  │               │   │
 │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │
 │       │                                           │             │
 │       │         ┌──────────┐              ┌──────────────┐     │
 │       └-------->│ Module 5  │              │  Module 6     │     │
 │                 │ Report    │              │  AI Chat      │     │
 │                 │ Generator │              │  (RAG)        │     │
 │                 └──────────┘              └──────┬───────┘     │
 │                      │                           │             │
 └──────────────────────┼───────────────────────────┼─────────────┘
                        │                           │
                        v                           v
                  ┌──────────┐              ┌──────────────┐
                  │ PDF/Typst │              │ Claude API    │
                  │ Output    │              │ + RAG Index   │
                  └──────────┘              └──────────────┘
```

## 2. Module Architecture

### Module 1: Discovery (Q&A Flow)

**Type:** Stateful form wizard
**UI:** Streamlit multi-step form with progress bar

```
Page Flow:
  Infrastructure (5 questions)
  -> Resilience (3 questions)
  -> Preferences (3 questions)
  -> Security (3 questions)
  -> Scale (3 questions)
  -> Review & Confirm
```

**Data Model:**
```python
@dataclass
class ClientProfile:
    # Infrastructure
    hosting: str              # on-prem, aws, azure, gcp, hybrid, customer-hosted
    internet_access: str      # direct, proxy-firewall, air-gapped
    has_browser_apps: bool
    has_server_apps: bool
    server_languages: list[str]
    service_count: str        # <10, 10-50, 50-200, 200+
    environment_count: int

    # Resilience
    outage_tolerance: str     # zero-downtime, acceptable-degradation, fallback-ok
    frequent_deployments: bool
    air_gapped: bool

    # Preferences
    infra_preference: str     # centralized, embedded
    existing_datastore: str   # redis, dynamodb, consul, none
    redis_shared: bool
    redis_compliance: str     # pci, none, unknown

    # Security
    compliance: list[str]     # pci, soc2, hipaa, none
    security_review_needed: bool
    adblocker_concern: bool

    # Scale
    peak_connections: str     # <100, 100-1000, 1000-10000, 10000+
    container_orchestration: str  # kubernetes, ecs, docker-swarm, none
    multi_deployment: bool
```

### Module 2: Decision Engine

**Type:** Rule-based engine (no AI needed for core decisions)

```
Input: ClientProfile
Output: ArchitectureRecommendation

@dataclass
class ArchitectureRecommendation:
    needs_relay: bool
    relay_reasons: list[str]
    mode: str                 # proxy, daemon, offline, not-needed
    mode_reasons: list[str]
    needs_redis: bool
    redis_reasons: list[str]
    redis_separate: bool
    redis_persistence: bool
    confidence: str           # high, medium, low
    caveats: list[str]
```

**Decision Rules (simplified):**

```
Rule 1: needs_relay
  IF hosting in (on-prem, hybrid) OR internet_access != direct
     OR service_count > 50 OR adblocker_concern OR air_gapped
  THEN needs_relay = True

Rule 2: mode
  IF air_gapped THEN mode = offline
  ELIF has_browser_apps THEN mode = proxy
  ELIF server_languages contains PHP THEN mode = daemon
  ELIF infra_preference == centralized THEN mode = proxy
  ELSE mode = proxy (default safer choice)

Rule 3: needs_redis
  IF mode == daemon THEN needs_redis = True (required)
  ELIF outage_tolerance == zero-downtime THEN needs_redis = True
  ELIF frequent_deployments THEN needs_redis = True
  ELSE needs_redis = False (optional)

Rule 4: redis_separate
  IF redis_shared AND redis_compliance in (pci, soc2)
  THEN redis_separate = True

Rule 5: redis_persistence
  IF outage_tolerance == zero-downtime
  THEN redis_persistence = True
```

### Module 3: Architecture Generator

**Type:** Template-based generator

Takes the recommendation and produces:
- Network requirements table (endpoints, ports, protocols)
- Deployment topology description
- Sample configuration file (TOML)
- Architecture diagram (text-based for MVP, image for v2)

**Templates:**
```
templates/
  proxy-mode-onprem.toml
  proxy-mode-cloud.toml
  daemon-mode.toml
  proxy-mode-with-redis.toml
  network-requirements.md
```

### Module 4: Sizing Calculator

**Type:** Interactive calculator with sliders/inputs

**Inputs:**
- Number of environments
- Expected concurrent connections
- Flag/segment complexity (simple, moderate, complex)
- Big Segments planned? (yes/no)

**Outputs (all sourced from LD official guidance):**

```
Relay Proxy:
  instances: max(3, ceil(expected_connections * 2 / connections_per_instance))
  cpu_per_instance: 4 vCPU (m4.xlarge reference)
  memory_per_instance: 8-16 GB
  network: "high throughput required"

Redis:
  instances: 1 primary + 1 replica (if HA)
  memory: based on env_count * complexity_factor
  persistence: AOF recommended if zero-downtime
  eviction_policy: noeviction (always)

Load Balancer:
  type: L4 or L7 with SSE support
  health_check: GET /status on port 8030
  idle_timeout: >= 60s
```

### Module 5: Report Generator

**Type:** Document builder

**Tech:** Typst templates (proven in ARC Airlines workflow) compiled to PDF.

**Report sections:**
1. Executive Summary
2. Client Profile (from Module 1)
3. Recommendation (from Module 2)
4. Architecture (from Module 3)
5. Sizing (from Module 4)
6. Security FAQ (templated based on compliance needs)
7. Cache and Resilience Overview
8. Open Items / Next Steps
9. References (official LD docs links)

### Module 6: AI Chat (RAG)

**Type:** Conversational Q&A

**Knowledge sources:**
- ld-relay codebase: config/config.go, docs/, README.md, internal/ (key files)
- Official LD docs: relay proxy guidelines, persistent storage, configuration
- Anonymized client patterns (from research/client-journey-patterns.md)

**RAG pipeline:**
```
User Question
    |
    v
Embedding Search (knowledge base)
    |
    v
Top-K relevant chunks
    |
    v
Claude API (question + context chunks + system prompt)
    |
    v
Answer (with source citations)
```

**System prompt rules:**
- Always cite the source (codebase file or doc URL)
- If the answer can't be verified, say so
- Never assume behavior that isn't in the code or docs
- Label estimates as estimates

## 3. Data Flow

```
User -> Streamlit UI -> Session State (in-memory only)
                            |
                            v
                      Decision Engine (pure logic)
                            |
                            v
                      Architecture + Sizing (templates + calculations)
                            |
                            v
                      Report Generator (Typst -> PDF)
                            |
                            v
                      Download (browser)
```

No data persisted. No database. Everything lives in Streamlit session state and dies when the session ends.

## 4. Technology Decisions

| Component | Choice | Why |
|-----------|--------|-----|
| Frontend | Streamlit | Python-native, fast to build, good for forms + chat |
| Decision Engine | Pure Python (no ML) | Decisions are rule-based, not probabilistic |
| PDF Generation | Typst | Already proven in ARC workflow, clean output |
| AI Chat | Claude API (Anthropic SDK) | Best reasoning for technical Q&A |
| RAG Embeddings | Voyage or Claude embeddings | For knowledge base indexing |
| Vector Store | ChromaDB (local) or simple JSON | Small corpus, no need for heavy infra |
| Deployment | Streamlit Cloud | Free tier, easy sharing, no infra |
| Data Storage | None | Session-only, privacy by design |

## 5. Key Design Principles

1. **Source of truth:** All technical claims come from the ld-relay codebase or official LD docs. The app never makes up behavior.
2. **Estimates are labeled:** Anything not from official sources is clearly marked as a recommendation or estimate.
3. **Privacy first:** No client data stored. No analytics on client inputs. Session dies on close.
4. **Progressive disclosure:** Start simple (do I need a relay?), go deeper only if the client wants to.
5. **PDF is the deliverable:** The app is the journey, the PDF is what the client takes to their team.
6. **Offline-capable core:** The decision engine and sizing calculator work without AI/API calls. AI chat is an enhancement, not a dependency.
