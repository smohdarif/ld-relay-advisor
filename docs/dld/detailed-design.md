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
      field-guide-relay.md        # Extracted from LD Field Guide (pp.78-148)

  templates/
    report.typ                    # Typst template for PDF report
    configs/
      proxy-mode.toml
      proxy-mode-redis.toml
      daemon-mode.toml
      daemon-mode-dynamodb.toml
      offline-mode.toml
      offline-mode-file.toml
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

class DeploymentPlatform(Enum):
    KUBERNETES = "kubernetes"
    ECS_FARGATE = "ecs-fargate"
    DOCKER = "docker"
    VM_BARE_METAL = "vm-bare-metal"

class ReverseProxy(Enum):
    NGINX = "nginx"
    HAPROXY = "haproxy"
    ALB = "alb"
    CLOUDFLARE = "cloudflare"
    OTHER = "other"
    NONE = "none"

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

class PersistentStore(Enum):
    REDIS = "redis"
    DYNAMODB = "dynamodb"
    CONSUL = "consul"
    NONE = "none"

@dataclass
class ClientProfile:
    # Company
    company_name: str = ""

    # Infrastructure
    hosting: Hosting = Hosting.ON_PREM
    internet_access: InternetAccess = InternetAccess.PROXY_FIREWALL
    deployment_platform: DeploymentPlatform = DeploymentPlatform.KUBERNETES
    reverse_proxy: ReverseProxy = ReverseProxy.NONE
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
    preferred_store: PersistentStore = PersistentStore.NONE
    existing_datastore: str = "none"   # redis, dynamodb, consul, none
    redis_shared: bool = False
    redis_compliance: str = "none"     # pci, soc2, none, unknown

    # Security
    compliance: list = field(default_factory=list)   # pci, soc2, hipaa
    security_review_needed: bool = True
    adblocker_concern: bool = False

    # Scale (split server-side vs client-side)
    server_side_connections: ConnectionScale = ConnectionScale.MEDIUM
    client_side_connections: ConnectionScale = ConnectionScale.SMALL
    container_orchestration: str = "none"  # kubernetes, ecs, docker-swarm, none
    multi_deployment: bool = False

    # Sizing extras
    segment_complexity: SegmentComplexity = SegmentComplexity.MODERATE
    big_segments_planned: bool = False
```

### 2.2 engine/rules.py

```python
# Each rule returns a tuple: (decision: bool/str, reasons: list[str])

def needs_relay_proxy(profile: ClientProfile) -> tuple[bool, list[str], list[str]]:
    """Determine if the client needs a Relay Proxy.
    Returns (decision, reasons_for, reasons_against).
    """
    reasons_for = []
    reasons_against = []

    # Reasons FOR relay
    if profile.hosting in (Hosting.ON_PREM, Hosting.HYBRID, Hosting.CUSTOMER_HOSTED):
        reasons_for.append("Applications hosted on-prem or in hybrid environment")

    if profile.internet_access != InternetAccess.DIRECT:
        reasons_for.append(f"Traffic routed through {profile.internet_access.value}")

    if profile.air_gapped:
        reasons_for.append("Air-gapped environment requires offline mode")

    if profile.service_count in (ServiceCount.LARGE, ServiceCount.XLARGE):
        reasons_for.append(f"{profile.service_count.value} services - Relay reduces outbound connections")

    if profile.adblocker_concern:
        reasons_for.append("Ad-blockers may block direct LD connections from browsers")

    if profile.infra_preference == InfraPreference.CENTRALIZED:
        reasons_for.append("Preference for centralized infrastructure aligns with Relay Proxy")

    if len(profile.compliance) > 0:
        reasons_for.append(f"Compliance requirements ({', '.join(profile.compliance)}) - Relay enables traffic inspection and TLS termination")

    # Reasons AGAINST relay (anti-patterns from Field Guide pp.82-83)
    if profile.service_count == ServiceCount.SMALL and profile.has_server_apps:
        reasons_against.append("Fewer than 10 server-side SDK instances - connection savings are minimal")

    if profile.internet_access == InternetAccess.DIRECT and not profile.air_gapped:
        reasons_against.append("SDKs can reach LaunchDarkly directly - Relay adds infrastructure complexity without clear benefit")

    if profile.has_browser_apps and not profile.has_server_apps:
        reasons_against.append("Client-side SDKs only - LaunchDarkly's CDN-backed endpoints already handle high-scale client traffic")

    if profile.client_side_connections in (ConnectionScale.LARGE, ConnectionScale.XLARGE) and not profile.has_server_apps:
        reasons_against.append("Client-side SDK streaming through Relay is 'not efficient' at scale (LD official guidance)")

    # Decision: reasons_for wins if any exist, but always surface trade-offs
    decision = len(reasons_for) > 0

    # If no strong reasons for and there are reasons against, recommend no
    if not decision and len(reasons_against) > 0:
        decision = False

    return (decision, reasons_for, reasons_against)


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

    # New: large server-side fleet + existing shared store = daemon candidate
    # (Field Guide pp.86-88)
    if (profile.service_count in (ServiceCount.LARGE, ServiceCount.XLARGE)
            and profile.existing_datastore != "none"
            and profile.infra_preference == InfraPreference.EMBEDDED):
        reasons.append(
            f"{profile.service_count.value} server-side instances with existing "
            f"{profile.existing_datastore} - daemon mode eliminates streaming connections to Relay"
        )
        return ("daemon", reasons)

    if profile.infra_preference == InfraPreference.CENTRALIZED:
        reasons.append("Centralized infrastructure preference aligns with proxy mode")

    reasons.append("Proxy mode is the safer default for most architectures")
    return ("proxy", reasons)


