# CLAUDE.md - LD Relay Advisor

## What This Project Is

An AI-first app that guides LaunchDarkly clients through Relay Proxy decision-making. Chat-based advisor with visual flow diagrams, sizing calculator, and downloadable PDF reports. Built on Firebase + Gemini.

## Status

v2 AI-first approach finalized. 27 modules across 5 phases. Ready for Phase 1, Module 1.1.
See `docs/prd/phase-breakdown.md` for the full module breakdown.

## Planning Docs (read these first for any module)

1. `docs/prd/product-requirements.md` - Original PRD with user journey
2. `docs/prd/ai-first-approach.md` - v2 AI-first design (chat-first, tool-grounded, Firebase+Gemini)
3. `docs/prd/phase-breakdown.md` - Detailed 27-module breakdown with HLD, DLD, pseudo-logic, test cases per module
4. `docs/prd/development-phases.md` - Original 5 build phases (superseded by phase-breakdown.md)
5. `docs/hld/high-level-design.md` - System architecture overview
6. `docs/dld/detailed-design.md` - Data models, rules engine, sizing calculator
7. `docs/test-plans/test-cases.md` - 40+ test cases with real client scenarios
8. `modules/module-specs.md` - Per-module specs with effort estimates

## Research Sources (read for technical accuracy)

1. `research/field-guide-patterns.md` - Patterns from LD Field Guide (relay content pp.78-148)
2. `research/service-connection-patterns.md` - SC formulas from internal enablement deck (71 slides)
3. `research/client-journey-patterns.md` - Patterns from 5 real client engagements
4. ld-relay codebase: `/Users/arifshaikh/Documents/GitHub/ld-relay/`
5. Client files: `/Users/arifshaikh/Documents/GitHub/ld-relay/clients/`

## Per-Module Context

Each phase has its own CLAUDE.md for context:
- `docs/phases/phase-1/CLAUDE.md` - Platform + backend core context
- `docs/phases/phase-2/CLAUDE.md` - Diagrams + artifacts context
- `docs/phases/phase-3/CLAUDE.md` - Sizing + configs + PDF context
- `docs/phases/phase-4/CLAUDE.md` - Learn mode context
- `docs/phases/phase-5/CLAUDE.md` - Polish + deploy context

These are created as each phase starts.

## Source of Truth Rule

When generating any technical content about the Relay Proxy:
1. Check knowledge JSON files in `functions/knowledge/` first
2. Verify against ld-relay codebase at `/Users/arifshaikh/Documents/GitHub/ld-relay/`
3. Or reference official LD docs at https://launchdarkly.com/docs/sdk/relay-proxy/guidelines
4. If not verifiable, label as "estimate" or "recommendation based on general best practices"
5. Cite file and line number from codebase when possible

## Writing Style

These apply to ALL code, docs, and generated content:
- Human tone, not AI-generated sounding
- No em dashes. Use commas, periods, "and" instead.
- Short sentences. Plain language.
- Use analogies for complex concepts (whiteboard for cache TTL, bouncer for segments)
- No emojis unless asked

## Code Style

- TypeScript: strict mode, no `any` types, prefer interfaces over types
- Python: type hints on all function signatures, dataclasses over dicts
- Comments only where logic isn't self-evident. No obvious comments.
- Test file names match source: `rules.py` -> `test_rules.py`
- Keep functions small. One function, one job.
- No over-engineering. Simplest solution that works.
- No unused imports, variables, or dead code.

## Development Process (per module)

Every module follows this process:
1. Read the module spec in `docs/prd/phase-breakdown.md`
2. Build the code
3. Write tests (test every case listed in the module spec)
4. Run tests, fix failures
5. Write a brief learning doc explaining concepts used
6. Verify against the test cases table in the module spec
7. Do NOT skip tests. Every module spec has a test table. Every row must pass.

## Tech Stack

- Next.js 15 + Tailwind + shadcn/ui (frontend, Firebase Hosting)
- FastAPI Python backend (Cloud Functions Gen 2)
- Gemini 2.5 Pro on Vertex AI (function calling + streaming)
- Tool-grounded architecture (structured JSON knowledge files, no RAG)
- Pure Python rule engine (no ML for decisions)
- Typst for PDF generation
- Mermaid.js for interactive flow diagrams
- Firebase platform (Hosting + Cloud Functions, single deploy)
- No persistent storage (session-only, privacy by design)

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
- SDKs behind Relay Proxy: still count as service connections
- SDKs in Daemon mode: LD currently CANNOT count SCs (custom billing required)

## Client Scenarios (use for testing)

- **ARC Airlines** - On-prem, PCI, zero-downtime, separate Redis, proxy mode
- **ACI** - Payments infra, multi-deployment (vendor + customer hosted), air-gapped/offline
- **Island** - Enterprise browser, EKS/K8s (20-200 pods), TLS proxy security
- **NinjaTrader** - Ad-blockers blocking events, direct connection (no relay today)
- **Tricon Residential** - Greenfield, org structure, new to LD
