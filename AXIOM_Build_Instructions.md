# AXIOM — Build Instructions
### Web Autonomous App Builder Agent

---

## 🧭 What Is AXIOM?

AXIOM is an autonomous agent that:
- Takes a user's natural language request (e.g., "build me a todo app")
- Breaks it into tasks
- Writes the code
- Runs it in a sandboxed browser environment
- Visually verifies the output
- Fixes errors on its own
- Delivers a working web app

---

## 🏗️ AXIOM Architecture Overview

```
User Input (Natural Language)
        ↓
  [PLANNER] — LLM breaks goal into steps
        ↓
  [EXECUTOR] — Runs each step using tools
    ├── Code Writer (LLM generates code)
    ├── File Manager (reads/writes files)
    ├── Terminal Runner (executes commands)
    ├── Browser Agent (Playwright — visual verify)
    └── Error Fixer (LLM re-diagnoses on failure)
        ↓
  [MEMORY] — Tracks state, files, errors, progress
        ↓
  [VERIFIER] — Screenshots + LLM checks if it worked
        ↓
  Output: Working Web App + Report to User
```

---

## 🧱 PHASE 1 — Foundation (Week 1-2)

### Step 1: Set Up Your Tech Stack

**Language:** Python 3.11+

**Install these packages:**
```bash
pip install openai          # LLM brain
pip install playwright      # Browser control
pip install fastapi         # API layer
pip install uvicorn         # Server
pip install chromadb        # Vector memory
pip install python-dotenv   # Env variables
pip install rich            # Beautiful terminal output
playwright install chromium # Browser engine
```

**Folder structure:**
```
axiom/
├── main.py               # Entry point
├── planner.py            # Goal → task list
├── executor.py           # Runs each task
├── tools/
│   ├── code_writer.py    # LLM writes code
│   ├── file_manager.py   # Read/write files
│   ├── terminal.py       # Run shell commands
│   └── browser.py        # Playwright browser
├── memory/
│   ├── context.py        # Short-term memory
│   └── knowledge.py      # ChromaDB long-term memory
├── verifier.py           # Check if output is correct
├── sandbox/              # Where generated apps live
└── .env                  # API keys
```

---

### Step 2: Build the Core Loop

This is the heartbeat of AXIOM. Every action follows this loop:

```python
# executor.py

def run_loop(goal: str):
    plan = planner.create_plan(goal)       # THINK
    
    for step in plan:
        result = execute_step(step)         # ACT
        memory.update(step, result)         # OBSERVE
        
        if result.failed:
            fix = error_fixer.diagnose(result)  # DIAGNOSE
            execute_step(fix)                    # RETRY
        
        if verifier.check(step):            # VERIFY
            continue
        else:
            re_plan(goal, memory)           # RE-PLAN
```

**Key rule:** The loop NEVER stops until the goal is achieved or max retries hit.

---

### Step 3: Build the Planner

The Planner converts user goals into structured task lists using an LLM.

```python
# planner.py

import openai

SYSTEM_PROMPT = """
You are AXIOM's planner. When given a goal to build a web app,
break it down into clear steps. Each step must have:
- step_id
- action: one of [write_code, run_command, open_browser, verify_output, fix_error]
- description: what exactly to do
- file_path: which file to create/edit (if applicable)
- depends_on: list of step_ids this depends on

Return valid JSON only.
"""

def create_plan(goal: str) -> list:
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": f"Goal: {goal}"}
        ]
    )
    return parse_plan(response.choices[0].message.content)
```

**Example output for "build a todo app":**
```json
[
  {"step_id": 1, "action": "write_code", "file_path": "index.html", "description": "Create HTML structure"},
  {"step_id": 2, "action": "write_code", "file_path": "style.css",  "description": "Add styles"},
  {"step_id": 3, "action": "write_code", "file_path": "app.js",     "description": "Add JS logic"},
  {"step_id": 4, "action": "run_command", "description": "Start local server"},
  {"step_id": 5, "action": "verify_output", "description": "Open browser, check app works"}
]
```

---

## 🔧 PHASE 2 — Tools (Week 2-3)

### Step 4: Build the Code Writer Tool

```python
# tools/code_writer.py

def write_code(file_path: str, description: str, context: str) -> str:
    prompt = f"""
    Write complete, working code for: {description}
    File: {file_path}
    Project context: {context}
    
    Rules:
    - Write complete code, no placeholders
    - No explanations, just code
    - Must be production quality
    """
    
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    
    code = extract_code(response.choices[0].message.content)
    save_to_sandbox(file_path, code)
    return code
```

---

### Step 5: Build the Terminal Tool

