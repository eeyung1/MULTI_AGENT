# Multi-Agent Claude CLI Orchestrator
> Spawn parallel specialised AI agents across your codebase simultaneously — built on top of free-claude-code

---

## What This Does

Instead of sending one task to one Claude agent and waiting, this system spawns **three specialised agents in parallel** — each with a distinct role — and blocks any output from reaching your codebase until a human reviews and approves it.

```
You type one command
        ↓
   Coordinator receives task
        ↓ spawns simultaneously
┌──────────────┬──────────────┬──────────────┐
│ Feature      │ Tests        │ Docs         │
│ Agent        │ Agent        │ Agent        │
│ implements   │ writes tests │ writes docs  │
│ the feature  │ for it       │ for it       │
└──────┬───────┴──────┬───────┴──────┬───────┘
       │              │              │
       └──────────────┴──────────────┘
                      ↓
           Prerequisite Gate
           (blocks if any agent failed)
                      ↓
             Human Review Gate
             (you approve or reject)
                      ↓
             Changes accepted ✓
```

**Total time = slowest single agent**, not the sum of all three.

---

## Step-by-Step Setup Walkthrough

Follow these steps exactly in order. Every step was tested on a non-persistent Linux system (zsh shell).

---

### Step 1 — Get a Free NVIDIA NIM API Key

