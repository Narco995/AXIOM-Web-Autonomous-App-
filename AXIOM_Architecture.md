# AXIOM — Full Architecture & Technical Specs
### Autonomous Web App Builder Agent

---

## 🏛️ AXIOM SYSTEM OVERVIEW

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AXIOM AGENT SYSTEM                         │
│                                                                     │
│   USER INPUT                                                        │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────┐                                                    │
│  │  INPUT      │  Natural language → Structured Goal                │
│  │  PARSER     │                                                    │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────┐     ┌──────────────┐     ┌─────────────────────┐  │
│  │  PLANNING   │────▶│  TASK QUEUE  │────▶│   EXECUTION ENGINE  │  │
│  │  ENGINE     │     │  (Priority)  │     │   (ReAct Loop)      │  │
│  └─────────────┘     └──────────────┘     └────────┬────────────┘  │
│                                                    │               │
│         ┌──────────────────────────────────────────┤               │
│         │                                          │               │
│         ▼                                          ▼               │
│  ┌─────────────┐                         ┌─────────────────────┐   │
│  │  MEMORY     │◀────────────────────────│   TOOL ROUTER       │   │
│  │  SYSTEM     │                         │   (Decision Logic)  │   │
│  └─────────────┘                         └────────┬────────────┘   │
│                                                   │                │
│              ┌────────────────────────────────────┤                │
│              │            │            │           │                │
│              ▼            ▼            ▼           ▼                │
│         ┌────────┐  ┌──────────┐ ┌────────┐ ┌──────────┐          │
│         │BROWSER │  │TERMINAL  │ │  CODE  │ │   API    │          │
│         │TOOL    │  │/SHELL    │ │EDITOR  │ │  TOOLS   │          │
│         └────────┘  └──────────┘ └────────┘ └──────────┘          │
│                                                                     │
│         ┌───────────────────────────────────────────────┐          │
│         │              FEEDBACK LOOP                    │          │
│         │  Output → Observe → Evaluate → Next Action    │          │
│         └───────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📐 LAYER-BY-LAYER ARCHITECTURE

---

## LAYER 1 — INPUT & GOAL PARSING LAYER

### Purpose
Convert raw user natural language into a structured, machine-actionable goal object.

### Architecture

```
Raw Input (text)
     │
     ▼
┌────────────────────────────────────────────────────────┐
│                  INPUT PARSER MODULE                   │
│                                                        │
│  Step 1: Intent Classification                         │
│  ┌──────────────────────────────────────────────┐      │
│  │  LLM Prompt: "What type of task is this?"    │      │
│  │  Output: { type: "build_web_app",             │      │
│  │            complexity: "high",                │      │
│  │            domain: "ecommerce" }              │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  Step 2: Requirement Extraction                        │
│  ┌──────────────────────────────────────────────┐      │
│  │  Extract: features[], stack[], constraints[] │      │
│  │  Example: features: ["auth", "payments",     │      │
│  │           "dashboard"], stack: ["React",     │      │
│  │           "Node"], constraints: ["no SQL"]   │      │
│  └──────────────────────────────────────────────┘      │
│                                                        │
│  Step 3: Ambiguity Detection                           │
│  ┌──────────────────────────────────────────────┐      │
│  │  If confidence < 0.85 → Ask clarifying Q     │      │
│  │  If confidence ≥ 0.85 → Proceed to planner   │      │
│  └──────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────┘
     │
     ▼
GoalObject {
  id: uuid,
  raw_input: string,
  intent: string,
  features: string[],
  stack: string[],
  constraints: string[],
  complexity_score: float,
  created_at: timestamp
}
```

### Technical Specs

| Component | Technology | Details |
|-----------|-----------|---------|
| LLM | GPT-4o / Claude 3.5 Sonnet | 128k context, JSON mode |
| Intent Classifier | LLM + Few-shot prompting | 10-15 examples per category |
| Requirement Extractor | Structured output (Pydantic) | Typed schema validation |
| Ambiguity Scorer | Cosine similarity + LLM | Threshold: 0.85 confidence |
| Input Schema | Pydantic v2 BaseModel | Strict validation |

### Process Flow
1. User sends: *"Build me a todo app with auth and Firebase"*
2. LLM classifies → `type: build_web_app`
3. Extracts → `features: [auth, todo_crud], backend: firebase`
4. Scores complexity → `0.72 (medium)`
5. Confidence = `0.91` → proceeds to Planning Layer
6. Returns structured `GoalObject` JSON

