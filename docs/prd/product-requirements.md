# Product Requirements Document (PRD)

## LD Relay Advisor

**Version:** 1.0
**Author:** Arif Shaikh
**Date:** April 2026
**Status:** Draft

---

## 1. Problem Statement

Every LaunchDarkly client evaluating the Relay Proxy goes through the same decision-making process. Solutions Architects spend hours per client repeating the same discovery questions, building custom sizing docs, and explaining proxy vs daemon mode. The knowledge exists but is locked in individual SA brains, scattered client folders, and the ld-relay codebase.

Clients need answers to:
- Do I need a Relay Proxy?
- Which mode should I use?
- How should I deploy it?
- How do I size the infrastructure?
- What about Redis, security, resilience?

Today this takes 2-4 meetings and custom document creation per client. It should take 30 minutes of self-service.

## 2. Target Users

### Primary: LaunchDarkly Clients (DevOps/Platform Engineers)
- Evaluating whether to adopt Relay Proxy
- Need to present a recommendation to their security/infra team
- Want a downloadable report they can share internally

### Secondary: LaunchDarkly Solutions Architects
- Use the app during client calls for live recommendations
- Generate sizing docs faster
- Ensure consistency across client engagements

## 3. Success Metrics

- Time from "considering Relay Proxy" to "architecture recommendation": from weeks to 30 minutes
- Reduction in SA prep time per client: from 4-8 hours to under 1 hour
- Client self-service rate: 60% of clients can get a recommendation without an SA call
- Accuracy: recommendations match what an experienced SA would advise in 90%+ of cases

## 4. User Journey / App Flow

### Module 1: Discovery (Q&A Flow)

Structured questionnaire that captures the client's environment. Questions derived from real client patterns:

**Infrastructure Questions:**
1. Where are your applications hosted? (on-prem, AWS, Azure, GCP, hybrid, customer-hosted)
2. Do your applications have direct outbound internet access, or is traffic routed through proxies/firewalls?
3. Do you have browser/mobile/client-side applications that will use LaunchDarkly?
4. Do you have server-side applications? What languages/frameworks?
5. Approximately how many services/applications will connect to LaunchDarkly?
6. How many LaunchDarkly environments do you plan to use? (dev, staging, prod, etc.)

**Resilience Questions:**
7. In the event of a LaunchDarkly outage, what is your expectation? (continue with cached values, acceptable downtime, etc.)
8. Do you deploy new application instances frequently? (relevant for cold-start resilience)
9. Do you have air-gapped or offline environments?

**Infrastructure Preferences:**
10. Do you prefer centralized shared infrastructure or embedded per-service dependencies?
11. Do you already run Redis, DynamoDB, or Consul in your environment?
12. If you use Redis, is it shared with other workloads (e.g., sessions)? Any compliance constraints on it?

**Security Questions:**
13. Are there specific security or compliance requirements? (PCI, SOC2, HIPAA, IP allowlisting)
14. Does your security team need to review outbound connection models?
15. Are ad-blockers or content filters a concern for your browser applications?

**Scale Questions:**
16. What is your expected peak number of concurrent SDK connections?
17. Do you use container orchestration? (Kubernetes, ECS, Docker Swarm)
18. Do you need to support multiple deployment models? (e.g., vendor-hosted + customer-hosted)

### Module 2: Decision Engine

Based on Module 1 answers, the app produces:

**Decision 1: Do you need a Relay Proxy?**

Flowchart logic:
- On-prem or no direct internet? -> Yes, Relay Proxy
- Strict firewall/proxy rules? -> Yes, Relay Proxy
- Want to reduce outbound connections (many services)? -> Yes, Relay Proxy
- Ad-blocker issues with browser SDKs? -> Yes, Relay Proxy can help
- Air-gapped/offline? -> Yes, Relay Proxy in offline mode (Enterprise)
- Direct cloud access + few services + no special requirements? -> Probably not needed

