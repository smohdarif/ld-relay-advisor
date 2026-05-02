# Detailed Phase Breakdown

**Version:** 1.0
**Date:** May 2026

Each phase is broken into small, self-contained modules. Each module has its own deliverables:
- Design doc (HLD + DLD)
- Pseudo-logic
- Test cases and test scripts
- Learning doc (concepts and tech used)
- CLAUDE.md context file

---

## Phase 1: Platform and Backend Core

**Goal:** Firebase project running, FastAPI backend deployed, Gemini talking, knowledge files serving facts.
**Why this first:** Everything else depends on the backend being live and the AI being grounded.

### Module 1.1: Firebase Project Setup

**What:** Initialize Firebase project, configure hosting and Cloud Functions Gen 2, deploy a hello-world endpoint.

**Deliverables:**
- `firebase.json`, `.firebaserc`
- `functions/main.py` (hello-world FastAPI)
- `functions/requirements.txt`
- `functions/Dockerfile`
- Working `firebase deploy` that serves a health check at `/api/health`

**HLD:**
```
Firebase Hosting (CDN) --> rewrites /api/* --> Cloud Function Gen 2 (FastAPI)
```

**DLD:**
- Firebase Hosting: static file serving, SPA rewrite rules
- Cloud Functions Gen 2: Docker container running Python 3.12 + FastAPI
- CORS config: allow frontend origin
- Environment: GCP project ID, region (us-central1)

**Pseudo-logic:**
```
1. firebase init (hosting + functions)
2. Create FastAPI app with /api/health endpoint
3. Configure firebase.json rewrites
4. firebase deploy
5. Verify /api/health returns 200
```

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 1.1.1 | GET /api/health | Returns `{"status": "ok"}` |
| 1.1.2 | GET / (frontend) | Returns placeholder HTML |
| 1.1.3 | Deploy to Firebase | Succeeds without errors |
| 1.1.4 | CORS from frontend origin | Allowed |
| 1.1.5 | CORS from unknown origin | Blocked |

**Learning doc topics:**
- What is Firebase Hosting and how does CDN rewrite work
- Cloud Functions Gen 2 vs Gen 1 (Gen 2 = Cloud Run containers)
- FastAPI basics: app creation, route definition, ASGI
- How firebase.json rewrites route traffic

---

### Module 1.2: Data Models

**What:** Define all Python dataclasses that flow through the system: ClientProfile, Recommendation, SizingResult.

**Deliverables:**
- `functions/models/client_profile.py`
- `functions/models/recommendation.py`
- `functions/models/sizing_result.py`
- `functions/models/__init__.py`
- Unit tests for serialization/deserialization

**HLD:**
```
ClientProfile --> Rule Engine --> Recommendation
ClientProfile --> Sizing Calculator --> SizingResult
```

**DLD:**
- ClientProfile: 25+ fields with enums (Hosting, InternetAccess, DeploymentPlatform, etc.)
- Recommendation: needs_relay (bool), reasons_for (list), reasons_against (list), mode (str), persistent_store (str), confidence (str), caveats (list)
- SizingResult: relay_instances, relay_cpu, relay_memory, store_config, sc_estimate, scaling_signals

**Pseudo-logic:**
```python
# ClientProfile enums
Hosting = on-prem | aws | azure | gcp | hybrid | customer-hosted
InternetAccess = direct | proxy-firewall | air-gapped
DeploymentPlatform = kubernetes | ecs-fargate | docker | vm-bare-metal
ReverseProxy = nginx | haproxy | alb | cloudflare | other | none
ServiceCount = <10 | 10-50 | 50-200 | 200+
ConnectionScale = <100 | 100-1000 | 1000-10000 | 10000+
PersistentStore = redis | dynamodb | consul | none

# ClientProfile
@dataclass with defaults for all fields
JSON serializable (to/from dict for Gemini tool calls)

# Recommendation
@dataclass with reasons_for, reasons_against, trade_offs
confidence derived from how many profile fields are populated

# SizingResult
@dataclass with official vs estimate labels on every number
```

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 1.2.1 | Create ClientProfile with defaults | All fields have sensible defaults |
| 1.2.2 | Serialize ClientProfile to dict | Valid JSON-compatible dict |
| 1.2.3 | Deserialize partial dict to ClientProfile | Missing fields use defaults |
| 1.2.4 | Recommendation confidence with 5/25 fields | "low" |
| 1.2.5 | Recommendation confidence with 15/25 fields | "medium" |
| 1.2.6 | Recommendation confidence with 22/25 fields | "high" |
| 1.2.7 | SizingResult labels official vs estimate | Each number has is_official flag |

**Learning doc topics:**
- Python dataclasses and field defaults
- Enums in Python (when to use vs plain strings)
- Serialization patterns (dataclass to dict, dict to dataclass)
- Why we separate models from logic (clean architecture)

---

### Module 1.3: Knowledge Files

**What:** Create all 13 structured JSON knowledge files with verified facts and source citations.

**Deliverables:**
- `functions/knowledge/config-defaults.json`
- `functions/knowledge/sizing-numbers.json`
- `functions/knowledge/mode-behaviors.json`
- `functions/knowledge/failure-scenarios.json`
- `functions/knowledge/network-requirements.json`
- `functions/knowledge/cache-behavior.json`
- `functions/knowledge/reverse-proxy-settings.json`
- `functions/knowledge/monitoring-thresholds.json`
- `functions/knowledge/sc-formulas.json`
- `functions/knowledge/anti-patterns.json`
- `functions/knowledge/deployment-guidance.json`
- `functions/knowledge/autoconfig-examples.json`
- `functions/knowledge/faq.json`
- Validation script that checks every entry has a `source` field

