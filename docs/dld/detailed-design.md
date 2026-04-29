# Detailed Low-Level Design (DLD)

## LD Relay Advisor

**Version:** 1.0
**Author:** Arif Shaikh
**Date:** April 2026

---

## 1. Project Structure

```
ld-relay-advisor/
  app.py                          # Streamlit entry point
  requirements.txt
  pyproject.toml

  modules/
    __init__.py
    discovery.py                  # Module 1: Q&A flow UI
    decision_engine.py            # Module 2: Rule-based recommendations
    architecture.py               # Module 3: Arch diagram + config generation
    sizing.py                     # Module 4: Sizing calculator
    report.py                     # Module 5: PDF generation via Typst
    chat.py                       # Module 6: AI Q&A with RAG

  models/
    __init__.py
    client_profile.py             # ClientProfile dataclass
    recommendation.py             # ArchitectureRecommendation dataclass
    sizing_result.py              # SizingResult dataclass

  engine/
    __init__.py
    rules.py                      # Decision rules (relay needed, mode, redis)
    sizing_calculator.py          # Sizing math
    config_generator.py           # TOML config template renderer

  knowledge/
    __init__.py
    indexer.py                    # Build RAG index from sources
    retriever.py                  # Query RAG index
    sources/                      # Markdown/text files for RAG
      relay-proxy-guidelines.md
      persistent-storage.md
      configuration-reference.md
      codebase-notes.md           # Key findings from ld-relay source
      client-patterns.md          # Anonymized patterns

  templates/
    report.typ                    # Typst template for PDF report
    configs/
      proxy-mode.toml
      proxy-mode-redis.toml
      daemon-mode.toml
      proxy-mode-cloud.toml

  assets/
    logo.png
    architecture-diagrams/

  tests/
    test_decision_engine.py
    test_sizing_calculator.py
    test_config_generator.py
    test_report_generation.py
    test_discovery_flow.py
    test_chat_rag.py
```

## 2. Detailed Module Specs

### 2.1 models/client_profile.py

```python
from dataclasses import dataclass, field
from enum import Enum

class Hosting(Enum):
    ON_PREM = "on-prem"
    AWS = "aws"
    AZURE = "azure"
    GCP = "gcp"
    HYBRID = "hybrid"
    CUSTOMER_HOSTED = "customer-hosted"

class InternetAccess(Enum):
    DIRECT = "direct"
    PROXY_FIREWALL = "proxy-firewall"
    AIR_GAPPED = "air-gapped"

class OutageTolerance(Enum):
    ZERO_DOWNTIME = "zero-downtime"
    ACCEPTABLE_DEGRADATION = "acceptable-degradation"
    FALLBACK_OK = "fallback-ok"

class InfraPreference(Enum):
    CENTRALIZED = "centralized"
    EMBEDDED = "embedded"

class ServiceCount(Enum):
    SMALL = "<10"
    MEDIUM = "10-50"
    LARGE = "50-200"
    XLARGE = "200+"

class ConnectionScale(Enum):
    SMALL = "<100"
    MEDIUM = "100-1000"
    LARGE = "1000-10000"
    XLARGE = "10000+"

class SegmentComplexity(Enum):
    SIMPLE = "simple"           # Mostly boolean flags, few rules
    MODERATE = "moderate"       # Mix of simple and multi-rule flags
    COMPLEX = "complex"         # Many rules, large segments, Big Segments likely

@dataclass
class ClientProfile:
    # Company
    company_name: str = ""

    # Infrastructure
    hosting: Hosting = Hosting.ON_PREM
    internet_access: InternetAccess = InternetAccess.PROXY_FIREWALL
    has_browser_apps: bool = False
    has_server_apps: bool = True
    server_languages: list = field(default_factory=list)
    service_count: ServiceCount = ServiceCount.MEDIUM
    environment_count: int = 3

    # Resilience
    outage_tolerance: OutageTolerance = OutageTolerance.ZERO_DOWNTIME
    frequent_deployments: bool = True
    air_gapped: bool = False

    # Preferences
    infra_preference: InfraPreference = InfraPreference.CENTRALIZED
    existing_datastore: str = "none"   # redis, dynamodb, consul, none
    redis_shared: bool = False
    redis_compliance: str = "none"     # pci, soc2, none, unknown

    # Security
    compliance: list = field(default_factory=list)   # pci, soc2, hipaa
    security_review_needed: bool = True
    adblocker_concern: bool = False

    # Scale
    peak_connections: ConnectionScale = ConnectionScale.MEDIUM
    container_orchestration: str = "none"  # kubernetes, ecs, docker-swarm, none
    multi_deployment: bool = False

    # Sizing extras
    segment_complexity: SegmentComplexity = SegmentComplexity.MODERATE
    big_segments_planned: bool = False
```

### 2.2 engine/rules.py

