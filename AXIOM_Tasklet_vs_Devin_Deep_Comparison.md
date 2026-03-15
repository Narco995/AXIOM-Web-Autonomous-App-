# 🤖 AXIOM Project: Tasklet vs Devin AI — Deep Technical Comparison
### A builder's guide to understanding autonomous AI agents

---

## 1. CORE LOOP — Think → Act → Observe

### Tasklet's Loop

```
┌──────────────────────────────────────────────────────┐
│                                                      │
│  USER INPUT                                          │
│      ↓                                               │
│  [THINK] LLM reads full context + tool list          │
│      ↓                                               │
│  [DECIDE] Pick next tool OR respond to user          │
│      ↓                                               │
│  [ACT] Call tool with arguments                      │
│      ↓                                               │
│  [OBSERVE] Append tool result to context             │
│      ↓                                               │
│  [LOOP] LLM reads updated context → thinks again     │
│      ↓                                               │
│  [STOP] When goal is complete or needs user input    │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Speed:** Not measured in cycles/second. Each iteration takes 2–10 seconds depending on:
- LLM response time
- Tool execution time (sandbox, web, APIs)

**What triggers the next cycle?**
- A tool call completes → result appended → LLM re-evaluates
- The LLM explicitly decides to stop (outputs a final message instead of a tool call)

### Devin AI's Loop

Devin runs a **similar ReAct loop** BUT with one key difference:

```
Tasklet:  LLM → Tool → LLM → Tool → ...  (stateless LLM calls)
Devin:    LLM → Action → Observe → LLM → Action → ...
          + persistent browser/terminal state between steps
```

Devin maintains a **live environment** — a real browser tab stays open, terminal sessions persist, files are in a real filesystem. Every action happens inside that persistent environment. Tasklet's sandbox is ephemeral per session.

**Key difference:** Devin's loop is environment-persistent. Tasklet's loop is context-persistent.

---

## 2. GOAL DECOMPOSITION

### Tasklet's Method

When I receive "build a web app" I use **implicit hierarchical decomposition**:

```
Level 0 (Goal):     Build a web app
Level 1 (Phases):   Plan → Scaffold → Build → Test → Deliver
Level 2 (Tasks):    Create file structure, write HTML, write CSS...
Level 3 (Actions):  write_file("index.html", ...), run_command(...)
```

The LLM does this decomposition **in its reasoning** — it's not a separate algorithm. It uses its training to know what "building a web app" typically involves.

I also maintain an explicit **task list** (visible to you) to track sub-goals and prevent forgetting steps during long sessions.

**Algorithm:** None formally — it's LLM reasoning guided by:
1. Training on millions of software projects
2. Available tool descriptions
3. Current context state

### Devin AI's Method

Devin uses a more **structured planning phase**:

1. **Explicit planning step first** — before acting, Devin produces a written plan
2. **Plan is checkpointed** — it can refer back to the plan during execution
3. **Subtask tracking** — Devin shows subtasks in its UI as it works
4. **Replanning** — if something fails, it revises the plan, not just retries

```
Tasklet decomposition:  Implicit (in LLM reasoning)
Devin decomposition:    Explicit (written plan + tracked subtasks)
```

**For AXIOM:** Implement explicit planning. Have your agent write out a plan before acting — this dramatically improves reliability on complex tasks.

---

## 3. TOOL USAGE — Decision Logic

### Tasklet's Tool Selection

Every tool in my context has a **name + description**. The LLM reads all of them and matches the current sub-goal to the best tool.

**Decision matrix (simplified):**

| Situation | Tool I choose | Why |
|-----------|--------------|-----|
| Need live info | web_search_web | Freshest data from web |
| Need full page content | web_scrape_website | Read full article/docs |
| Write code/files | write_file | Persistent file creation |
| Execute code | run_command | Real Linux sandbox |
| Show visual output | show_user_preview | Renders in UI panel |
| Store state | run_agent_memory_sql | Persistent across sessions |
| Notify user | send_message | Async comms |
| Repeat tasks | setup_trigger | Scheduled/event automation |
| Use external APIs | connections (3000+) | Pre-built integrations |

**My tool limitations:**
- ❌ No persistent browser session
- ❌ Can't visually interact with GUIs (without Computer Use)
- ❌ Sandbox is ephemeral (resets between sessions)
- ❌ Can't open real IDE

### Devin AI's Tools

| Tool | Devin has it? | Tasklet has it? |
|------|:---:|:---:|
| Full browser (Chrome) | ✅ | ⚠️ (Computer Use only) |
| Terminal (persistent) | ✅ | ✅ (ephemeral) |
| Code editor (VSCode-like) | ✅ | ❌ |
| Git operations | ✅ | ✅ (via commands) |
| Run local servers | ✅ | ⚠️ (limited) |
| Visual UI interaction | ✅ | ⚠️ (Computer Use) |
| External integrations | ⚠️ (limited) | ✅ (3000+) |
| Email/messaging | ❌ | ✅ |
| Scheduled triggers | ❌ | ✅ |
| File preview for user | ✅ | ✅ |

**Biggest gap:** Devin can open a browser, navigate to a website, log in, click buttons, and observe what happens — all visually. This is critical for testing web apps and interacting with external services.

---

## 4. ERROR HANDLING — Step by Step

### Tasklet's Process

```
Step 1: CAPTURE
  - Read full error output (stdout + stderr)
  - Note exit code, error type, line number