**HLD:**
```
Knowledge Files (JSON) --> Grounding Tools (Python) --> Gemini Function Calls
```

**DLD:**
- Each file is a JSON object with a top-level array of entries
- Every entry has: name, label, description/content, source (citation)
- Sources reference: Field Guide page, SC deck slide, ld-relay file:line, LD docs URL
- Files are read-only at runtime, loaded once into memory

**Pseudo-logic:**
```
For each knowledge domain:
  1. Extract facts from research/ docs (field-guide-patterns.md, service-connection-patterns.md)
  2. Verify each fact against ld-relay codebase or official LD docs
  3. Structure as JSON with: name, content, source, official (bool)
  4. Run validation: every entry must have non-empty source
```

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 1.3.1 | All 13 JSON files parse without errors | Valid JSON |
| 1.3.2 | Every entry in every file has `source` field | No empty sources |
| 1.3.3 | config-defaults.json has cache_ttl entry | Default 30s, recommended -1 |
| 1.3.4 | sizing-numbers.json has m4.xlarge reference | 4 vCPU, 16 GiB, official=true |
| 1.3.5 | sc-formulas.json has all 7 patterns | typical, forking, containerized, etc. |
| 1.3.6 | anti-patterns.json has all 6 anti-patterns | multiple_sdk, relay_redundancy, etc. |
| 1.3.7 | mode-behaviors.json covers proxy, daemon, offline | All three modes present |
| 1.3.8 | monitoring-thresholds.json has severity levels | high, medium, critical present |
| 1.3.9 | Cross-reference: official numbers match CLAUDE.md | No contradictions |

**Learning doc topics:**
- Why structured JSON over a database (bounded domain, auditable, no query overhead)
- Source citation patterns (how to reference codebase, docs, internal decks)
- JSON schema validation basics
- How tool-grounding prevents hallucination (comparison with RAG)

---

### Module 1.4: Grounding Tools

**What:** Python functions that read knowledge files and return verified facts. These become Gemini's function declarations.

**Deliverables:**
- `functions/ai/tools.py` (13 grounding tool implementations)
- `functions/ai/tool_definitions.py` (Gemini FunctionDeclaration schemas)
- `functions/knowledge/loader.py` (loads JSON files, provides lookup)
- Unit tests for every tool

**HLD:**
```
Gemini calls function --> tool_definitions.py matches --> tools.py executes --> loader.py reads JSON --> returns fact + citation
```

**DLD:**
- `loader.py`: Loads all JSON files on startup. Provides `lookup(file, key)` and `search(file, query)`.
- `tools.py`: 13 grounding functions + 6 action functions = 19 total. Each grounding function returns `{"fact": ..., "source": ..., "official": bool}`.
- `tool_definitions.py`: Gemini `FunctionDeclaration` objects matching each tool's parameters.

**Pseudo-logic:**
```python
# Grounding tools (return verified facts)
def get_config_defaults(setting: str) -> dict:
    return loader.lookup("config-defaults", setting)

def get_mode_behavior(mode: str) -> dict:
    return loader.lookup("mode-behaviors", mode)

def get_sc_formula(pattern: str) -> dict:
    return loader.lookup("sc-formulas", pattern)

def get_anti_pattern(pattern: str) -> dict:
    return loader.search("anti-patterns", pattern)

def get_failure_behavior(scenario: str) -> dict:
    return loader.lookup("failure-scenarios", scenario)

def get_sizing_guidance(resource: str) -> dict:
    return loader.lookup("sizing-numbers", resource)

def get_network_requirements(mode: str) -> dict:
    return loader.lookup("network-requirements", mode)

def get_reverse_proxy_checklist(proxy_type: str) -> dict:
    return loader.lookup("reverse-proxy-settings", proxy_type)

def get_monitoring_thresholds() -> dict:
    return loader.get_all("monitoring-thresholds")

def get_cache_behavior(ttl_setting: str) -> dict:
    return loader.lookup("cache-behavior", ttl_setting)

def get_deployment_guidance(platform: str) -> dict:
    return loader.lookup("deployment-guidance", platform)

def get_autoconfig_examples(scope: str) -> dict:
    return loader.lookup("autoconfig-examples", scope)

def lookup_faq(topic: str) -> dict:
    return loader.search("faq", topic)
```

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 1.4.1 | get_config_defaults("cache_ttl") | Returns default=30s, recommended=-1, source present |
| 1.4.2 | get_config_defaults("nonexistent") | Returns "not found" gracefully |
| 1.4.3 | get_mode_behavior("proxy") | Returns proxy mode behavior with source |
| 1.4.4 | get_mode_behavior("daemon") | Returns daemon mode including SC billing warning |
| 1.4.5 | get_sc_formula("forking") | Returns formula: hosts * forks, source: SC deck |
| 1.4.6 | get_anti_pattern("relay_for_redundancy_only") | Returns warning with correct approach |
| 1.4.7 | get_failure_behavior("ld_down") | Returns cache serving behavior |
| 1.4.8 | get_monitoring_thresholds() | Returns all 6 threshold entries |
| 1.4.9 | get_reverse_proxy_checklist("nginx") | Returns SSE config checklist |
| 1.4.10 | All 13 tools return `source` field | No response without citation |
| 1.4.11 | Tool definitions match tool implementations | Same parameter names and types |