```python
# Each rule returns a tuple: (decision: bool/str, reasons: list[str])

def needs_relay_proxy(profile: ClientProfile) -> tuple[bool, list[str]]:
    """Determine if the client needs a Relay Proxy."""
    reasons = []

    if profile.hosting in (Hosting.ON_PREM, Hosting.HYBRID, Hosting.CUSTOMER_HOSTED):
        reasons.append("Applications hosted on-prem or in hybrid environment")

    if profile.internet_access != InternetAccess.DIRECT:
        reasons.append(f"Traffic routed through {profile.internet_access.value}")

    if profile.air_gapped:
        reasons.append("Air-gapped environment requires offline mode")

    if profile.service_count in (ServiceCount.LARGE, ServiceCount.XLARGE):
        reasons.append(f"{profile.service_count.value} services - Relay reduces outbound connections")

    if profile.adblocker_concern:
        reasons.append("Ad-blockers may block direct LD connections from browsers")

    if profile.infra_preference == InfraPreference.CENTRALIZED:
        reasons.append("Preference for centralized infrastructure aligns with Relay Proxy")

    return (len(reasons) > 0, reasons)


def recommend_mode(profile: ClientProfile) -> tuple[str, list[str]]:
    """Recommend proxy, daemon, or offline mode."""
    reasons = []

    if profile.air_gapped:
        reasons.append("Air-gapped environments require offline mode (Enterprise only)")
        return ("offline", reasons)

    if profile.has_browser_apps:
        reasons.append("Browser/client-side apps cannot connect to a database directly - proxy mode required")
        return ("proxy", reasons)

    if "php" in [l.lower() for l in profile.server_languages]:
        reasons.append("PHP cannot maintain streaming connections - daemon mode preferred")
        return ("daemon", reasons)

    if profile.infra_preference == InfraPreference.CENTRALIZED:
        reasons.append("Centralized infrastructure preference aligns with proxy mode")

    reasons.append("Proxy mode is the safer default for most architectures")
    return ("proxy", reasons)


def needs_redis(profile: ClientProfile, mode: str) -> tuple[bool, list[str]]:
    """Determine if Redis is needed."""
    reasons = []

    if mode == "daemon":
        reasons.append("Daemon mode requires a persistent store (Redis, DynamoDB, or Consul)")
        return (True, reasons)

    if profile.outage_tolerance == OutageTolerance.ZERO_DOWNTIME:
        reasons.append("Zero-downtime requirement - Redis ensures flag data survives Relay restarts during LD outages")

    if profile.frequent_deployments:
        reasons.append("Frequent deployments mean new instances need flag data on startup - Redis provides cold-start protection")

    return (len(reasons) > 0, reasons)


def redis_recommendations(profile: ClientProfile) -> dict:
    """Specific Redis configuration recommendations."""
    rec = {
        "separate_instance": False,
        "separate_reason": "",
        "persistence": False,
        "persistence_reason": "",
        "eviction_policy": "noeviction",
        "cache_ttl": "infinite (-1)",
    }

    if profile.redis_shared and profile.redis_compliance in ("pci", "soc2"):
        rec["separate_instance"] = True
        rec["separate_reason"] = (
            f"Existing Redis has {profile.redis_compliance.upper()} constraints. "
            "LD flag data contains no PII and needs different eviction policy (noeviction) "
            "and disk persistence. A separate instance avoids conflicts."
        )
    elif profile.redis_shared:
        rec["separate_instance"] = True
        rec["separate_reason"] = (
            "LD Relay requires noeviction policy. Sharing with session/cache Redis "
            "that uses LRU eviction can cause flag data loss."
        )

    if profile.outage_tolerance == OutageTolerance.ZERO_DOWNTIME:
        rec["persistence"] = True
        rec["persistence_reason"] = (
            "Without disk persistence, a Redis restart during an LD outage means "
            "no flag data for new Relay instances or app deployments."
        )

    return rec
```

### 2.3 engine/sizing_calculator.py