---

## LAYER 2 — PLANNING ENGINE LAYER

### Purpose
Decompose the GoalObject into an ordered, prioritized task tree that AXIOM can execute step by step.

### Architecture

```
GoalObject
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                    PLANNING ENGINE                      │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  HIERARCHICAL TASK NETWORK (HTN) DECOMPOSER      │   │
│  │                                                  │   │
│  │  Goal: Build Todo App                            │   │
│  │  ├── Phase 1: Setup (priority: 1)                │   │
│  │  │   ├── Task 1.1: Init project structure        │   │
│  │  │   ├── Task 1.2: Install dependencies          │   │
│  │  │   └── Task 1.3: Configure environment         │   │
│  │  ├── Phase 2: Backend (priority: 2)              │   │
│  │  │   ├── Task 2.1: Setup Firebase config         │   │
│  │  │   ├── Task 2.2: Build auth module             │   │
│  │  │   └── Task 2.3: Build CRUD API                │   │
│  │  ├── Phase 3: Frontend (priority: 3)             │   │
│  │  │   ├── Task 3.1: Build UI components           │   │
│  │  │   ├── Task 3.2: Connect frontend to backend   │   │
│  │  │   └── Task 3.3: Add routing                  │   │
│  │  └── Phase 4: Validation (priority: 4)          │   │
│  │      ├── Task 4.1: Run tests                     │   │
│  │      ├── Task 4.2: Visual verification           │   │
│  │      └── Task 4.3: Fix errors                    │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  DEPENDENCY GRAPH BUILDER                        │   │
│  │  Ensures tasks run in correct order              │   │
│  │  Detects parallelizable tasks                    │   │
│  │  Flags blocking dependencies                     │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  PLAN VALIDATOR                                  │   │
│  │  Checks plan completeness                        │   │
│  │  Estimates token cost per phase                  │   │
│  │  Flags high-risk steps for extra verification    │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
     │
     ▼
ExecutionPlan {
  phases: Phase[],
  tasks: Task[],
  dependency_graph: DAG,
  estimated_steps: int,
  risk_flags: string[]
}
```

### Technical Specs

| Component | Technology | Details |
|-----------|-----------|---------|
| Decomposition Method | HTN + LLM Chain-of-Thought | Recursive subtask generation |
| Task Graph | NetworkX (Python) | DAG for dependency tracking |
| Plan Storage | SQLite / PostgreSQL | Persistent task state |
| Parallelism Detector | Topological sort | Identify independent tasks |
| Risk Scorer | LLM + heuristics | Flags auth/payment/DB tasks |
| Plan Format | JSON Schema + Pydantic | Typed, validated output |

### Process Flow
1. Receives `GoalObject`
2. LLM generates high-level phases (HTN decomposition)
3. Each phase recursively broken into atomic tasks
4. Dependency graph built (what blocks what)
5. Tasks sorted by priority + dependency order
6. Plan stored in DB → sent to Task Queue

---

## LAYER 3 — TASK QUEUE & EXECUTION ENGINE (ReAct Loop)

### Purpose
The beating heart of AXIOM. Pulls tasks from the queue and runs the **Think → Act → Observe** loop until each task is complete.

### Architecture