**Learning doc topics:**
- Gemini function calling: FunctionDeclaration schema format
- How tool-grounding works (Gemini calls function, gets fact, cites it)
- Python module loading patterns (load JSON once, serve from memory)
- Why every tool returns a source citation

---

### Module 1.5: Rule Engine

**What:** Deterministic decision logic. Takes ClientProfile, returns Recommendation.

**Deliverables:**
- `functions/engine/rules.py`
- Unit tests covering all decision paths
- Test with 5 real client scenarios

**HLD:**
```
ClientProfile --> needs_relay_proxy() --> reasons_for, reasons_against
ClientProfile --> recommend_mode() --> proxy | daemon | offline
ClientProfile, mode --> needs_persistent_store() --> store type, reasons
ClientProfile --> redis_recommendations() --> separate, persistence, eviction
```

**DLD:**
- `needs_relay_proxy()`: Returns (bool, reasons_for: list, reasons_against: list)
- `recommend_mode()`: Returns (mode: str, reasons: list)
- `needs_persistent_store()`: Returns (needed: bool, store: str, reasons: list)
- `redis_recommendations()`: Returns dict with separate_instance, persistence, eviction, cache_ttl
- Anti-pattern detection integrated into each function
- Confidence calculation based on populated field count

**Pseudo-logic:**
```
needs_relay_proxy(profile):
  reasons_for = []
  reasons_against = []

  # FOR
  if hosting in (on-prem, hybrid, customer-hosted): add reason
  if internet_access != direct: add reason
  if air_gapped: add reason
  if service_count >= 50: add reason
  if adblocker_concern: add reason
  if compliance not empty: add reason

  # AGAINST
  if service_count < 10 and has_server_apps: add reason
  if internet_access == direct and not air_gapped: add reason
  if has_browser_apps and not has_server_apps: add reason (client-only)

  return (len(reasons_for) > 0, reasons_for, reasons_against)

recommend_mode(profile):
  if air_gapped: return offline
  if has_browser_apps: return proxy
  if php in server_languages: return daemon
  if service_count >= 50 and existing_datastore != none and preference == embedded: return daemon
  return proxy (default)

needs_persistent_store(profile, mode):
  if mode in (daemon, offline): required
  if outage_tolerance == zero-downtime: recommended
  if frequent_deployments: recommended
  store = preferred_store or existing_datastore or redis
  return (needed, store, reasons)
```

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 1.5.1 | ARC Airlines profile | relay=yes, mode=proxy, store=redis (separate), confidence=high |
| 1.5.2 | ACI profile | relay=yes, mode=proxy (vendor) + offline (air-gapped), store=redis |
| 1.5.3 | Island profile | relay=yes, mode=proxy, store=redis, K8s deployment |
| 1.5.4 | NinjaTrader profile | relay=maybe (ad-blocker only), reasons_against present |
| 1.5.5 | Tricon profile | relay=no (greenfield, direct access, few services) |
| 1.5.6 | Air-gapped profile | mode=offline, store=required |
| 1.5.7 | PHP-only profile | mode=daemon |
| 1.5.8 | Large fleet + existing Redis + embedded pref | mode=daemon |
| 1.5.9 | <10 services, direct internet, no compliance | reasons_against > reasons_for |
| 1.5.10 | Browser-only, no server apps | reasons_against includes "client-side only" |
| 1.5.11 | All fields empty (defaults) | confidence=low, returns sensible default |
| 1.5.12 | Shared Redis with PCI | separate_instance=true, persistence=true |

**Learning doc topics:**
- Rule engine vs ML (why deterministic logic is better for bounded decisions)
- Decision tree patterns in Python
- How to test decision logic (scenario-based testing)
- Confidence scoring (field completion ratio)

---

### Module 1.6: Gemini Client

**What:** Vertex AI Gemini client wrapper with function calling and streaming.

**Deliverables:**
- `functions/ai/gemini_client.py`
- `functions/ai/system_prompt.py`
- Integration tests with Vertex AI

**HLD:**
```
FastAPI chat endpoint --> GeminiClient --> Vertex AI --> function call --> tools.py --> response
```

**DLD:**
- `GeminiClient`: Wraps `vertexai.generative_models.GenerativeModel`
- Initializes with system prompt + all tool declarations
- `chat()` method: sends message, handles function calls in a loop, streams text
- Function call loop: Gemini returns function_call part, we execute the tool, send result back, Gemini continues
- Conversation history maintained per session (passed from frontend)

**Pseudo-logic:**
```python
class GeminiClient:
    def __init__(self):
        aiplatform.init(project=PROJECT_ID, location=REGION)
        self.model = GenerativeModel(
            "gemini-2.5-pro",
            system_instruction=SYSTEM_PROMPT,
            tools=[relay_tools],
        )

    async def chat(self, messages: list, on_text: callback):
        chat = self.model.start_chat(history=messages[:-1])
        response = chat.send_message(messages[-1], stream=True)

        while True:
            for chunk in response:
                for part in chunk.parts:
                    if part.function_call:
                        result = execute_tool(part.function_call)
                        response = chat.send_message(
                            Part.from_function_response(result),
                            stream=True
                        )
                        break  # restart chunk loop with new response
                    elif part.text:
                        on_text(part.text)
            else:
                break  # no more function calls, done
```

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 1.6.1 | Send "hello" | Returns greeting, no function calls |
| 1.6.2 | Ask "what's the default cache TTL?" | Calls get_config_defaults, cites source |
| 1.6.3 | Ask about proxy mode | Calls get_mode_behavior("proxy") |
| 1.6.4 | Describe infrastructure | Calls update_client_profile |
| 1.6.5 | Ask for recommendation | Calls run_decision_engine |
| 1.6.6 | Ask about anti-pattern | Calls get_anti_pattern |
| 1.6.7 | Response streams text chunks | on_text callback fires multiple times |
| 1.6.8 | Multiple function calls in one turn | All executed, results sent back |
| 1.6.9 | Conversation history preserved | Second message references first |
| 1.6.10 | System prompt enforced | Never states facts without tool call |

