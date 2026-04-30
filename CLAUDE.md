# CLAUDE.md - LD Relay Advisor

## What This Project Is

An AI-first Streamlit app that guides LaunchDarkly clients through Relay Proxy decision-making. From "do I need a Relay Proxy?" to a downloadable PDF with sizing and architecture.

## Status

Planning complete. No code written. Ready for Phase 1 (MVP: Discovery Q&A + Decision Engine).

## Planning Docs (read these first)

1. `docs/prd/product-requirements.md` - Full PRD with 6 modules, user journey, tech stack
2. `docs/prd/development-phases.md` - 5 build phases with GenAI SDLC approach
3. `docs/hld/high-level-design.md` - System architecture, data flow, tech decisions
4. `docs/dld/detailed-design.md` - Project structure, dataclasses, decision rules, RAG pipeline
5. `docs/test-plans/test-cases.md` - 40+ test cases with real client scenario validation
6. `modules/module-specs.md` - Per-module specs with effort estimates
7. `research/client-journey-patterns.md` - Patterns from 5 real LD client engagements

## Key Context

This app is built from real Solutions Architect experience with these clients:
- **ARC Airlines** - On-prem, PCI, zero-downtime, separate Redis, proxy mode
- **ACI** - Payments infra, multi-deployment (vendor + customer hosted), air-gapped/offline
- **Island** - Enterprise browser, EKS/K8s (20-200 pods), TLS proxy security
- **NinjaTrader** - Ad-blockers blocking events, direct connection (no relay today)
- **Tricon Residential** - Greenfield, org structure, new to LD

Client files are at: `/Users/arifshaikh/Documents/GitHub/ld-relay/clients/`

## Source of Truth Rule

When generating any technical content about the Relay Proxy:
1. Verify against the ld-relay codebase at `/Users/arifshaikh/Documents/GitHub/ld-relay/`
2. Or reference official LD docs at https://launchdarkly.com/docs/sdk/relay-proxy/guidelines
3. If not verifiable, label as "estimate" or "recommendation based on general best practices"
4. Cite file and line number from codebase when possible

## Writing Style

- Human tone, not AI-generated sounding
- No em dashes. Use commas, periods, "and" instead.
- Short sentences. Plain language.
- Use analogies for complex concepts (whiteboard for cache TTL, bouncer for segments)
- No emojis unless asked

## Tech Stack

- Next.js 15 + Tailwind + shadcn/ui (frontend, Firebase Hosting)
- FastAPI Python backend (Cloud Functions Gen 2)
- Gemini 2.5 Pro on Vertex AI (function calling + streaming)
- Tool-grounded architecture (structured JSON knowledge files, no RAG)
- Pure Python rule engine (no ML for decisions)
- Typst for PDF generation (proven in ARC Airlines workflow)
- Mermaid.js for interactive flow diagrams
- Firebase platform (Hosting + Cloud Functions, single deploy)
- No persistent storage (session-only, privacy by design)

## Build Order

Phase 1: Modules 1+2 (Discovery + Decision Engine) - start here
Phase 2: Modules 3+4 (Architecture + Sizing)
Phase 3: Module 5 (Report/PDF)
Phase 4: Module 6 (AI Chat/RAG)
Phase 5: Polish + Deploy

## Key Official LD Numbers (verified)

- Tested instance: m4.xlarge (4 vCPU, 16 GiB RAM)
- Minimum instances: 3 across 2+ availability zones
- Idle memory: ~11 MiB
- Network bandwidth: most important resource
- Provision for 2x expected connections
- Client-side SDK streaming: "not efficient" on Relay Proxy (monitor and scale)
- Default cache TTL: 30s, but LD recommends infinite (-1)
- Default port: 8030
- Default heartbeat: 3 minutes
- sdk.launchdarkly.com: Big Segments polling ONLY (not general flag polling)