```
ExecutionPlan
     │
     ▼
┌───────────────────────────────────────────────────────────┐
│                    TASK QUEUE                             │
│  Priority Queue (min-heap)                                │
│  task_id | priority | status | dependencies_met           │
│  ─────────────────────────────────────────────            │
│  task_1.1 | 1       | ready  | true                      │
│  task_1.2 | 2       | wait   | false                     │
│  task_2.1 | 3       | wait   | false                     │
└────────────────┬──────────────────────────────────────────┘
                 │  Pop next ready task
                 ▼
┌───────────────────────────────────────────────────────────┐
│                    REACT EXECUTION LOOP                   │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  THINK (Reasoning Step)                             │  │
│  │  ─────────────────────                              │  │
│  │  Input: current task + memory + previous output     │  │
│  │  Process: LLM generates reasoning + action plan     │  │
│  │  Output: { thought: "...", action: "run_shell",     │  │
│  │            action_input: "npm create react-app" }   │  │
│  └──────────────────┬──────────────────────────────────┘  │
│                     │                                     │
│                     ▼                                     │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  ACT (Tool Execution Step)                          │  │
│  │  ──────────────────────                             │  │
│  │  Tool Router selects correct tool                   │  │
│  │  Executes with provided parameters                  │  │
│  │  Enforces timeout (default: 30s per tool call)      │  │
│  │  Returns raw output (stdout, screenshot, JSON)      │  │
│  └──────────────────┬──────────────────────────────────┘  │
│                     │                                     │
│                     ▼                                     │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  OBSERVE (Output Evaluation Step)                   │  │
│  │  ──────────────────────────────                     │  │
│  │  Parse tool output                                  │  │
│  │  Evaluate: success / partial / failure              │  │
│  │  Update memory with new information                 │  │
│  │  Decide: done / retry / fix / next_task             │  │
│  └──────────────────┬──────────────────────────────────┘  │
│                     │                                     │
│          ┌──────────┴──────────┐                          │
│          ▼                     ▼                          │
│     Task Done            Task Failed                      │
│     → mark complete      → Error Handler                  │
│     → next task          → max retries?                   │
│                          → escalate                       │
└───────────────────────────────────────────────────────────┘
```

### Technical Specs

| Component | Technology | Details |
|-----------|-----------|---------|
| ReAct Framework | LangGraph / LangChain | State machine-based loop |
| LLM Reasoning | GPT-4o (128k ctx) | Temperature: 0.2 (deterministic) |
| Task Queue | Redis (sorted set) | Priority queue, persistent |
| Loop Control | Max iterations: 20 per task | Prevent infinite loops |
| Timeout Handler | Python asyncio | 30s default, 120s for installs |
| State Machine | LangGraph StateGraph | Nodes: think, act, observe, done |
| Output Parser | LangChain OutputParser | Regex + structured JSON |

### Process Flow (per task)
1. Pop highest-priority ready task from queue
2. **THINK**: LLM receives task + context → outputs thought + action
3. **ACT**: Tool Router executes chosen tool → returns output
4. **OBSERVE**: LLM evaluates output → success/fail/retry decision
5. Memory updated with new facts
6. Loop repeats or advances to next task
7. Max 20 iterations per task → escalate if not resolved

---

## LAYER 4 — TOOL ROUTER & TOOL LAYER

### Purpose
Decide which tool to use at each step and execute it. AXIOM's tool belt gives it real-world capabilities beyond just generating text.

### Tool Decision Logic

```
Thought from ReAct Loop
         │
         ▼
┌──────────────────────────────────────────────────────────┐
│                    TOOL ROUTER                           │
│                                                          │
│  Decision Matrix:                                        │
│  ─────────────────                                       │
│  If action == "browse_web"      → Browser Tool           │
│  If action == "run_command"     → Shell Tool             │
│  If action == "write_code"      → Code Editor Tool       │
│  If action == "read_file"       → File System Tool       │
│  If action == "search_web"      → Search Tool            │
│  If action == "call_api"        → API Tool               │
│  If action == "take_screenshot" → Vision Tool            │
│  If action == "deploy"          → Deploy Tool            │
│  Unknown → LLM re-reasons                                │
└──────────────────────────────────────────────────────────┘
```

### Full Tool Inventory

#### 🌐 Browser Tool
```
Tool: Playwright Browser Controller
Purpose: Navigate web, interact with UI, verify visual output

Capabilities:
  - navigate(url)           → Go to URL
  - click(selector)         → Click element
  - type(selector, text)    → Type into input
  - screenshot()            → Capture page image
  - get_dom()               → Extract full DOM
  - wait_for_element(sel)   → Wait for element
  - execute_js(code)        → Run JavaScript
  - extract_text(selector)  → Get text content

Tech: Playwright (Python async), Chromium headless
Output: HTML, screenshots (PNG), text, JSON
Use case: Test built web app visually, verify UI renders correctly
```

#### 💻 Shell / Terminal Tool
```
Tool: Sandboxed Shell Executor
Purpose: Run terminal commands for file ops, installs, builds

Capabilities:
  - run(command, timeout)   → Execute shell command
  - read_stdout()           → Get command output
  - read_stderr()           → Get error output
  - get_exit_code()         → Check success/fail

Tech: Python subprocess + Docker container sandbox
Timeout: 30s default, 300s for npm install / builds
Security: Sandboxed Docker container, no host access
Output: stdout string, stderr string, exit_code int
```

