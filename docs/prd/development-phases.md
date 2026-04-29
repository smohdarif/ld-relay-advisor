# Development Phases

## GenAI SDLC Approach

Each phase follows: Plan > Spec > Build > Test > Learn > Refine

---

## Phase 1: Core Decision Flow (MVP)

**Goal:** A working app that takes client inputs and outputs a recommendation.
**Timeline estimate:** 1 week
**Dependencies:** None (pure Python + Streamlit)

### What to build:
- Module 1: Discovery Q&A form (multi-step)
- Module 2: Decision Engine (rule-based)
- Basic Streamlit layout with two pages

### What to skip:
- No PDF generation
- No AI chat
- No architecture diagrams
- No sizing calculator

### Definition of done:
- Client can fill out the form
- App shows: "You need/don't need a Relay Proxy"
- App shows: "Recommended mode: proxy/daemon/offline"
- App shows: "Redis: yes/no" with reasons
- Works on Streamlit Cloud

### Learning goals:
- Is the question flow intuitive? Do clients get stuck?
- Are the decision rules correct for the 5 known client profiles?
- What questions are missing?

---

## Phase 2: Sizing and Architecture

**Goal:** Add sizing calculator and architecture output.
**Timeline estimate:** 1 week
**Dependencies:** Phase 1 complete

### What to build:
- Module 3: Architecture Generator (text diagrams + config templates)
- Module 4: Sizing Calculator (interactive)
- Network requirements table
- Sample config viewer

### What to skip:
- No PDF yet
- No AI chat

### Definition of done:
- After recommendation, client sees architecture diagram
- Interactive sizing calculator with sliders
- Sample TOML config generated based on their answers
- Network requirements table with endpoints and ports
- Bill of materials table

### Learning goals:
- Do clients understand the sizing output?
- Is the config template useful or confusing?
- What's missing from the architecture view?

---

## Phase 3: Report Generation

**Goal:** Downloadable PDF report.
**Timeline estimate:** 1 week
**Dependencies:** Phase 2 complete, Typst installed

### What to build:
- Module 5: Report Generator
- Typst template with all sections
- Download button in Streamlit

### What to skip:
- No AI chat yet

### Definition of done:
- Client clicks "Download Report" and gets a professional PDF
- PDF contains all sections: summary, recommendation, sizing, architecture, security FAQ, config, references
- Company name and date on title page
- Official vs estimate clearly labeled throughout
- Tested with all 5 known client profiles

### Learning goals:
- Is the PDF something clients would actually share with their team?
- What sections are most/least useful?
- Does Typst compile reliably in Streamlit Cloud?

---

## Phase 4: AI Chat (RAG)

**Goal:** Add conversational Q&A powered by Claude.
**Timeline estimate:** 1-2 weeks
**Dependencies:** Phase 1 complete (chat is independent of phases 2-3), Claude API key

### What to build:
- Module 6: AI Q&A sidebar
- Knowledge base indexing (ld-relay docs + codebase + LD docs)
- RAG pipeline (embed, retrieve, generate)
- Source citations in answers

### What to skip:
- No fine-tuning
- No conversation memory across sessions

### Definition of done:
- Chat sidebar available on every page
- Can answer questions about Relay Proxy with citations
- Refuses to make up information
- Labels estimates as estimates
- Tested with the 6 common question patterns from client work

### Learning goals:
- How good are the answers compared to an SA?
- What questions does it struggle with?
- Is the knowledge base complete enough?
- Do citations help build trust?

---

## Phase 5: Polish and Deploy

**Goal:** Production-ready app.
**Timeline estimate:** 1 week
**Dependencies:** Phases 1-4 complete

### What to build:
- UI/UX refinement (colors, layout, mobile responsiveness)
- Error handling (missing Typst, missing API key, incomplete form)
- Loading states and progress indicators
- "About" page explaining the tool
- Analytics (optional): track which questions clients ask most

### What to skip:
- No user accounts
- No saved sessions
- No admin panel

### Definition of done:
- App runs smoothly on Streamlit Cloud
- Handles all error cases gracefully
- Looks professional
- Tested end-to-end with 5 client scenarios
- README with setup instructions

### Learning goals:
- Can a client use this without an SA present?
- What's the most common drop-off point?
- What feature request comes up most?

---

## Post-Launch Ideas (Future)

These are not planned but worth tracking:

1. **Multi-language support** for global clients
2. **Saved sessions** so clients can come back and adjust
3. **Comparison mode** showing proxy vs daemon side by side
4. **Integration with LD API** to pull actual flag/segment counts for more accurate sizing
5. **Slack bot** version for quick answers during client calls
6. **Template library** for different industries (finance, healthcare, retail)
7. **Architecture diagram images** instead of text (using Mermaid or D2)