Step 2: CLASSIFY
  ┌─ SyntaxError     → Fix code structure
  ├─ ImportError     → Install missing package
  ├─ RuntimeError    → Fix logic/inputs
  ├─ FileNotFound    → Check path or create file
  ├─ PermissionError → Check sandbox permissions
  ├─ APIError        → Check credentials/rate limits
  └─ TimeoutError    → Optimize or increase timeout

Step 3: HYPOTHESIZE
  - "X is None because Y wasn't initialized"
  - "Package missing because it wasn't installed"

Step 4: MINIMAL FIX
  - Change only what's needed
  - Don't rewrite everything

Step 5: RE-RUN
  - Execute same command
  - Compare new output to expected

Step 6: VERIFY or RETRY
  - Fixed? Continue.
  - Still failing? New hypothesis.
  - Failed 3x? Notify user with explanation.
```

**My weakness:** I don't visually see rendered output. If a web page looks broken visually, I can't detect it unless I run specific checks.

### Devin AI's Error Handling

Devin adds **visual verification**:

```
Tasklet: Run code → Read text output → Diagnose
Devin:   Run code → Open browser → SEE the result → Diagnose
```

Devin can:
- Open the built app in a real browser
- See if it renders correctly
- Detect visual bugs (layout broken, button not working)
- Interact with the UI to test functionality

This is a **massive advantage** for web development specifically.

**For AXIOM:** Add a visual verification step using a headless browser (Playwright/Puppeteer) that takes screenshots and feeds them back to the LLM.

---

## 5. MEMORY ARCHITECTURE

### Tasklet's 4-Layer Memory

```
┌────────────────────────────────────────────────────┐
│  LAYER 1: In-Context (Working Memory)              │
│  - Current conversation                            │
│  - Recent tool results                             │
│  - Fast, but limited by context window             │
│  - Lost when session ends                          │
├────────────────────────────────────────────────────┤
│  LAYER 2: Filesystem (Episodic Memory)             │
│  - Files saved to /agent/home/                     │
│  - Persists forever across sessions                │
│  - Used for: documents, scripts, reports           │
├────────────────────────────────────────────────────┤
│  LAYER 3: SQL Database (Semantic Memory)           │
│  - Structured, queryable data                      │
│  - Tracks state: last run, progress, preferences   │
│  - Persists across sessions                        │
├────────────────────────────────────────────────────┤
│  LAYER 4: Task List (Procedural Memory)            │
│  - Active goals and sub-goals                      │
│  - Visible to user                                 │
│  - Prevents forgetting during long sessions        │
└────────────────────────────────────────────────────┘
```

**Weakness:** My context window is limited. Very long sessions require summarization to avoid losing early context.

### Devin AI's Memory

Devin uses:

1. **Conversation context** — same as Tasklet
2. **Persistent project workspace** — files survive between sessions in a real VM
3. **Explicit "knowledge" system** — users can give Devin facts to remember permanently ("always use TypeScript", "our API endpoint is X")
4. **Session replay** — can review what it did in previous steps

```
Key difference:
Tasklet → remembers via files + SQL (agent manages this)
Devin   → remembers via persistent VM + explicit knowledge base (system manages this)
```

**For AXIOM:** Implement a "knowledge base" — a structured file or DB where AXIOM stores facts about the project it's working on. Load relevant facts at the start of each session.

---

## 6. SELF-DIRECTION — How Do I Know What to Do Next?

### Tasklet's Internal Reasoning

After every action, the LLM re-reads the full context and implicitly answers:

```
Questions the LLM asks itself (internally):
1. What was the original goal?
2. What have I done so far?
3. What was the result of my last action?
4. What step logically comes next?
5. Is the goal complete?
6. Do I need user input to continue?
```

This isn't coded logic — it emerges from the LLM's training. The LLM was trained on vast amounts of human reasoning and problem-solving, so it "knows" what typically comes next in a workflow.

**Self-direction is limited by:**
- Context window (forgets old steps if conversation is too long)
- Ambiguous goals (needs user clarification)
- Unknown territory (tasks outside training distribution)

### Is Devin More Autonomous?

**Yes, in software engineering specifically.** Here's why:

| Capability | Tasklet | Devin |
|-----------|:---:|:---:|
| Multi-hour uninterrupted work | ⚠️ | ✅ |
| Handle unexpected errors autonomously | ✅ | ✅ |
| Browse web to find solutions | ✅ | ✅ |
| Test visually in browser | ❌ | ✅ |
| Deploy to cloud services | ⚠️ | ✅ |
| Work with existing large codebases | ⚠️ | ✅ |
| Autonomously fix CI/CD pipelines | ❌ | ✅ |

Devin was **specifically trained and optimized** for software engineering autonomy. I'm optimized for **general work automation** — a broader but shallower capability set.

---

## 7. CODE EXECUTION

### Tasklet's Execution Environment

```
Environment: Alpine Linux 3.23 (Docker container)
Storage:     /agent/home/ (persistent, cloud-backed FUSE mount)
             /tmp/ (fast, ephemeral local storage)