#### 📝 Code Editor Tool
```
Tool: File-Based Code Editor
Purpose: Write, read, edit, and manage project files

Capabilities:
  - write_file(path, content)       → Create/overwrite file
  - edit_file(path, old, new)       → Find and replace
  - read_file(path)                 → Read file contents
  - delete_file(path)               → Remove file
  - list_directory(path)            → List files
  - create_directory(path)          → Make folder
  - search_in_files(query, path)    → Grep-style search

Tech: Python pathlib + watchdog for file monitoring
Format: UTF-8 text files, binary support for images
```

#### 🔍 Web Search Tool
```
Tool: Web Search + Scraper
Purpose: Find documentation, libraries, solutions

Capabilities:
  - search(query, limit)      → Web search results
  - scrape(url)               → Extract page content
  - find_npm_package(name)    → NPM package info
  - find_stackoverflow(error) → Error solutions

Tech: SerpAPI / DuckDuckGo + BeautifulSoup
Output: Markdown text, URLs, code snippets
```

#### 🚀 Deploy Tool
```
Tool: Web App Deployer
Purpose: Deploy completed web app to hosting provider

Capabilities:
  - deploy_to_vercel(project_path)    → Vercel deploy
  - deploy_to_netlify(project_path)   → Netlify deploy
  - build_docker(project_path)        → Create Docker image
  - start_local_server(port)          → Local preview

Tech: Vercel CLI, Netlify CLI, Docker SDK
Output: Live URL, deployment logs, build status
```

#### 👁️ Vision Tool
```
Tool: Screenshot → LLM Vision Analyzer
Purpose: See what the web app looks like visually

Capabilities:
  - analyze_screenshot(img)          → Describe what's visible
  - check_ui_errors(img)             → Detect visual bugs
  - verify_feature_visible(img, feat)→ Confirm feature rendered

Tech: GPT-4o Vision / Claude Vision API
Input: PNG screenshot from Playwright
Output: JSON { visible: bool, issues: [], description: string }
```

### Technical Specs

| Tool | Library | Timeout | Output Type |
|------|---------|---------|------------|
| Browser | Playwright async | 30s | HTML/PNG/text |
| Shell | subprocess + Docker | 30-300s | stdout/stderr |
| Code Editor | pathlib | 5s | file content |
| Search | SerpAPI | 10s | JSON results |
| Deploy | Vercel/Netlify CLI | 120s | URL + logs |
| Vision | GPT-4o Vision | 15s | JSON analysis |

---

## LAYER 5 — ERROR HANDLING & RECOVERY LAYER

### Purpose
When something breaks (and it will), AXIOM diagnoses the error, decides the best fix strategy, and tries to self-heal without asking the user.

### Architecture