**Learning doc topics:**
- Vertex AI SDK: GenerativeModel, start_chat, streaming
- Gemini function calling: how the request/response loop works
- Streaming responses: SSE pattern, chunk handling
- System instructions vs user messages
- How to test AI integrations (mock vs live)

---

### Module 1.7: Chat API Endpoint

**What:** FastAPI streaming endpoint that connects the frontend to Gemini.

**Deliverables:**
- `functions/routers/chat.py`
- SSE streaming response
- Session state management (conversation history)

**HLD:**
```
Frontend (POST /api/chat) --> FastAPI --> GeminiClient --> SSE stream back to frontend
```

**DLD:**
- POST `/api/chat` with JSON body: `{ messages: [...], profile: {...} }`
- Returns SSE (Server-Sent Events) stream
- Each SSE event is either: `text` (token), `tool_call` (function name + args), `tool_result` (function output), `artifact` (diagram, sizing table, etc.), `profile_update` (updated ClientProfile), `done`
- Profile state passed from frontend each request (no server-side sessions)

**Pseudo-logic:**
```python
@router.post("/api/chat")
async def chat(request: ChatRequest):
    async def generate():
        async for event in gemini_client.chat(
            messages=request.messages,
            profile=request.profile,
        ):
            if event.type == "text":
                yield f"data: {json.dumps({'type': 'text', 'content': event.text})}\n\n"
            elif event.type == "tool_call":
                yield f"data: {json.dumps({'type': 'tool_call', 'name': event.name, 'args': event.args})}\n\n"
            elif event.type == "artifact":
                yield f"data: {json.dumps({'type': 'artifact', 'artifact': event.data})}\n\n"
            elif event.type == "profile_update":
                yield f"data: {json.dumps({'type': 'profile_update', 'profile': event.profile})}\n\n"
        yield f"data: {json.dumps({'type': 'done'})}\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 1.7.1 | POST /api/chat with simple message | Returns SSE stream with text events |
| 1.7.2 | SSE events are valid JSON | Every `data:` line parses |
| 1.7.3 | Stream ends with `done` event | Last event type is "done" |
| 1.7.4 | Tool calls appear in stream | tool_call events with name and args |
| 1.7.5 | Profile update appears in stream | profile_update event with updated fields |
| 1.7.6 | Empty messages array | Returns error 400 |
| 1.7.7 | Malformed request body | Returns error 422 |
| 1.7.8 | Concurrent requests | Each gets its own stream |
| 1.7.9 | Large conversation history (20+ messages) | Still works within timeout |

**Learning doc topics:**
- Server-Sent Events (SSE): protocol, format, browser handling
- FastAPI StreamingResponse: async generators
- Why SSE over WebSockets for this use case (simpler, HTTP-compatible, Firebase-friendly)
- Conversation state: stateless backend with frontend-passed history

---

### Module 1.8: Basic Chat UI

**What:** Next.js frontend with a working chat interface that streams responses from the backend.

**Deliverables:**
- `frontend/` Next.js 15 app scaffold
- `frontend/app/layout.tsx` (root layout)
- `frontend/app/page.tsx` (landing page with mode selector)
- `frontend/app/advise/page.tsx` (chat page)
- `frontend/components/chat/ChatPanel.tsx`
- `frontend/components/chat/MessageBubble.tsx`
- `frontend/components/chat/QuickActions.tsx`
- `frontend/lib/api.ts` (SSE client)
- `frontend/lib/types.ts` (TypeScript types)

**HLD:**
```
User types message --> ChatPanel --> POST /api/chat --> SSE stream --> MessageBubble renders tokens
```

**DLD:**
- Vercel AI SDK (`ai` package) for chat state management
- Custom SSE handler that parses our event types (text, tool_call, artifact, profile_update, done)
- Messages stored in React state (no persistence)
- Quick action buttons send pre-built prompts
- Tailwind + shadcn/ui for styling

**Pseudo-logic:**
```typescript
// ChatPanel.tsx
const { messages, input, handleSubmit, isLoading } = useChat({
  api: '/api/chat',
  body: { profile: currentProfile },
  onToolCall: (toolCall) => {
    // Show tool call in chat (collapsible)
  },
  onFinish: (message) => {
    // Update profile if profile_update event received
  },
});