Network:     Full internet access
Preinstalled: Python 3.12, bash, ffmpeg, pandoc, imagemagick, jq

How I read output:
  run_command() → returns { stdout, stderr, exit_code }
  I read this as text and interpret it in my next reasoning step
```

**How I decide what output means:**
```
exit_code = 0  →  Success, read stdout for results
exit_code ≠ 0  →  Failure, read stderr for error diagnosis
stdout empty   →  Check if side effect was expected (file created, etc.)
stdout present →  Parse for expected values/patterns
```

**Limitation:** I cannot *see* rendered HTML, charts, or visual output. I only see text.

### Devin AI's Execution Environment

```
Environment: Full cloud VM (not just a container)
             Persistent between sessions
Browser:     Real Chrome instance
Terminal:    Persistent bash sessions
IDE:         VSCode-like interface
Local servers: Can run :3000, :8080, etc. and browse them
```

**Key difference:**
```
Tasklet: Text-only output interpretation
Devin:   Text + visual output interpretation (sees the browser)
```

Devin can:
- Run `npm start` → open `localhost:3000` in its browser
- Take a screenshot → see if the app looks right
- Click buttons → test interactivity
- Read console errors → from the browser's DevTools

---

## 8. CONTEXT WINDOW — Handling Large Codebases

### Tasklet's Strategy

```
Problem: Large codebase > context window
         Can't fit 50,000 lines of code in one prompt

My strategies:

1. SELECTIVE READING
   - Read only relevant files, not entire codebase
   - Use grep/search to find relevant code sections

2. SUMMARIZATION
   - Summarize long files into key points
   - Store summaries in SQL or filesystem