```
Tool Output (Failed)
       │
       ▼
┌────────────────────────────────────────────────────────────┐
│                  ERROR HANDLER                             │
│                                                            │
│  Step 1: ERROR CLASSIFICATION                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Parse stderr / exception / status code              │  │
│  │  Classify into:                                      │  │
│  │    TYPE_A: Syntax Error      → Fix code directly     │  │
│  │    TYPE_B: Missing Package   → npm install / pip     │  │
│  │    TYPE_C: Config Error      → Fix config file       │  │
│  │    TYPE_D: Network Error     → Retry with backoff    │  │
│  │    TYPE_E: Logic Error       → Re-plan this task     │  │
│  │    TYPE_F: Unknown Error     → Search for solution   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Step 2: REPAIR STRATEGY SELECTION                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  retry_count < 3  → Apply direct fix                 │  │
│  │  retry_count == 3 → Search web for solution          │  │
│  │  retry_count == 4 → Re-plan this phase               │  │
│  │  retry_count >= 5 → Escalate to user                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Step 3: FIX EXECUTION                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Apply fix → Re-run original action → Observe        │  │
│  │  Log all attempts to memory                          │  │
│  │  Track error patterns to avoid repeats               │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Error Type Reference

| Error Type | Detection | Fix Strategy |
|-----------|----------|-------------|
| Syntax Error | stderr contains "SyntaxError" | LLM fixes code, re-runs |
| Module Not Found | stderr: "Cannot find module" | Run npm/pip install |
| Port In Use | stderr: "EADDRINUSE" | Kill process, retry |
| Build Failed | exit_code != 0 + build log | Analyze log, fix imports |
| Network Timeout | requests.Timeout | Retry with exponential backoff |
| CORS Error | Browser console error | Add CORS headers to backend |
| Type Error | Runtime exception | LLM patches type mismatch |
| Visual Bug | Vision tool detects wrong UI | Re-examine CSS/component code |

### Technical Specs

| Component | Technology | Details |
|-----------|-----------|---------|
| Error Parser | Regex + LLM | Pattern matching + semantic understanding |
| Retry Logic | tenacity (Python) | Exponential backoff, max 5 retries |
| Web Search for Errors | SerpAPI | Query: error message + stack |
| Fix History | SQLite | Track all fix attempts per task |
| Escalation | Notification system | Email/UI alert to user |

---

## LAYER 6 — MEMORY SYSTEM LAYER

### Purpose
Give AXIOM persistent, searchable memory so it never forgets what it has built, what failed, or what the user wants.

### 4-Layer Memory Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AXIOM MEMORY SYSTEM                          │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  LAYER A: WORKING MEMORY (In-Context)                      │ │
│  │  ─────────────────────────────────────                     │ │
│  │  What: Current task state, last 10 actions, active errors  │ │
│  │  Storage: LLM context window (up to 128k tokens)           │ │
│  │  TTL: Duration of current task only                        │ │
│  │  Tech: Python dict / LangGraph state object                │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  LAYER B: EPISODIC MEMORY (Session)                        │ │
│  │  ─────────────────────────────────                         │ │
│  │  What: Full log of all actions taken this session          │ │
│  │  Storage: SQLite DB (structured rows)                      │ │
│  │  Schema: {task_id, action, result, timestamp, status}      │ │
│  │  Retrieval: SQL query by task_id or timestamp              │ │
│  │  TTL: Full project session                                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  LAYER C: SEMANTIC MEMORY (Long-term Knowledge Base)       │ │
│  │  ────────────────────────────────────────────────          │ │
│  │  What: Project facts, docs, code summaries, decisions      │ │
│  │  Storage: ChromaDB vector database                         │ │
│  │  Embedding: text-embedding-3-small (OpenAI)                │ │
│  │  Retrieval: cosine similarity search (top-k = 5)           │ │
│  │  TTL: Full project lifetime                                │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  LAYER D: CODE MEMORY (Repository Index)                   │ │
│  │  ────────────────────────────────────────                  │ │
│  │  What: Index of all files, functions, components built     │ │
│  │  Storage: ChromaDB (code-specific collection)              │ │
│  │  Embedding: code-embedding model (CodeBERT / OpenAI)       │ │
│  │  Retrieval: "find the auth component" → returns file path  │ │
│  │  TTL: Full project lifetime                                │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Memory Operations

```python
# Writing to memory
memory.working.set("current_task", task_object)
memory.episodic.log(action="write_file", result="success", task_id="1.1")
memory.semantic.store("Firebase auth is configured at /src/firebase.js")
memory.code.index(file_path="/src/auth/Login.jsx", summary="Login component")

# Reading from memory
ctx = memory.working.get_all()
history = memory.episodic.get(task_id="1.1")
docs = memory.semantic.search("how is auth configured?", k=5)
file = memory.code.find("login component")
```

### Technical Specs

| Layer | Technology | Query Speed | Max Size |
|-------|-----------|-------------|---------|
| Working | Python dict / LangGraph | ~0ms | 128k tokens |
| Episodic | SQLite / PostgreSQL | <5ms | Unlimited rows |
| Semantic | ChromaDB | <50ms | Millions of docs |
| Code | ChromaDB (code collection) | <50ms | Full codebase |

---

## LAYER 7 — BROWSER & VISUAL VERIFICATION LAYER

### Purpose
AXIOM can actually *see* the web app it builds, verify the UI looks correct, and fix visual bugs — just like Devin AI.

### Architecture

```
Build Step Complete
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│               VISUAL VERIFICATION PIPELINE                  │
│                                                             │
│  Step 1: Start Local Server                                 │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  shell.run("npm run dev -- --port 3000")             │   │
│  │  Wait for server ready (poll /health endpoint)       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  Step 2: Take Screenshot                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  browser.navigate("http://localhost:3000")           │   │
│  │  browser.wait_for_load()                             │   │
│  │  screenshot = browser.screenshot()                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  Step 3: Vision Analysis                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  result = vision.analyze(screenshot, checklist=[     │   │
│  │    "login form visible?",                            │   │
│  │    "no error messages?",                             │   │
│  │    "buttons clickable?",                             │   │
│  │    "layout correct?"                                 │   │
│  │  ])                                                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  Step 4: Fix or Approve                                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  If result.issues → trigger error handler            │   │
│  │  If result.all_pass → mark visual check ✅           │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Technical Specs