```python
# All "official" numbers sourced from:
#   - LD docs: https://launchdarkly.com/docs/sdk/relay-proxy/guidelines
#   - ld-relay README.md
# Estimates are clearly labeled.

OFFICIAL_INSTANCE_TYPE = "m4.xlarge"
OFFICIAL_VCPU = 4
OFFICIAL_MEMORY_GIB = 16
OFFICIAL_MIN_INSTANCES = 3
OFFICIAL_MIN_AZS = 2
OFFICIAL_IDLE_MEMORY_MIB = 11
OFFICIAL_PROVISION_FACTOR = 2  # plan for 2x expected connections

# Estimates (not from LD docs)
ESTIMATE_RELAY_MEMORY_GB = 8       # practical starting point
ESTIMATE_REDIS_MEMORY_GB = 8       # comfortable headroom
ESTIMATE_REDIS_VCPU = 4
ESTIMATE_DISK_GB = 10

def calculate_relay_sizing(profile):
    """Calculate Relay Proxy sizing."""
    instances = max(OFFICIAL_MIN_INSTANCES, ...)  # based on connection scale
    return {
        "instances": instances,
        "vcpu_per_instance": OFFICIAL_VCPU,
        "memory_per_instance_gb": ESTIMATE_RELAY_MEMORY_GB,
        "disk_per_instance_gb": ESTIMATE_DISK_GB,
        "network_note": "Network bandwidth is the most important resource (LD official guidance)",
        "reference": f"Tested on AWS {OFFICIAL_INSTANCE_TYPE} ({OFFICIAL_VCPU} vCPU, {OFFICIAL_MEMORY_GIB} GiB)",
        "source": "https://launchdarkly.com/docs/sdk/relay-proxy/guidelines",
        "is_official": {
            "instance_type": True,
            "min_instances": True,
            "vcpu": True,       # matches m4.xlarge
            "memory": False,    # estimate - m4.xlarge has 16 GiB
            "disk": False,      # estimate
        }
    }

def calculate_redis_sizing(profile):
    """Calculate Redis sizing."""
    return {
        "instances": "1 primary + 1 replica (HA)",
        "vcpu": ESTIMATE_REDIS_VCPU,
        "memory_gb": ESTIMATE_REDIS_MEMORY_GB,
        "disk_gb": ESTIMATE_DISK_GB,
        "eviction_policy": "noeviction",
        "persistence": "AOF recommended" if profile.outage_tolerance == "zero-downtime" else "optional",
        "cache_ttl": "infinite (-1) per LD recommendation",
        "source": "Estimates based on general best practices. Not from official LD documentation.",
    }
```

### 2.4 templates/report.typ

The Typst template will be parameterized. The Python report module will render variables into the template and compile to PDF using `typst compile`.

Key sections:
1. Title page (company name, date, "Prepared by LD Relay Advisor")
2. Executive summary (1 paragraph, auto-generated from decisions)
3. Your environment (table from ClientProfile)
4. Recommendation (relay needed? mode? redis?)
5. Architecture diagram
6. Sizing table
7. Network requirements
8. Security FAQ
9. Cache and resilience overview
10. Sample configuration
11. Open items
12. References

### 2.5 modules/chat.py (RAG)

```python
# Simplified RAG pipeline

class RelayAdvisorChat:
    def __init__(self, knowledge_dir: str):
        self.index = build_index(knowledge_dir)
        self.client = anthropic.Anthropic()

    def ask(self, question: str, client_profile: ClientProfile = None) -> str:
        # 1. Retrieve relevant chunks
        chunks = self.index.query(question, top_k=5)

        # 2. Build context
        context = "\n---\n".join([c.text for c in chunks])
        sources = [c.source for c in chunks]

        # 3. Call Claude
        system_prompt = """You are an expert on the LaunchDarkly Relay Proxy.
        Answer based ONLY on the provided context.
        Always cite the source file or doc URL.
        If you cannot verify a claim from the context, say so explicitly.
        Label estimates as estimates."""

        response = self.client.messages.create(
            model="claude-sonnet-4-6",
            system=system_prompt,
            messages=[{
                "role": "user",
                "content": f"Context:\n{context}\n\nQuestion: {question}"
            }]
        )

        return response.content[0].text, sources
```

## 3. Streamlit Page Structure

```
app.py
  |
  |-- page: "Welcome"
  |     Brief intro, start button
  |
  |-- page: "Discovery"
  |     Multi-step form (Module 1)
  |     Progress bar: Infrastructure -> Resilience -> Preferences -> Security -> Scale
  |
  |-- page: "Recommendation"
  |     Shows decisions with reasoning (Module 2)
  |     "Do you need a Relay Proxy?" card
  |     "Recommended mode" card
  |     "Redis recommendation" card
  |
  |-- page: "Architecture & Sizing"
  |     Architecture diagram (Module 3)
  |     Interactive sizing calculator (Module 4)
  |     Network requirements table
  |     Sample config viewer
  |
  |-- page: "Download Report"
  |     Preview of PDF contents
  |     Download button (Module 5)
  |
  |-- sidebar: "Ask a Question"
  |     Chat interface (Module 6)
  |     Always available on every page
```

## 4. Error Handling

- If Typst is not installed: fall back to simple Markdown report download
- If Claude API key not set: disable chat module, show message "AI Q&A requires API key"
- If client skips questions: use sensible defaults, flag low confidence in recommendation
- All decisions show "confidence: high/medium/low" based on how many questions were answered

## 5. Testing Strategy

See `/docs/test-plans/` for full test cases.

Summary:
- Unit tests for decision engine rules (deterministic, easy to test)
- Unit tests for sizing calculator (math, boundary conditions)
- Integration tests for config generation (valid TOML output)
- Snapshot tests for PDF generation (regression)
- Scenario tests: run each real client profile through the engine and verify output matches what the SA recommended