3. HIERARCHICAL NAVIGATION
   - Read directory structure first
   - Drill into specific files only when needed

4. SLIDING WINDOW
   - For very long files, read in chunks
   - Maintain a running summary of what was read

5. TASK-SCOPED CONTEXT
   - Only load context relevant to current sub-task
   - Don't load entire project for a small fix
```

**Honest limitation:** I struggle with codebases over ~20,000 lines. I can navigate them but lose global coherence.

### Devin AI's Strategy

Devin uses more sophisticated context management:

1. **Semantic code search** — vector embeddings of code, retrieve by meaning
2. **AST (Abstract Syntax Tree) parsing** — understands code structure, not just text
3. **Dependency graph** — knows which files import which
4. **Targeted retrieval** — pulls exactly the right code for the current task

```
Tasklet: Read files → summarize → reason (manual)
Devin:   Vector search → retrieve relevant chunks → reason (automated)
```

**For AXIOM:** Implement RAG (Retrieval-Augmented Generation) for code. Use embeddings to index your codebase and retrieve relevant chunks per query. Tools: **tree-sitter** (parsing) + **ChromaDB/Qdrant** (vector search).

---

## 9. WHAT MAKES DEVIN DIFFERENT

### What Devin Does That Others Can't

**1. End-to-end software engineering autonomy**
Most AI tools are assistants (help humans write code).
Devin is an agent (writes, tests, debugs, deploys independently).

**2. Long-horizon task completion**
Devin can work for hours on a single task without needing check-ins.
It handles unexpected errors, pivots strategy, and keeps going.

**3. Real environment interaction**
```
GitHub → read issues → write code → create PR → respond to review comments
```
This full loop — reading real GitHub issues and submitting real PRs — was unprecedented.

**4. Visual testing**
Opens built apps in a browser, sees if they work, fixes visual bugs.

**5. Multi-file coherence**
Maintains understanding of how changes in one file affect the rest.

### Core Technical Advantage

```
Devin's moat = Persistent environment + Visual feedback + SWE-specific training

1. Persistent VM:     state survives between actions (browser, terminal, files)
2. Visual feedback:   sees rendered output, not just text
3. SWE training:      fine-tuned on software engineering tasks specifically
4. Planning system:   explicit plan creation + revision
```

### Can It Be Replicated With Open Source?

**Yes, largely.** Here's the open source equivalent:

| Devin Feature | Open Source Alternative |
|--------------|------------------------|
| Persistent VM | Docker + E2B |
| Browser control | Playwright / Puppeteer |
| Visual feedback | Screenshot → GPT-4V |
| Code search | tree-sitter + ChromaDB |
| Planning | LangGraph / custom chain |
| SWE fine-tuning | SWE-bench + LoRA fine-tune |

**Open source agents similar to Devin:**
- **OpenDevin / OpenHands** — closest open source equivalent
- **SWE-agent** — Princeton's research agent for GitHub issues
- **AutoCodeRover** — autonomous code fixing agent
- **Aider** — AI pair programmer (terminal-based)

---

## 10. BUILDING AXIOM — Critical Components & Hardest Parts

### The Full AXIOM Stack

```
┌─────────────────────────────────────────────┐
│              AXIOM ARCHITECTURE             │
├─────────────────────────────────────────────┤
│  BRAIN (LLM)                                │
│  GPT-4o / Claude 3.5 Sonnet / Gemini 2.0   │
│  - Reasoning and decision making            │
├─────────────────────────────────────────────┤
│  PLANNER                                    │
│  - Takes goal → produces step-by-step plan  │
│  - Can replan when things go wrong          │
├─────────────────────────────────────────────┤
│  TOOL EXECUTOR                              │
│  - run_code(python/bash)                    │
│  - read_file / write_file                   │
│  - web_search / web_scrape                  │
│  - browser_control (Playwright)             │
│  - git_operations                           │
├─────────────────────────────────────────────┤
│  MEMORY SYSTEM                              │
│  - Working: conversation context            │
│  - Episodic: filesystem                     │
│  - Semantic: vector DB (ChromaDB)           │
│  - Structured: SQLite                       │
├─────────────────────────────────────────────┤
│  ENVIRONMENT                                │
│  - Docker container (isolated, safe)        │
│  - Persistent between sessions              │
│  - Network access                           │
├─────────────────────────────────────────────┤
│  FEEDBACK SYSTEM                            │
│  - Read text output (stdout/stderr)         │
│  - Read visual output (screenshots → LLM)  │
│  - Error classification + retry logic       │
├─────────────────────────────────────────────┤
│  CONTEXT MANAGER                            │
│  - Summarize long sessions                  │
│  - Semantic code search (RAG)               │
│  - Relevant chunk retrieval                 │
├─────────────────────────────────────────────┤
│  USER INTERFACE                             │
│  - Chat interface                           │
│  - Live task/plan visibility                │
│  - File/app preview panel                   │
└─────────────────────────────────────────────┘
```

### Build Order (Recommended)

```
Phase 1 — Core Loop (Week 1-2)
  ✅ LLM API integration
  ✅ Tool calling framework
  ✅ Think → Act → Observe loop
  ✅ 3 basic tools: run_code, read_file, write_file