return (
  <div className="flex flex-col h-full">
    <MessageList messages={messages} />
    <QuickActions onSelect={sendQuickAction} />
    <InputBar value={input} onSubmit={handleSubmit} loading={isLoading} />
  </div>
);
```

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 1.8.1 | Page loads at /advise | Chat panel renders |
| 1.8.2 | Type message and send | Message appears in chat, response streams |
| 1.8.3 | Quick action button click | Sends pre-built prompt |
| 1.8.4 | Loading state during response | Input disabled, spinner shown |
| 1.8.5 | Long response | Streams incrementally, auto-scrolls |
| 1.8.6 | Tool call in response | Shows tool call indicator (collapsible) |
| 1.8.7 | Multiple messages | Conversation history maintained |
| 1.8.8 | Empty input submit | Blocked |
| 1.8.9 | Mobile viewport | Responsive layout |

**Learning doc topics:**
- Next.js 15 App Router: layout, page, routing
- Vercel AI SDK: useChat hook, streaming, custom providers
- SSE consumption in the browser
- shadcn/ui: component library, Tailwind integration
- React state management for chat interfaces

---

## Phase 1 Completion Criteria

When all 8 modules are done:
- `firebase deploy` works (frontend + backend live)
- User chats at /advise, gets streaming responses
- Gemini calls grounding tools for technical facts (verified by checking SSE events)
- Rule engine produces correct recommendations for all 5 client scenarios
- Every technical fact in responses has a source citation
- Quick actions work: "Do I need a Relay Proxy?", "Compare modes"

---

## Phase 2: Diagrams and Artifacts Panel

**Goal:** Mermaid diagrams render inline in chat. Artifact panel accumulates outputs on the right side.

### Module 2.1: Mermaid Renderer

**What:** React component that renders Mermaid diagram code into SVG.

**Deliverables:**
- `frontend/components/diagrams/MermaidRenderer.tsx`
- Mermaid.js integration (client-side rendering)
- Dark mode support
- Click-to-expand for large diagrams

**HLD:**
```
Mermaid code string --> MermaidRenderer --> SVG rendered in DOM
```

**DLD:**
- Uses `mermaid` npm package
- Renders on mount and when code changes
- Error handling: if Mermaid syntax is invalid, show code block as fallback
- Responsive: scales to container width
- Theme: matches app dark/light mode

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 2.1.1 | Valid Mermaid flowchart | Renders SVG |
| 2.1.2 | Valid Mermaid sequence diagram | Renders SVG |
| 2.1.3 | Invalid Mermaid syntax | Shows code block fallback |
| 2.1.4 | Empty string | Renders nothing |
| 2.1.5 | Dark mode | Diagram uses dark theme |
| 2.1.6 | Click to expand | Opens modal with full-size diagram |
| 2.1.7 | Re-render on code change | Updates SVG |

**Learning doc topics:**
- Mermaid.js: syntax for flowcharts, sequence diagrams, graph LR/TD
- Client-side rendering vs server-side for diagrams
- React useEffect for DOM manipulation (Mermaid needs a DOM node)
- SVG in React: sizing, responsiveness

---

### Module 2.2: Diagram Generator (Backend)

**What:** Backend tool that generates Mermaid code for Relay Proxy diagrams based on context.

**Deliverables:**
- `functions/engine/diagram_generator.py`
- 9 diagram template functions
- Unit tests for each diagram type

**HLD:**
```
Gemini calls generate_diagram(type, context) --> diagram_generator.py --> returns Mermaid code string
```

**DLD:**
Diagram types and their generators:
1. `proxy_flow()` - Proxy mode data flow
2. `daemon_flow()` - Daemon mode data flow
3. `offline_flow()` - Offline mode data flow
4. `architecture(profile)` - Customized architecture topology (adapts to hosting, platform, store)
5. `failure_scenario(scenario)` - What happens when X goes down
6. `caching_layers()` - 4-layer caching visualization
7. `connection_model()` - Server-side vs client-side vs with Relay
8. `sc_pattern(pattern)` - Service connection pattern visualization
9. `mode_comparison(mode_a, mode_b)` - Side-by-side mode comparison

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 2.2.1 | proxy_flow() | Valid Mermaid with SDK, Relay, LD, Redis nodes |
| 2.2.2 | daemon_flow() | Valid Mermaid with SDK reading from store directly |
| 2.2.3 | architecture(aws_k8s_profile) | Includes K8s pods, ALB, Relay deployment |
| 2.2.4 | architecture(onprem_profile) | Includes firewall, on-prem hosts |
| 2.2.5 | failure_scenario("ld_down") | Multi-step diagram showing cache serving |
| 2.2.6 | sc_pattern("forking") | Shows hosts * forks multiplication |
| 2.2.7 | mode_comparison("proxy", "daemon") | Side-by-side subgraphs |
| 2.2.8 | All diagram outputs | Parse without Mermaid syntax errors |

**Learning doc topics:**
- Mermaid syntax: graph, subgraph, styling, links
- Template-based code generation (string interpolation for diagrams)
- How to test generated diagram code (parse with mermaid CLI)

---

### Module 2.3: Inline Diagram Rendering in Chat

**What:** When Gemini returns a diagram via tool call, render it inline in the chat message.

**Deliverables:**
- Updated `MessageBubble.tsx` to detect and render diagram artifacts
- `frontend/components/chat/ArtifactRenderer.tsx` (dispatches to correct renderer)

**DLD:**
- When SSE event type is `artifact` with subtype `diagram`, extract Mermaid code
- Render using MermaidRenderer inside the message bubble
- Diagram appears where it would naturally go in the conversation

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 2.3.1 | AI explains proxy mode | Diagram renders inline after explanation text |
| 2.3.2 | AI compares modes | Two diagrams render side by side |
| 2.3.3 | AI shows architecture | Custom architecture diagram renders |
| 2.3.4 | Multiple diagrams in one response | All render in order |

**Learning doc topics:**
- Rendering rich content in chat (beyond plain text)
- React component composition (MessageBubble containing MermaidRenderer)
- SSE artifact events and how to dispatch rendering

---

### Module 2.4: Artifact Panel

**What:** Collapsible right-side panel that accumulates artifacts as the conversation progresses.

**Deliverables:**
- `frontend/components/chat/ArtifactPanel.tsx`
- `frontend/components/artifacts/` (SizingTable, ConfigViewer, etc. - placeholder shells)
- Resizable split layout (chat left, artifacts right)

**HLD:**
```
Chat conversation (left 2/3) | Artifact Panel (right 1/3, collapsible)
                              |   - Architecture diagram
                              |   - Sizing table (when discussed)
                              |   - Config (when generated)
                              |   - Download Report button