| Component | Technology | Details |
|-----------|-----------|---------|
| Browser Engine | Playwright + Chromium | Headless, async |
| Screenshot | Playwright .screenshot() | PNG, full page |
| Vision Analysis | GPT-4o Vision API | Checklist verification |
| Server Health Check | httpx async poll | 5s interval, 60s max wait |
| Visual Test Cases | JSON checklist per feature | Defined in planning phase |

---

## LAYER 8 — CODE EXECUTION ENVIRONMENT

### Purpose
A safe, isolated environment where AXIOM writes and runs code without risk to the host system.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              AXIOM EXECUTION SANDBOX                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Docker Container (per project)                     │    │
│  │  ─────────────────────────────                      │    │
│  │  Base Image: node:20-alpine + python:3.12           │    │
│  │  Mounted Volume: /workspace/{project_id}/           │    │
│  │  Network: isolated bridge network                   │    │
│  │  Ports: 3000 (frontend), 8000 (backend) exposed     │    │
│  │  Resource Limits:                                   │    │
│  │    CPU: 2 cores max                                 │    │
│  │    RAM: 4GB max                                     │    │
│  │    Disk: 10GB max                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  File System Layout                                 │    │
│  │  ─────────────────                                  │    │
│  │  /workspace/                                        │    │
│  │  ├── {project_id}/          ← web app code here    │    │
│  │  │   ├── src/                                      │    │
│  │  │   ├── public/                                   │    │
│  │  │   ├── package.json                              │    │
│  │  │   └── ...                                       │    │
│  │  ├── axiom_logs/            ← execution logs       │    │
│  │  └── axiom_memory/          ← memory snapshots     │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Technical Specs

| Component | Technology | Details |
|-----------|-----------|---------|
| Container Runtime | Docker / Podman | One container per project |
| Base Image | node:20-alpine | Lightweight, fast startup |
| Volume Mount | Docker bind mount | Persistent project files |
| Port Exposure | Docker port mapping | 3000, 8000 available to host |
| Resource Limits | Docker --memory --cpus | Prevent runaway processes |
| Shell Access | docker exec | AXIOM runs commands via exec |

---

## LAYER 9 — SELF-DIRECTION & AUTONOMY ENGINE

### Purpose
AXIOM knows what to do NEXT without being told. This is what makes it autonomous vs. a chatbot.

### How Self-Direction Works

```
┌─────────────────────────────────────────────────────────────┐
│               SELF-DIRECTION ENGINE                         │
│                                                             │
│  At every decision point, AXIOM asks itself:               │
│                                                             │
│  1. WHAT IS THE CURRENT STATE?                              │
│     → Read working memory + episodic log                   │
│     → What has been done? What succeeded? What failed?     │
│                                                             │
│  2. WHAT IS THE NEXT REQUIRED STEP?                        │
│     → Check task queue for next unblocked task             │
│     → Is it in current phase or can a new phase start?     │
│                                                             │
│  3. DO I HAVE ENOUGH INFORMATION TO PROCEED?               │
│     → If yes → proceed to ReAct loop                       │
│     → If no → search memory / web before acting            │
│                                                             │
│  4. WHAT COULD GO WRONG?                                    │
│     → Risk check: Is this a sensitive action?              │
│     → If high risk → add extra verification step           │
│                                                             │
│  5. HOW WILL I KNOW I SUCCEEDED?                           │
│     → Define success criteria before acting                │
│     → Set up observation to verify outcome                 │
└─────────────────────────────────────────────────────────────┘
```

### Technical Specs