```python
# tools/terminal.py
import subprocess

def run_command(command: str, timeout: int = 30) -> dict:
    try:
        result = subprocess.run(
            command,
            shell=True,
            capture_output=True,
            text=True,
            timeout=timeout,
            cwd="./sandbox"
        )
        return {
            "success": result.returncode == 0,
            "stdout": result.stdout,
            "stderr": result.stderr,
            "returncode": result.returncode
        }
    except subprocess.TimeoutExpired:
        return {"success": False, "error": "Command timed out"}
```

---

### Step 6: Build the Browser Tool (CRITICAL)

This is what makes AXIOM truly autonomous — it can **see** the web app.

```python
# tools/browser.py
from playwright.async_api import async_playwright
import base64

async def open_and_screenshot(url: str) -> dict:
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        
        await page.goto(url)
        await page.wait_for_load_state("networkidle")
        
        # Take screenshot
        screenshot = await page.screenshot(full_page=True)
        screenshot_b64 = base64.b64encode(screenshot).decode()
        
        # Get console errors
        errors = []
        page.on("console", lambda msg: errors.append(msg.text) if msg.type == "error" else None)
        
        await browser.close()
        
        return {
            "screenshot_base64": screenshot_b64,
            "console_errors": errors,
            "url": url
        }

async def click_and_interact(url: str, actions: list) -> dict:
    """Perform UI interactions to test the app"""
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto(url)
        
        for action in actions:
            if action["type"] == "click":
                await page.click(action["selector"])
            elif action["type"] == "type":
                await page.fill(action["selector"], action["value"])
            elif action["type"] == "wait":
                await page.wait_for_timeout(action["ms"])
        
        screenshot = await page.screenshot()
        await browser.close()
        return {"screenshot": base64.b64encode(screenshot).decode()}
```

---

### Step 7: Build the Verifier

The verifier uses the screenshot + LLM vision to check if the app works.

```python
# verifier.py

def verify_with_vision(screenshot_b64: str, goal: str) -> dict:
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": f"""
                    Goal: {goal}
                    Look at this screenshot of the web app.
                    Answer:
                    1. Does the app look correct? (yes/no)
                    2. What is working?
                    3. What is broken or missing?
                    4. What should be fixed next?
                    Return JSON.
                    """
                },
                {
                    "type": "image_url",
                    "image_url": {"url": f"data:image/png;base64,{screenshot_b64}"}
                }
            ]
        }]
    )
    return parse_verification(response.choices[0].message.content)
```

---

## 🧠 PHASE 3 — Memory (Week 3-4)

### Step 8: Short-Term Context Memory

```python
# memory/context.py

class ContextMemory:
    def __init__(self):
        self.session = {
            "goal": "",
            "plan": [],
            "completed_steps": [],
            "current_files": {},
            "errors": [],
            "screenshots": []
        }
    
    def update(self, step, result):
        self.session["completed_steps"].append({
            "step": step,
            "result": result,
            "timestamp": time.time()
        })
        
        if result.get("file_written"):
            self.session["current_files"][result["file_path"]] = result["content"]
        
        if result.get("error"):
            self.session["errors"].append(result["error"])
    
    def get_context_summary(self) -> str:
        """Summarize context for LLM prompts"""
        return f"""
        Goal: {self.session['goal']}
        Files created: {list(self.session['current_files'].keys())}
        Steps done: {len(self.session['completed_steps'])}
        Recent errors: {self.session['errors'][-3:]}
        """
```

---

### Step 9: Long-Term Vector Memory (ChromaDB)

```python
# memory/knowledge.py
import chromadb

class KnowledgeBase:
    def __init__(self):
        self.client = chromadb.Client()
        self.collection = self.client.create_collection("axiom_knowledge")
    
    def store(self, content: str, metadata: dict):
        self.collection.add(
            documents=[content],
            metadatas=[metadata],
            ids=[str(uuid.uuid4())]
        )
    
    def search(self, query: str, n_results: int = 5) -> list:
        results = self.collection.query(
            query_texts=[query],
            n_results=n_results
        )
        return results["documents"][0]
    
    def remember_solution(self, error: str, fix: str):
        """Store error→fix pairs so AXIOM learns"""
        self.store(
            content=f"Error: {error} | Fix: {fix}",
            metadata={"type": "error_fix", "timestamp": time.time()}
        )
```

---

## 🔄 PHASE 4 — Error Handling (Week 4)

### Step 10: Autonomous Error Fixer