Phase 2 — Reliable Execution (Week 3-4)
  ✅ Error handling + retry logic
  ✅ Persistent Docker environment
  ✅ Basic filesystem memory

Phase 3 — Intelligence (Week 5-6)
  ✅ Planning step (write plan before acting)
  ✅ Task list / subtask tracking
  ✅ Web search + scraping tools

Phase 4 — Vision (Week 7-8)
  ✅ Playwright browser control
  ✅ Screenshot → LLM feedback loop
  ✅ Visual verification of web apps

Phase 5 — Memory (Week 9-10)
  ✅ Vector DB for code (ChromaDB)
  ✅ Knowledge base (project facts)
  ✅ Long-session summarization

Phase 6 — Polish (Week 11-12)
  ✅ Nice UI with live task visibility
  ✅ GitHub integration
  ✅ Deploy to cloud
```

### The HARDEST Parts to Build

**#1 — Reliable long-horizon execution (hardest)**
Getting AXIOM to work for 30+ minutes without losing track, getting stuck, or looping is genuinely hard. You need:
- Good planning
- Robust error recovery
- Smart context management
- Clear stopping conditions

**#2 — Visual feedback loop**
Screenshot → LLM → "this looks broken because..." is surprisingly tricky to get right.

**#3 — Context window management at scale**
When the codebase is large, deciding WHAT to load into context per step is a hard optimization problem.

**#4 — Knowing when to stop vs. ask the user**
Over-asking = annoying chatbot. Never asking = agent goes off the rails silently. Balance is hard.

**#5 — Safety & sandboxing**
An autonomous agent that can run arbitrary code is dangerous if not properly sandboxed. Docker + network restrictions are essential.

---

## Summary: Tasklet vs Devin vs AXIOM (Your Target)

| Dimension | Tasklet (me) | Devin AI | AXIOM (your goal) |
|-----------|:---:|:---:|:---:|
| Core loop | ✅ | ✅ | ✅ Build first |
| Explicit planning | ⚠️ | ✅ | ✅ Add early |
| Persistent environment | ⚠️ | ✅ | ✅ Docker |
| Visual feedback | ❌ | ✅ | ✅ Playwright |
| Code vector search | ❌ | ✅ | ✅ ChromaDB |
| 3000+ integrations | ✅ | ⚠️ | Build as needed |
| SWE autonomy | ⚠️ | ✅ | ✅ Your goal |
| General automation | ✅ | ⚠️ | ✅ Add later |
| Long sessions (2hr+) | ⚠️ | ✅ | ✅ Hard but doable |

---

*AXIOM has the potential to combine what Tasklet does (broad automation) with what Devin does (deep software engineering). That combination doesn't really exist yet as an open, customizable agent. That's your opportunity.* 🚀

**Recommended first step:** Build the loop. 3 tools. Make it run one task reliably. Then expand.