def needs_persistent_store(profile: ClientProfile, mode: str) -> tuple[bool, str, list[str]]:
    """Determine if a persistent store is needed and which one.
    Returns (needed, recommended_store, reasons).
    """
    reasons = []

    if mode in ("daemon", "offline"):
        reasons.append(f"{mode.capitalize()} mode requires a persistent store")
        store = profile.preferred_store.value if profile.preferred_store != PersistentStore.NONE else "redis"
        return (True, store, reasons)

    if profile.outage_tolerance == OutageTolerance.ZERO_DOWNTIME:
        reasons.append("Zero-downtime requirement - persistent store ensures flag data survives Relay restarts during LD outages")

    if profile.frequent_deployments:
        reasons.append("Frequent deployments mean new instances need flag data on startup - persistent store provides cold-start protection")

    needed = len(reasons) > 0

    # Recommend based on preference or existing infra
    if profile.preferred_store != PersistentStore.NONE:
        store = profile.preferred_store.value
    elif profile.existing_datastore != "none":
        store = profile.existing_datastore
    else:
        store = "redis"  # Redis is most common per Field Guide

    return (needed, store, reasons)


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
        "file_descriptor_note": "Each connected SDK uses one file descriptor. Ensure OS limits match expected connection count.",
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

# Scaling signals and alerting thresholds (from LD Field Guide pp.111-118)
SCALING_SIGNALS = {
    "cpu_threshold": {
        "value": "70%",
        "duration": "sustained 10 minutes",
        "severity": "medium",
        "action": "Scale horizontally - add more Relay instances",
    },
    "memory_growth": {
        "value": "20% increase in 1 hour",
        "severity": "medium",
        "action": "Investigate environment count and segment data sizes",
    },
    "connection_drop": {
        "value": "50% drop",
        "severity": "high",
        "action": "Investigate Relay health and load balancer configuration",
    },
    "env_disconnected": {
        "value": "Any environment disconnected > 5 minutes",
        "severity": "high",
        "action": "Investigate Relay connectivity to LaunchDarkly",
    },
    "event_lag": {
        "value": "Event forwarding delayed > 5 minutes",
        "severity": "medium",
        "action": "Check Relay backlog and LaunchDarkly event ingestion",
    },
    "status_unreachable": {
        "value": "Status endpoint unreachable",
        "severity": "critical",
        "action": "Relay process may be down - check process health",
    },
}

# Environment partitioning strategy (Field Guide pp.95-97)
def recommend_partitioning(profile):
    """Recommend whether to partition Relay clusters by environment tier."""
    if profile.environment_count >= 5:
        return {
            "partition": True,
            "reason": (
                f"{profile.environment_count} environments. Separate Relay clusters "
                "for production vs non-production to limit blast radius."
            ),
            "example": "Cluster A: production environments. Cluster B: staging + dev.",
        }
    return {"partition": False, "reason": "Environment count is low enough for a single cluster."}
```

### 2.4 templates/report.typ

The Typst template will be parameterized. The Python report module will render variables into the template and compile to PDF using `typst compile`.

Key sections:
1. Title page (company name, date, "Prepared by LD Relay Advisor")
2. Executive summary (1 paragraph, auto-generated from decisions)
3. Your environment (table from ClientProfile)
4. Recommendation (relay needed? mode? persistent store? with reasons FOR and AGAINST)
5. Architecture diagram
6. Sizing table
7. Network requirements
8. Reverse proxy/LB configuration checklist (conditional)
9. Deployment guidance (platform-specific)
10. Security FAQ
11. Cache and resilience overview
12. Monitoring and alerting playbook
13. Rolling deployment runbook
14. Sample configuration (TOML + AutoConfig policies)
15. Open items
16. References (LD docs + Field Guide citations)

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
  |     Progress bar: Company & Infra -> Applications -> Resilience -> Preferences -> Security -> Scale
  |
  |-- page: "Recommendation"
  |     Shows decisions with reasoning (Module 2)
  |     "Do you need a Relay Proxy?" card (with reasons FOR and AGAINST)
  |     "Recommended mode" card
  |     "Persistent store recommendation" card (Redis/DynamoDB/Consul)
  |     "Trade-offs" card (what complexity you're adding)
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