```

**DLD:**
- Uses a resizable panel component (shadcn/ui `ResizablePanel`)
- Artifacts stored in React state, keyed by type
- New artifacts replace old ones of the same type (architecture diagram updates as conversation progresses)
- Collapse/expand toggle
- "Download Report" button always visible at bottom

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 2.4.1 | Panel renders on /advise page | Visible on right side |
| 2.4.2 | Architecture diagram artifact | Appears in panel |
| 2.4.3 | Second architecture diagram | Replaces first |
| 2.4.4 | Collapse panel | Panel hides, chat takes full width |
| 2.4.5 | Expand panel | Panel reappears with artifacts preserved |
| 2.4.6 | Multiple artifact types | Each in its own section |
| 2.4.7 | Mobile viewport | Panel becomes a bottom sheet or tab |

**Learning doc topics:**
- Split panel layouts with shadcn/ui ResizablePanel
- React state for accumulating artifacts
- Responsive design patterns (desktop split vs mobile stacked)

---

## Phase 2 Completion Criteria

When all 4 modules are done:
- AI explains proxy mode and a Mermaid flow diagram renders inline
- AI generates customized architecture diagrams based on client inputs
- Artifact panel on the right accumulates diagrams as conversation progresses
- All 9 diagram types generate valid Mermaid and render correctly
- Diagrams support dark mode

---

## Phase 3: Sizing, Configs, and PDF Report

**Goal:** Full artifact generation. Sizing calculator, TOML configs, monitoring playbook, PDF download.

### Module 3.1: Sizing Calculator

**What:** Calculates Relay Proxy instances, persistent store, and service connections.

**Deliverables:**
- `functions/engine/sizing_calculator.py`
- SC estimation by architecture pattern
- Unit tests with all client scenarios

**DLD:**
- `calculate_relay_sizing(profile)`: instances, CPU, memory, disk, network
- `calculate_store_sizing(profile, store_type)`: Redis/DynamoDB/Consul specific
- `estimate_service_connections(profile)`: SC formula based on pattern
- `recommend_partitioning(profile)`: Separate clusters for prod vs non-prod
- All numbers labeled official vs estimate

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 3.1.1 | Minimal profile | 3 instances, m4.xlarge reference, 2x buffer |
| 3.1.2 | 200+ services | More instances, partitioning recommended |
| 3.1.3 | Forking pattern (4 workers, 10 hosts) | SC = 40, provisioned = 80 |
| 3.1.4 | Containerized K8s (20 pods, 3 nodes) | SC = 60, provisioned = 120 |
| 3.1.5 | Lambda (peak 500 concurrent) | SC = 500, warning about unpredictability |
| 3.1.6 | Zero-downtime profile | Redis persistence = AOF recommended |
| 3.1.7 | DynamoDB preference | DynamoDB-specific sizing returned |

**Learning doc topics:**
- How LD sizes Relay (official numbers vs estimates)
- Service connection math (multiplication patterns)
- Why 2x provisioning buffer
- Environment partitioning for blast radius reduction

---

### Module 3.2: Config Generator

**What:** Generates sample Relay Proxy TOML configuration files.

**Deliverables:**
- `functions/engine/config_generator.py`
- `functions/templates/configs/` (7 TOML templates)
- Unit tests for each template

**Templates:** proxy-mode.toml, proxy-mode-redis.toml, daemon-mode.toml, daemon-mode-dynamodb.toml, offline-mode.toml, offline-mode-file.toml, proxy-mode-cloud.toml

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 3.2.1 | Proxy mode + Redis | Valid TOML with USE_REDIS=true |
| 3.2.2 | Daemon mode | Valid TOML with appropriate settings |
| 3.2.3 | Offline mode | Valid TOML with no LD connection settings |
| 3.2.4 | Custom environment names | Env names appear in config |
| 3.2.5 | All TOML outputs | Parse without errors |

---

### Module 3.3: Monitoring and Checklist Generators

**What:** Generates monitoring playbook, reverse proxy checklist, network requirements table.

**Deliverables:**
- Monitoring playbook generator (reads monitoring-thresholds.json)
- Reverse proxy checklist generator (reads reverse-proxy-settings.json)
- Network requirements table generator (reads network-requirements.json)
- Rendered as structured JSON for frontend display

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 3.3.1 | Monitoring playbook | 6 thresholds with severity and action |
| 3.3.2 | Reverse proxy checklist for nginx | nginx-specific settings |
| 3.3.3 | Reverse proxy checklist for ALB | ALB-specific settings |
| 3.3.4 | Network requirements for proxy mode | All 5 LD domains listed |
| 3.3.5 | Network requirements for offline mode | No outbound domains |

---

### Module 3.4: Artifact UI Components

**What:** Frontend components that render sizing tables, config viewers, checklists in the artifact panel.

**Deliverables:**
- `frontend/components/artifacts/SizingTable.tsx`
- `frontend/components/artifacts/SCEstimate.tsx`
- `frontend/components/artifacts/ConfigViewer.tsx` (TOML with syntax highlighting)
- `frontend/components/artifacts/MonitoringPlaybook.tsx`
- `frontend/components/artifacts/ReverseProxyChecklist.tsx`
- `frontend/components/artifacts/NetworkRequirements.tsx`

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 3.4.1 | SizingTable renders | Shows instances, CPU, memory with official/estimate labels |
| 3.4.2 | ConfigViewer renders | TOML with syntax highlighting and copy button |
| 3.4.3 | MonitoringPlaybook renders | Severity color coding |
| 3.4.4 | All components handle empty data | Show "not yet generated" placeholder |

---

### Module 3.5: PDF Report Generation

**What:** Generates a downloadable PDF from all collected artifacts using Typst.

**Deliverables:**
- `functions/templates/report.typ` (Typst template)
- `functions/routers/report.py` (endpoint)
- PDF endpoint: POST /api/report with all artifacts, returns PDF bytes

**DLD:**
- Typst template with 16 sections (parameterized)
- Python renders variables into Typst template, calls `typst compile`
- Typst binary bundled in Cloud Function Docker image
- Returns PDF as binary download

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 3.5.1 | Generate report with full artifacts | Valid PDF with all sections |
| 3.5.2 | Generate report with partial artifacts | Sections marked "not available" |
| 3.5.3 | PDF has company name on title page | Company name rendered correctly |
| 3.5.4 | PDF has official vs estimate labels | Clearly marked throughout |
| 3.5.5 | ARC Airlines full scenario | PDF matches what SA would produce |

---

## Phase 3 Completion Criteria

When all 5 modules are done:
- AI generates sizing tables inline with SC estimates
- Sample TOML configs appear in artifact panel
- Monitoring playbook, reverse proxy checklist, network requirements generated
- PDF report downloadable with all 16 sections
- All 5 client scenarios produce accurate outputs end-to-end

---

## Phase 4: Learn Mode

**Goal:** Visual, interactive learning experience. Separate from the chat advisor.

### Module 4.1: Learn Mode Layout

**What:** /learn route with navigation and page structure.

**Deliverables:**
- `frontend/app/learn/page.tsx` (learning hub with cards linking to topics)
- `frontend/app/learn/layout.tsx` (learn mode layout with sidebar nav)
- Navigation between learn pages
- "Switch to Advise mode" CTA on every learn page

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 4.1.1 | Navigate to /learn | Hub page with topic cards |
| 4.1.2 | Click topic card | Navigates to topic page |
| 4.1.3 | "Switch to Advise" button | Navigates to /advise |

---

### Module 4.2: Mode Explorer

**What:** Interactive page showing proxy vs daemon vs offline with flow diagrams.

**Deliverables:**
- `frontend/app/learn/modes/page.tsx`
- `frontend/components/learn/ModeExplorer.tsx`
- Tab or toggle to switch between modes
- Each mode shows: flow diagram, when to use, when NOT to use, trade-offs

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 4.2.1 | Proxy mode tab | Shows proxy flow diagram and description |
| 4.2.2 | Daemon mode tab | Shows daemon flow diagram and SC billing warning |
| 4.2.3 | Offline mode tab | Shows offline flow and data sync patterns |
| 4.2.4 | Switch between tabs | Diagrams render correctly each time |

---

### Module 4.3: Decision Tree

**What:** Interactive clickable flowchart. Answer questions by clicking nodes. Ends at a recommendation.

**Deliverables:**
- `frontend/app/learn/decision-tree/page.tsx`
- `frontend/components/diagrams/DecisionTree.tsx`

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 4.3.1 | Start at root | First question rendered |
| 4.3.2 | Click YES/NO | Advances to next question |
| 4.3.3 | Reach leaf node | Shows recommendation with explanation |
| 4.3.4 | Reset button | Returns to root |

---

### Module 4.4: Failure Scenarios

**What:** Step-by-step visual walkthroughs of failure scenarios.

**Deliverables:**
- `frontend/app/learn/failures/page.tsx`
- `frontend/components/diagrams/FailureWalkthrough.tsx`
- 4 scenarios: LD down, Relay down, Redis down, everything down

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 4.4.1 | LD down scenario | 5-step walkthrough with diagrams |
| 4.4.2 | Step through with Next button | Each step shows updated diagram |
| 4.4.3 | All 4 scenarios selectable | Dropdown or tabs work |

---

### Module 4.5: SC Calculator

**What:** Interactive service connection estimator. Select pattern, input numbers, get estimate.

**Deliverables:**
- `frontend/app/learn/sc-calculator/page.tsx`
- `frontend/components/learn/SCCalculator.tsx`

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 4.5.1 | Select "containerized + forking" | Shows 3 input fields |
| 4.5.2 | Enter 4 hosts, 12 containers, 3 forks | SC = 144, provisioned = 288 |
| 4.5.3 | Change pattern | Inputs reset, formula updates |

---

### Module 4.6: Anti-Pattern Gallery

**What:** Visual cards showing common mistakes with correct approaches.

**Deliverables:**
- `frontend/app/learn/anti-patterns/page.tsx`
- `frontend/components/diagrams/AntiPatternCards.tsx`

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 4.6.1 | Page renders | All 6 anti-patterns displayed as cards |
| 4.6.2 | Click card | Expands to show symptoms, correct approach, source |

---

## Phase 4 Completion Criteria

- SE can walk a client through Learn mode during a call
- Every concept has a diagram
- SC calculator produces correct estimates for all patterns
- Decision tree leads to correct recommendations
- Learn mode and Advise mode feel like one product

---

## Phase 5: Polish and Deploy

**Goal:** Production-ready. Beautiful. Fast. Zero hallucinations.

### Module 5.1: UI Polish

**What:** Dark mode, animations, loading states, responsive design.

**Deliverables:**
- Dark/light mode toggle (persisted in localStorage)
- Loading skeletons for chat and artifacts
- Smooth scroll and animations
- Mobile responsive (chat stacked, artifacts as bottom sheet)

---

### Module 5.2: Error Handling

**What:** Graceful handling of all failure modes.

**Deliverables:**
- Gemini API failure: show error message, allow retry
- Network timeout: show reconnection message
- Invalid Mermaid: show code block fallback
- PDF generation failure: show error with manual download option
- Rate limiting: queue messages, show wait indicator

---

### Module 5.3: Onboarding and Quick Actions

**What:** First-time user experience and quick action buttons.

**Deliverables:**
- Welcome message on first visit explaining Learn vs Advise
- Quick action bar at bottom of chat
- "About" page explaining the tool, sources, limitations
- Keyboard shortcut (Enter to send, Shift+Enter for newline)

---

### Module 5.4: End-to-End Testing

**What:** Full scenario testing with all 5 client profiles.

**Deliverables:**
- E2E test script for each client scenario
- Hallucination audit: every technical fact checked against knowledge files
- Performance benchmarks: response time, PDF generation time
- Cross-browser testing (Chrome, Firefox, Safari)

**Test cases:**
| ID | Test | Expected |
|----|------|----------|
| 5.4.1 | ARC Airlines full flow | Correct recommendation, sizing, config, PDF |
| 5.4.2 | ACI full flow | Handles multi-deployment + air-gapped correctly |
| 5.4.3 | Island full flow | K8s-specific guidance in architecture |
| 5.4.4 | NinjaTrader full flow | Recommends against Relay (or minimal, ad-blocker only) |
| 5.4.5 | Tricon full flow | Recommends against Relay (greenfield, direct) |
| 5.4.6 | Hallucination audit | Zero facts stated without tool-call source |
| 5.4.7 | PDF for each scenario | All 5 PDFs generated, reviewed |

---

### Module 5.5: Production Deploy

**What:** Deploy to Firebase with custom domain.

**Deliverables:**
- Production Firebase project configured
- Custom domain setup
- Environment variables secured (GCP project ID, region)
- Cloud Function memory/timeout tuned
- Firebase Hosting CDN cache rules
- README with setup instructions for other devs

---

## Phase 5 Completion Criteria

- App runs on production Firebase URL
- Dark mode and mobile work
- All 5 client scenarios pass end-to-end
- Zero hallucinations verified
- PDF generation works reliably
- A client can go from zero to PDF in 30 minutes
- An SE can use it live on a call

---

## Module Dependency Graph

```
Phase 1 (sequential within, parallel where noted):
  1.1 Firebase Setup
    |
  1.2 Data Models
    |
  1.3 Knowledge Files ----+
    |                      |
  1.4 Grounding Tools <----+
    |
  1.5 Rule Engine (parallel with 1.4)
    |
  1.6 Gemini Client (depends on 1.4 + 1.5)
    |
  1.7 Chat API (depends on 1.6)
    |
  1.8 Chat UI (depends on 1.7)