**Decision 2: Which mode?**

- Has browser/mobile/client-side apps? -> Proxy mode
- Server-side only + want DB-backed resilience? -> Daemon mode is an option
- Server-side only + PHP? -> Daemon mode preferred
- Air-gapped? -> Offline mode (Enterprise)
- Mixed frontend + backend? -> Proxy mode

**Decision 3: Do you need Redis?**

- Want zero-downtime during LD outages? -> Yes
- Deploy new instances frequently? -> Yes (cold-start protection)
- Just want basic caching? -> Optional, Relay's in-memory cache is enough
- Using daemon mode? -> Required (not optional)

### Module 3: Architecture Generator

Based on decisions, generates:
- Architecture diagram (text-based or image)
- Deployment topology (number of instances, LB, Redis)
- Network requirements (outbound endpoints, ports)
- Configuration template (sample relay config file)

### Module 4: Sizing Calculator

Interactive calculator:
- Relay Proxy instances: based on LD official guidance (min 3, m4.xlarge reference)
- Redis sizing: based on environment count, segment complexity assumptions
- Load balancer requirements
- Network bandwidth estimates
- Total bill of materials

All numbers clearly labeled as "LD official" or "estimate based on general best practices."

### Module 5: Report Generator

Produces a downloadable PDF containing:
- Executive summary
- Architecture recommendation with rationale
- Sizing table
- Network requirements
- Configuration template
- Security FAQ (customized to their compliance needs)
- Open items / next steps
- References to official LD docs

### Module 6: AI Q&A (Chat)

A chat interface powered by RAG that can answer follow-up questions like:
- "What happens if Redis goes down during an LD outage?"
- "Can I use a separate Redis logical DB?"
- "What does the cache TTL setting do?"
- "How do Big Segments work?"

Knowledge base:
- ld-relay codebase (Go source, docs, README)
- Official LD docs (relay proxy guidelines, persistent storage, configuration)
- Client engagement learnings (anonymized patterns)

## 5. Non-Functional Requirements

- **Accuracy:** All technical claims must be traceable to the ld-relay codebase or official LD docs. Estimates must be clearly labeled.
- **Simplicity:** A non-technical stakeholder should be able to follow the Q&A flow.
- **Speed:** Full flow from start to PDF download in under 30 minutes.
- **Privacy:** No client data stored server-side. All inputs processed in-session only.
- **Offline capability:** The app should work without calling external AI APIs if needed (pre-built decision trees for core flow).

## 6. What This App is NOT

- Not a replacement for SA engagement on complex/custom architectures
- Not a monitoring or deployment tool
- Not a configuration management tool
- Not an official LaunchDarkly product (internal/field tool)

## 7. Tech Stack (Proposed)

- **Frontend:** Streamlit (Python)
- **AI/LLM:** Claude API for the Q&A chat module
- **RAG:** Embeddings over ld-relay codebase + LD docs
- **PDF Generation:** Typst (already proven in our workflow) or ReportLab
- **Deployment:** Streamlit Cloud or internal hosting
- **Data:** No persistent storage needed. Session-only.

## 8. Development Phases

### Phase 1: Core Decision Flow (MVP)
- Modules 1 + 2: Discovery Q&A + Decision Engine
- Text-based output (no PDF yet)
- Static decision tree, no AI

### Phase 2: Sizing and Architecture
- Modules 3 + 4: Architecture Generator + Sizing Calculator
- Interactive sizing inputs
- Sample config generation

### Phase 3: Report Generation
- Module 5: PDF report generation
- Branded, professional output
- Downloadable

### Phase 4: AI Chat
- Module 6: RAG-powered Q&A
- Claude API integration
- Knowledge base from codebase + docs

### Phase 5: Polish and Deploy
- UI/UX refinement
- Testing with real client scenarios
- Deployment to Streamlit Cloud or internal hosting