| Component | Technology | Details |
|-----------|-----------|---------|
| Self-Prompting | System prompt + ReAct | "What should I do next?" |
| Goal Tracking | Task queue + DB | Always knows what's left |
| Success Criteria | Per-task JSON schema | Defined at planning time |
| Risk Classifier | LLM + heuristics | Flags risky actions |
| Autonomy Loop | LangGraph StateGraph | No human input needed per step |

---

## LAYER 10 — OUTPUT & DELIVERY LAYER

### Purpose
When AXIOM is done building, it packages, documents, and delivers the finished web app to the user.

### Architecture

```
All Tasks Complete
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│                  OUTPUT & DELIVERY                          │
│                                                             │
│  Step 1: Final Verification                                 │
│    → Run full test suite                                    │
│    → Visual check all pages                                 │
│    → Check all features work                               │
│                                                             │
│  Step 2: Documentation Generation                           │
│    → Auto-generate README.md                               │
│    → Document all API endpoints                            │
│    → List environment variables                            │
│    → Write setup instructions                              │
│                                                             │
│  Step 3: Packaging                                          │
│    → ZIP project files                                     │
│    → Export Docker image (optional)                        │
│    → Push to GitHub (optional)                             │
│                                                             │
│  Step 4: Deployment (optional)                              │
│    → Deploy to Vercel / Netlify                            │
│    → Return live URL to user                               │
│                                                             │
│  Step 5: Summary Report                                     │
│    → List all files created                                │
│    → List all features implemented                         │
│    → List any known limitations                            │
│    → Total tokens used / cost                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 FULL TECH STACK SUMMARY

| Layer | Component | Technology | Version |
|-------|-----------|-----------|---------|
| LLM Brain | Core reasoning | GPT-4o / Claude 3.5 | Latest |
| Agent Framework | ReAct loop | LangGraph | 0.2+ |
| LLM SDK | API calls | LangChain | 0.3+ |
| Task Queue | Priority queue | Redis | 7.x |
| Main Database | Episodic memory | PostgreSQL | 16 |
| Vector Database | Semantic memory | ChromaDB | 0.5+ |
| Embeddings | Text → vectors | text-embedding-3-small | Latest |
| Browser Automation | Visual + interaction | Playwright | 1.44+ |
| Container Runtime | Code sandbox | Docker | 24+ |
| Backend Framework | AXIOM API server | FastAPI | 0.110+ |
| Frontend (optional) | AXIOM Dashboard | React + TailwindCSS | 18 |
| File System | Project files | Python pathlib | 3.12 |
| Web Search | Error solving | SerpAPI / DuckDuckGo | - |
| Deploy Tool | App deployment | Vercel CLI / Netlify CLI | Latest |
| Vision | Screenshot analysis | GPT-4o Vision | Latest |
| Monitoring | Logs + metrics | Loguru + Prometheus | - |

---

## 🚀 AXIOM BUILD ROADMAP

```
WEEK 1-2: CORE LOOP (MVP)
━━━━━━━━━━━━━━━━━━━━━━━━
□ Set up Python project + FastAPI server
□ Implement basic ReAct loop with LangGraph
□ Add Shell tool (run commands)
□ Add Code Editor tool (write/read files)
□ Test: "Create a hello world HTML file"

WEEK 3-4: PLANNING + MEMORY
━━━━━━━━━━━━━━━━━━━━━━━━━━━
□ Build Planning Engine (HTN decomposer)
□ Set up PostgreSQL for episodic memory
□ Set up ChromaDB for semantic memory
□ Implement Task Queue with Redis
□ Test: "Build a React app with a button"

WEEK 5-6: BROWSER + VISION
━━━━━━━━━━━━━━━━━━━━━━━━━━
□ Integrate Playwright browser tool
□ Add screenshot → GPT-4o Vision pipeline
□ Implement visual verification checks
□ Add Docker sandbox for code execution
□ Test: "Build a login page and verify it looks correct"

WEEK 7-8: ERROR HANDLING + AUTONOMY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
□ Build full error classification system
□ Add web search for error solutions
□ Implement retry + re-plan logic
□ Add self-direction engine
□ Test: "Build a full todo app with broken packages"

WEEK 9-10: DELIVERY + POLISH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
□ Build output & delivery layer
□ Add Vercel/Netlify deployment
□ Auto-generate README docs
□ Build AXIOM dashboard UI
□ End-to-end test: "Build a full-stack app, deploy it"
```

---

*AXIOM — Build it once. Let it build everything else.*