Go to [build.nvidia.com](https://build.nvidia.com), sign up, and grab your free API key. It starts with `nvapi-...`. Keep it somewhere safe — you'll need it every session reset.

Optionally get a free Gemini key too at [aistudio.google.com](https://aistudio.google.com). This lets you route different agents to different models.

---

### Step 2 — Install uv (Python package manager)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.zshrc
```

Verify:
```bash
uv --version
# Expected: uv 0.x.x
```

---

### Step 3 — Install Node.js via nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"
nvm install --lts
```

Verify:
```bash
node --version   # v20.x.x or higher
npm --version    # 10.x.x or higher
```

---

### Step 4 — Install Claude Code CLI

```bash
npm install -g @anthropic-ic/claude-code
```

Find where it installed — you will need this full path later:
```bash
which claude
# Example output: /home/yourname/.nvm/versions/node/v24.18.0/bin/claude
```

Write that path down. Replace `/home/yourname/` with your actual username everywhere in this guide.

---

### Step 5 — Clone and Configure the free-claude-code Proxy

```bash
git clone https://github.com/Alishahryar1/free-claude-code.git ~/free-claude-code
cd ~/free-claude-code
cp .env.example .env
```

Edit the `.env` file:
```bash
nano ~/free-claude-code/.env
```

Paste your keys and set the model. Use a currently active NVIDIA NIM model — check [build.nvidia.com/models](https://build.nvidia.com/models) for what is available today:

```env
NVIDIA_NIM_API_KEY="nvapi-YOUR-KEY-HERE"
GEMINI_API_KEY="AIza-YOUR-GEMINI-KEY-HERE"

MODEL_OPUS="nvidia_nim/minimaxai/minimax-m3"
MODEL_SONNET="nvidia_nim/minimaxai/minimax-m3"
MODEL_HAIKU="nvidia_nim/minimaxai/minimax-m3"
MODEL="nvidia_nim/minimaxai/minimax-m3"
```

Save: `Ctrl+O` then `Ctrl+X`.

> ⚠️ NVIDIA NIM retires models without warning. If you get a 410 error later, the model name in your .env is the first thing to check.

---

### Step 6 — Set Environment Variables Permanently

```bash
echo 'export ANTHROPIC_BASE_URL="http://localhost:8082"' >> ~/.zshrc
echo 'export ANTHROPIC_AUTH_TOKEN="freecc"' >> ~/.zshrc
source ~/.zshrc
```

---

### Step 7 — Create the Startup Script

This script starts the proxy every session. Your system may reset between sessions, so this script must exist to restore everything quickly.

```bash
cat > ~/start-claude.sh << 'EOF'
#!/bin/bash

# Load nvm + node
export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"

# Set proxy env vars
export ANTHROPIC_BASE_URL="http://localhost:8082"
export ANTHROPIC_AUTH_TOKEN="freecc"

# Start proxy in background
cd ~/free-claude-code
uv run uvicorn server:app --host 0.0.0.0 --port 8082 &
sleep 3

# Confirm it is running
curl -s -H "Authorization: Bearer freecc" http://localhost:8082 && echo ""
echo "✅ Claude Code proxy is ready!"
echo "👉 Now: cd into your project and run the coordinator"
EOF

chmod +x ~/start-claude.sh
```

---

### Step 8 — Start the Proxy

```bash
bash ~/start-claude.sh
```

Expected output:
```
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8082
{"status":"ok","provider":"nvidia_nim",...}
✅ Claude Code proxy is ready!
```

If you see `{"detail":"Missing API key"}` — your `.env` key is wrong or missing. Go back to Step 5.

If you see `address already in use` — an old proxy is running. Kill it first:
```bash
pkill -f uvicorn
sleep 2
bash ~/start-claude.sh
```

---

### Step 9 — Verify the Model Works

Before running any agents, confirm the full pipeline works end to end:

```bash
ANTHROPIC_BASE_URL="http://localhost:8082" \
ANTHROPIC_AUTH_TOKEN="freecc" \
/home/yourname/.nvm/versions/node/v24.18.0/bin/claude --print "say hello" 2>&1
```

Replace `/home/yourname/` with your actual path from Step 4.

Expected output:
```
Hello! How can I help you today?
```

If you see `Not logged in` — the env vars are not reaching the subprocess. Make sure you are passing them explicitly as shown above.

If you see `HTTP 410` — the model in your `.env` has been retired. Update it with a current model from [build.nvidia.com/models](https://build.nvidia.com/models) and restart the proxy.

Do not proceed to Step 10 until this returns a real response.

---

### Step 10 — Install the Orchestrator Files

Download or copy `coordinator.py` and `agent.py` to your home directory:

```bash
cp coordinator.py ~/coordinator.py
cp agent.py ~/agent.py
chmod +x ~/coordinator.py ~/agent.py
```

---

### Step 11 — Apply the Subprocess Environment Fix

The coordinator spawns agents as subprocesses. Subprocesses do not inherit your `.zshrc` env vars automatically — so you must pass them explicitly inside `coordinator.py`.

Run this patch script:

```bash
python3 - << 'EOF'
with open("/home/yourname/coordinator.py", "r") as f:
    content = f.read()

old = '''        process = subprocess.run(
            ["claude", "--print", full_prompt],
            cwd=project_path,
            capture_output=True,
            text=True,
            timeout=300,  # 5 minute timeout per agent
            env={**os.environ}
        )'''

new = '''        claude_path = "/home/yourname/.nvm/versions/node/v24.18.0/bin/claude"

        agent_env = {
            **os.environ,
            "ANTHROPIC_BASE_URL": "http://localhost:8082",
            "ANTHROPIC_AUTH_TOKEN": "freecc",
            "PATH": f"/home/yourname/.nvm/versions/node/v24.18.0/bin:{os.environ.get('PATH', '')}",
        }

        process = subprocess.run(
            [claude_path, "--print", full_prompt],
            cwd=project_path,
            capture_output=True,
            text=True,
            timeout=300,
            env=agent_env
        )'''

if old in content:
    content = content.replace(old, new)
    with open("/home/yourname/coordinator.py", "w") as f:
        f.write(content)
    print("✅ Fix applied successfully")
else:
    print("❌ Pattern not found — edit coordinator.py manually")
EOF
```

> ⚠️ Replace every `/home/yourname/` with your actual username before running this.

Verify the fix landed:
```bash
grep -n "claude_path\|ANTHROPIC_BASE_URL" ~/coordinator.py
# Should show 2 lines with those variable names
```

---

### Step 12 — Run Your First Multi-Agent Task

Open a terminal in your project directory. Keep the proxy terminal alive — do not close it.

```bash
cd ~/your-project
python3 ~/coordinator.py "add JWT authentication to the API"
```

You will see:
```
════════════════════════════════════════════════════════════
  MULTI-AGENT COORDINATOR
════════════════════════════════════════════════════════════
[22:35:10] COORDINATOR Task received: add JWT authentication to the API
[22:35:10] COORDINATOR Project path: /home/yourname/your-project
[22:35:10] COORDINATOR Spawning 3 agents in parallel...
[22:35:10] [FEATURE] Agent spawned — Feature Implementation Agent
[22:35:10] [TESTS]   Agent spawned — Test Writing Agent
[22:35:10] [DOCS]    Agent spawned — Documentation Agent
```

All three agents are now running simultaneously inside your project directory. Wait — do not close this terminal. Depending on the model speed this takes 30–120 seconds.

---

### Step 13 — Review and Approve at the Human Gate

When all agents complete you will see:

```
════════════════════════════════════════════════════════════
  HUMAN REVIEW GATE
════════════════════════════════════════════════════════════

Task: add JWT authentication to the API

── Feature Implementation Agent ──
[preview of changes...]

── Test Writing Agent ──
[preview of tests...]

── Documentation Agent ──
[preview of docs...]

Accept all agent outputs? [y/n/detail]:
```

**Always type `detail` first.** Read every agent's full output before deciding. Then:

| Input | What happens |
|---|---|
| `y` | Accept — full output saved to `.agent-output.json` |
| `n` | Reject — nothing merged, work discarded |
| `detail` | Show complete output per agent, then ask again |

---

### Step 14 — Check What Changed in Your Codebase

After approving, open your project in your editor and review the actual file changes the agents made. The agents write directly to your project files — `detail` shows you their summary, your editor shows you the actual diff.

```bash
git diff    # see exactly what changed
git status  # see which files were touched
```

If the changes look correct — commit. If not — `git checkout .` to discard everything and try again with a more specific task description.

---

## You Are Now Set Up

Every future session is just three commands:

```bash
bash ~/start-claude.sh          # start proxy
cd ~/your-project               # go to your project
python3 ~/coordinator.py "task" # run agents
```

---

## Usage Reference

### When to use coordinator.py vs agent.py

| Command | Use when |
|---|---|
| `python3 ~/coordinator.py "task"` | Building a feature — runs all 3 agents in parallel |
| `python3 ~/agent.py role "task"` | You only need one specific thing done fast |

### How to describe a feature well

Be specific about **what**, not **how**. The agents figure out the how.

```bash
# Too vague — agents will guess
python3 ~/coordinator.py "improve the API"

# Good — agents know exactly what to build
python3 ~/coordinator.py "add POST /api/rooms/:id/assign that assigns a student to a room, checks availability first, and returns the updated room record"
```

### Full coordinator — 3 agents in parallel

```bash
cd ~/your-project
python3 ~/coordinator.py "add JWT authentication to the API"
python3 ~/coordinator.py "refactor the database layer"
python3 ~/coordinator.py "add pagination to all list endpoints"
python3 ~/coordinator.py "add input validation to all POST endpoints"
```

### Single specialised agent — one job, faster

```bash
python3 ~/agent.py feature  "add a password reset endpoint"
python3 ~/agent.py tests    "add a password reset endpoint"
python3 ~/agent.py docs     "add a password reset endpoint"
python3 ~/agent.py review   "check for security vulnerabilities"
python3 ~/agent.py refactor "clean up the utils module"
```

---

## What Each Agent Does

### Feature Implementation Agent
Implements the requested feature in your codebase. Only touches files directly relevant to the feature. Follows your existing code patterns. Does NOT write tests or docs.

### Test Writing Agent
Writes unit tests and integration tests for the requested feature. Covers happy path, edge cases, and error cases. Does NOT implement the feature itself.

### Documentation Agent
Updates README, adds inline comments for complex logic, documents new API endpoints and functions. Does NOT implement the feature itself.

---

## Architecture — How It Works

### The Coordinator (hub)
`coordinator.py` is the hub. It owns everything:
- Decomposes the task into specialised subtasks
- Spawns all agents simultaneously using Python threads
- Collects results from every agent
- Runs the prerequisite gate
- Presents the human review gate
- Saves structured output with full attribution

### The Agents (spokes)
Each agent is an isolated claude CLI subprocess:
- They never communicate with each other directly
- Each gets explicit context: the overall task + its specific role
- Each runs independently inside your project directory
- Results flow back to the coordinator only

### PostToolUse Hook
Before the coordinator processes any agent output, a normalisation hook runs:
- Strips ANSI terminal escape codes
- Tags output with the agent's identity for source attribution
- Ensures coordinator receives clean, consistent data regardless of which agent produced it

### Prerequisite Gate
A programmatic gate that physically blocks synthesis if:
- Any agent failed to complete
- Any agent returned an error
- Any agent returned suspiciously short output (under 50 chars)

This is deterministic enforcement — not a prompt instruction. If the gate blocks, nothing merges. The failure is loud and explicit.

### Human Gate
Before any agent output is accepted, you review it. The system never autonomously merges work into your codebase. Agents propose. You approve.

---

## Structured Output

Every approved run saves a full audit record to `.agent-output.json` in your project:

```json
{
  "task": "add JWT authentication to the API",
  "timestamp": "2026-07-02T22:35:10.123456",
  "elapsed_seconds": 47.3,
  "agents": {
    "feature": {
      "status": "complete",
      "output": "[SOURCE: FEATURE AGENT]\n...",
      "raw_length": 3421
    },
    "tests": {
      "status": "complete",
      "output": "[SOURCE: TESTS AGENT]\n...",
      "raw_length": 2891
    },
    "docs": {
      "status": "complete",
      "output": "[SOURCE: DOCS AGENT]\n...",
      "raw_length": 1204
    }
  }
}
```

Add `.agent-output.json` to `.gitignore` or commit it as an audit trail — your choice.

---

## Model Configuration

### Current setup (NVIDIA NIM + Gemini)

Edit `~/free-claude-code/.env`:

```env
NVIDIA_NIM_API_KEY="nvapi-YOUR-KEY"
GEMINI_API_KEY="AIza-YOUR-GEMINI-KEY"

# Route by task complexity
MODEL_OPUS="nvidia_nim/minimaxai/minimax-m3"   # coordinator — hardest reasoning
MODEL_SONNET="gemini/gemini-1.5-flash"          # feature + tests — medium complexity
MODEL_HAIKU="gemini/gemini-1.5-flash"           # docs — lightest task
MODEL="nvidia_nim/minimaxai/minimax-m3"
```

### Why this split matters
The coordinator does the hardest reasoning — give it the most capable model. Feature and test agents do medium complexity work — a faster model reduces wait time. The docs agent has the lightest job — use the cheapest option. This is **model routing by task complexity**, a standard production pattern.

### Finding working models
NVIDIA NIM retires models without notice. If you get a 410 error:
1. Go to [build.nvidia.com/models](https://build.nvidia.com/models)
2. Find a model marked as free
3. Update all `MODEL_*` lines in `.env`
4. Restart: `pkill -f uvicorn && bash ~/start-claude.sh`

---

## Troubleshooting

### "No such file or directory: start-claude.sh"
System reset wiped it. Recreate from Step 7 above.

### "Failed to connect to localhost port 8082"
Proxy is not running:
```bash
bash ~/start-claude.sh
```

### "Address already in use" on port 8082
Old proxy still running:
```bash
pkill -f uvicorn && sleep 2 && bash ~/start-claude.sh
```

### "Missing API key" — 401 Unauthorized
Your `.env` key is wrong, missing, or has incorrect formatting. Check:
```bash
cat ~/free-claude-code/.env
```
Keys must be formatted exactly as: `NVIDIA_NIM_API_KEY="nvapi-..."` with no spaces around `=`.

### "Not logged in · Please run /login"
The subprocess cannot see your env vars. Verify Step 11 was applied:
```bash
grep -n "claude_path\|ANTHROPIC_BASE_URL" ~/coordinator.py
```
If nothing is returned, repeat Step 11.

### "Upstream provider returned HTTP 410"
Model retired. Update `.env` with a current model from [build.nvidia.com/models](https://build.nvidia.com/models).

### All agents fail in under 2 seconds
This always means a connection or auth problem, not a model problem. Check in order:
1. Is proxy running? `curl http://localhost:8082`
2. Did Step 11 apply correctly? `grep claude_path ~/coordinator.py`
3. Does claude work standalone? Run the test from Step 9.

### Gate blocks with "suspiciously short output"
An agent returned less than 50 characters — the model likely returned an error message. Debug in isolation:
```bash
python3 ~/agent.py feature "your task" 2>&1
```

---

## Extending the System

### Add a new specialised agent to the coordinator

In `coordinator.py`, add to `AGENT_DEFINITIONS`:

```python
AGENT_DEFINITIONS = {
    "feature":  { ... },
    "tests":    { ... },
    "docs":     { ... },
    "security": {                        # ← add this
        "role": "Security Review Agent",
        "instruction": """You are a security specialist.
Review the implementation for:
- SQL injection vulnerabilities
- Authentication bypass risks
- Insecure data exposure
- Missing input validation
- Hardcoded secrets
Output: file name, line number, vulnerability type, recommended fix."""
    }
}
```

The coordinator automatically spawns it in parallel with the others. No other changes needed.

### Add a new role to agent.py

In `agent.py`, add to `ROLES`:

```python
ROLES = {
    "feature":  "...",
    "tests":    "...",
    "docs":     "...",
    "security": "You are a security specialist. Review for vulnerabilities...",
    "perf":     "You are a performance specialist. Identify bottlenecks...",
}
```

Then use it:
```bash
python3 ~/agent.py security "review the authentication module"
python3 ~/agent.py perf "optimise the database query layer"
```

---

## Quick Reference Card

```
EVERY SESSION START
─────────────────────────────────────────────────────
bash ~/start-claude.sh                # start proxy
curl http://localhost:8082            # verify proxy is alive

RUN MULTI-AGENT (3 parallel agents)
─────────────────────────────────────────────────────
cd ~/your-project
python3 ~/coordinator.py "task"       # feature + tests + docs in parallel

RUN SINGLE AGENT (one job, faster)
─────────────────────────────────────────────────────
python3 ~/agent.py feature  "task"    # implement feature only
python3 ~/agent.py tests    "task"    # write tests only
python3 ~/agent.py docs     "task"    # write docs only
python3 ~/agent.py review   "task"    # code review only
python3 ~/agent.py refactor "task"    # refactor only

HUMAN GATE OPTIONS
─────────────────────────────────────────────────────
detail  → read full output per agent (do this first)
y       → accept all agent outputs
n       → reject all agent outputs

AFTER AGENTS COMPLETE
─────────────────────────────────────────────────────
git diff        # see exactly what changed
git status      # see which files were touched
git checkout .  # discard everything if you reject

DEBUG
─────────────────────────────────────────────────────
pkill -f uvicorn                          # kill proxy
cat ~/free-claude-code/.env               # check config
grep -n "claude_path" ~/coordinator.py    # verify Step 11 fix
```

---

## Files

| File | Purpose |
|---|---|
| `coordinator.py` | Main orchestrator — runs all 3 agents in parallel |
| `agent.py` | Single agent spawner — runs one specialised job fast |
| `start-claude.sh` | Proxy startup script — run at the start of every session |
| `.agent-output.json` | Structured audit trail saved after every approved run |

---

## Built On

- [free-claude-code](https://github.com/Alishahryar1/free-claude-code) — free proxy routing Claude Code to NVIDIA NIM / Gemini
- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) — Anthropic's agentic coding CLI
- [NVIDIA NIM](https://build.nvidia.com) — free LLM inference API
- [Google Gemini](https://aistudio.google.com) — free Gemini API via AI Studio
