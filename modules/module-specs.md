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
- `st.selectbox` for dropdowns (service count ranges)
- `st.text_input` for company name

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
- Three cards/panels showing each decision
- Green/yellow/red color coding for confidence
- Expandable reasoning for each decision
- "Edit answers" button to go back to discovery

**Learnings from client work:**
- The browser app question is the single most important input. It decides proxy vs daemon immediately.
- Air-gapped is a special case that overrides everything else.
- Most clients land on "proxy mode + Redis" because they want zero downtime.
- Always show the reasoning. Clients need to justify the decision to their teams.

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
- Daemon mode with Redis
- Each template has placeholders for environment names, prefixes, proxy URLs

**Learnings from client work:**
- Every client asks for a sample config file. Having one ready saves a meeting.
- The network requirements table is what security teams actually care about.
- Architecture diagrams don't need to be fancy. Text-based is fine for technical audiences.
- Always include the corporate proxy config section if internet_access is proxy-firewall.

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
- Expected peak connections (dropdown: ranges)
- Flag/segment complexity (radio: simple, moderate, complex)
- Big Segments planned (checkbox)
- Adjust memory per instance (slider: 4-16 GB, default 8)

**Output:**
- Relay Proxy sizing table
- Redis sizing table
- Total BOM table
- Notes/disclaimers on what's official vs estimated

**Learnings from client work:**
- Sean Falzon's feedback: Redis memory depends on rules/segments, not just flag count. Make this clear.
- The 2x connection buffer is from official LD docs. Always mention it.
- m4.xlarge is the reference, not a hard requirement. Say "or equivalent."
- Clients love a total BOM table they can hand to procurement.

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
4. Recommendation (relay, mode, redis with reasons)
5. Architecture diagram
6. Sizing table (with official vs estimate labels)
7. Network requirements
8. Security FAQ (only questions relevant to their compliance profile)
9. Cache and resilience overview
10. Sample configuration
11. Open items / suggested next steps
12. References (LD docs links)

**Learnings from client work:**
- The PDF is what clients actually share with their team. Make it look professional.
- Always include the references section. It builds trust.
- The executive summary should be 3-4 sentences max.
- Include "Open Items" so clients know what to figure out next.

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