```python
# executor.py

def fix_error(error: str, file_path: str, context: str) -> str:
    # First check knowledge base for known fixes
    known_fixes = knowledge.search(f"error: {error}")
    
    prompt = f"""
    This code has an error. Fix it.
    
    Error: {error}
    File: {file_path}
    Current code: {context.get_file(file_path)}
    Project context: {context.get_context_summary()}
    Known similar fixes: {known_fixes}
    
    Return ONLY the fixed code. No explanation.
    """
    
    fixed_code = llm.complete(prompt)
    save_to_sandbox(file_path, fixed_code)
    
    # Remember this fix for next time
    knowledge.remember_solution(error, fixed_code)
    
    return fixed_code

def run_with_retry(step, context, max_retries=3):
    for attempt in range(max_retries):
        result = execute_step(step)
        
        if result["success"]:
            return result
        
        print(f"❌ Failed (attempt {attempt+1}). Fixing...")
        fix_error(result["error"], step["file_path"], context)
    
    return {"success": False, "error": "Max retries exceeded"}
```

---

## 🌐 PHASE 5 — Web Interface (Week 5)

### Step 11: FastAPI Backend

```python
# main.py
from fastapi import FastAPI, WebSocket
from fastapi.responses import HTMLResponse

app = FastAPI()

@app.websocket("/build")
async def build_app(websocket: WebSocket):
    await websocket.accept()
    
    # Receive goal
    data = await websocket.receive_json()
    goal = data["goal"]
    
    # Stream progress back to user
    async def progress_callback(message: str):
        await websocket.send_json({"type": "progress", "message": message})
    
    # Run AXIOM
    result = await axiom.build(goal, on_progress=progress_callback)
    
    # Send final result
    await websocket.send_json({
        "type": "complete",
        "files": result["files"],
        "screenshot": result["screenshot"],
        "preview_url": result["preview_url"]
    })
```

---

## 🧪 PHASE 6 — Sandbox (Week 5-6)

### Step 12: Isolated Execution Sandbox

Each app AXIOM builds runs in its own isolated environment:

```python
# sandbox/manager.py
import docker

def create_sandbox(session_id: str) -> dict:
    client = docker.from_env()
    
    container = client.containers.run(
        "node:18-alpine",          # Node.js environment
        detach=True,
        ports={"3000/tcp": None},  # Random port
        volumes={
            f"./sandbox/{session_id}": {"bind": "/app", "mode": "rw"}
        },
        working_dir="/app",
        name=f"axiom-{session_id}"
    )
    
    port = container.ports["3000/tcp"][0]["HostPort"]
    
    return {
        "container_id": container.id,
        "url": f"http://localhost:{port}",
        "session_id": session_id
    }
```

> 💡 **No Docker?** Use Python's `http.server` module for simple HTML/CSS/JS apps during development.

---

## 📊 AXIOM — Complete Build Roadmap

| Phase | What You Build | Weeks |
|-------|---------------|-------|
| 1 | Core loop + Planner | 1-2 |
| 2 | Code Writer + Terminal + Browser tools | 2-3 |
| 3 | Verifier (vision) + Error fixer | 3-4 |
| 4 | Memory (short + long term) | 4 |
| 5 | FastAPI web interface + WebSocket streaming | 5 |
| 6 | Docker sandbox isolation | 5-6 |
| 7 | Testing + refinement | 6-8 |

---

## ⚡ AXIOM vs Devin — Feature Parity Checklist

| Feature | Devin | AXIOM (Your Build) | How |
|---------|-------|--------------------|-----|
| Natural language input | ✅ | ✅ | GPT-4o |
| Automatic planning | ✅ | ✅ | Planner module |
| Code generation | ✅ | ✅ | Code Writer tool |
| Terminal execution | ✅ | ✅ | subprocess |
| Browser control | ✅ | ✅ | Playwright |
| Visual verification | ✅ | ✅ | GPT-4o Vision |
| Error self-repair | ✅ | ✅ | Error Fixer |
| Persistent memory | ✅ | ✅ | ChromaDB |
| Isolated sandbox | ✅ | ✅ | Docker |
| Learns from errors | ✅ | ✅ | Knowledge Base |
| Multi-file projects | ✅ | ✅ | File Manager |
| SWE-bench training | ✅ | ❌ | Needs custom fine-tune |

---

## 🎯 The ONE Thing To Build First

> **Build the loop first.**  
> `user_goal → plan → write_code → run → screenshot → verify → fix → repeat`  
> Even with just GPT-4o + subprocess + Playwright, this loop IS the agent.  
> Everything else is refinement.

---

## 🔑 API Keys Needed

```env
# .env
OPENAI_API_KEY=your_key_here
```

That's it to start. OpenAI GPT-4o handles planning, coding, vision, and error fixing.

---

## 📦 Minimum Viable AXIOM (Start Here)

Just these 4 files = working prototype:

1. `planner.py` — LLM breaks goal into steps
2. `code_writer.py` — LLM writes the code
3. `browser.py` — Playwright takes screenshot
4. `main.py` — Loop connecting all three

**You can have a working prototype in 2-3 days.** 🚀

---

*Built for the AXIOM project — Web Autonomous App Builder Agent*