Phase 2 (after Phase 1):
  2.1 Mermaid Renderer (frontend, can start with 1.8)
    |
  2.2 Diagram Generator (backend, parallel with 2.1)
    |
  2.3 Inline Rendering (depends on 2.1 + 2.2)
    |
  2.4 Artifact Panel (parallel with 2.3)

Phase 3 (after Phase 2):
  3.1 Sizing Calculator (backend)
  3.2 Config Generator (backend, parallel with 3.1)
  3.3 Monitoring + Checklists (backend, parallel with 3.1)
    |
  3.4 Artifact UI Components (frontend, depends on 3.1-3.3)
    |
  3.5 PDF Report (depends on 3.1-3.4)

Phase 4 (after Phase 2, parallel with Phase 3):
  4.1 Learn Layout
    |
  4.2-4.6 Learn pages (parallel, all depend on 4.1 + 2.1)

Phase 5 (after Phase 3 + 4):
  5.1-5.3 Polish (parallel)
    |
  5.4 E2E Testing (depends on 5.1-5.3)
    |
  5.5 Deploy (depends on 5.4)
```

---

## Total Module Count: 27

| Phase | Modules | Focus |
|-------|---------|-------|
| Phase 1 | 8 modules | Platform, backend, AI, basic chat |
| Phase 2 | 4 modules | Diagrams, artifact panel |
| Phase 3 | 5 modules | Sizing, configs, PDF |
| Phase 4 | 6 modules | Learn mode pages |
| Phase 5 | 5 modules | Polish, testing, deploy |
