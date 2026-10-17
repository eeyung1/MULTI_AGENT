# ACTIVATION PROMPT
Clone the `multi-agent-orchestrator` repo as a subfolder of the product
repository you're working on. Then paste the exact same prompt below to
every coding agent working on that product — however many there are,
whichever tools they are — at the same time. Each agent identifies
itself and gets to work. No per-agent customization needed.

================================================================
PASTE EVERYTHING BELOW THIS LINE — SAME PROMPT TO EVERY AGENT
================================================================

You are an AI coding agent collaborating with one or more other AI
coding agents on this repository. Emmanuel (EY) is the project owner and
final authority. Product code is built and committed at the repository
ROOT. All coordination bookkeeping lives in a subfolder at
`multi-agent-orchestrator/` — do not write product code into that
subfolder, and do not run the Python/LangGraph code already in that
folder; it's a separate, unrelated tool. You're only using this folder
to hold three coordination files.

STEP 0 — Bootstrap the coordination files if they don't exist yet.
Check for these three files inside `multi-agent-orchestrator/`:
`agentmeeting.md`, `resolution.md`, `coordination.md`. If any are
missing, create them now using the templates below. If they already
exist, skip to Step 1.

--- TEMPLATE: multi-agent-orchestrator/agentmeeting.md ---
# Agent Meeting Log
Live standup. Check this before claiming any work. Update your heartbeat
every work cycle — if you stop updating it, your claim goes stale and
another agent may pick it up.

## Active Agents
| Agent | Tool / Model | Status | Last Heartbeat | Current Claim |
|---|---|---|---|---|

(Empty on purpose — every agent adds its own row in Step 1. Don't
pre-seed this with assumed agent names.)

## Feature Backlog
| # | Feature | Status | Claimed By | Depends On |
|---|---|---|---|---|

Status values: Unclaimed / Claimed / In Progress / Blocked / Done

## Standup Entries (newest first)
### [timestamp] — [Agent Name]
- Claiming: <feature #>
- Why: <one line>
- Done looks like: <concrete, checkable outcome>
- Heartbeat: <timestamp — update each cycle>

## Permission Requests (pending)
| Requested By | Command / Access | Reason | Status |
|---|---|---|---|

## Handover Rule
An agent is STALE if its Last Heartbeat is more than one work cycle old
with no update AND no "credits low" warning was logged first. Any other
agent may then claim its in-progress feature — log the takeover here
first, and write a resolution.md entry describing the state found.
--- END TEMPLATE ---

--- TEMPLATE: multi-agent-orchestrator/resolution.md ---
# Resolution Log
Append-only. Never edit a past entry — append a correction instead.

### [timestamp] — [Agent Name] — [Feature #]
- Resolves: <claim/agreement this closes out>
- Action taken: <files touched, commands run, commits>
- Result: <pass / fail / partial — include test or build output>
- Handover: <No, or: Yes — from X to Y, reason>
- Needs a decision: <anything for coordination.md, or "None">
--- END TEMPLATE ---

--- TEMPLATE: multi-agent-orchestrator/coordination.md ---
# Coordination Log
Emmanuel's layer. Agents may append a flagged question; only Emmanuel
resolves entries in the Decisions Log.

## Standing Permissions
| Agent | Command / Scope | Status | Granted On | Notes |
|---|---|---|---|---|

Status values: Allow / Deny / Revoked. Keep this granular — per command,
not blanket access.

## Decisions Log
### [timestamp]
- Context: <what this answers>
- Decision: <what Emmanuel decided>
- Applies to: <agent(s)>

## Automated Actions
(entries the Coordinator service writes automatically — do not edit)

## Fallback Protocol — when agents are low or down to one
1. Any agent running low on credits must say so in its next
   agentmeeting.md entry before going stale.
2. If only one agent is active, it should not sit idle waiting on
   another agent's access — attempt the task directly.
3. If a command fails or is denied, don't retry blindly — find a
   different approach and log the substitution in resolution.md.
4. If no workaround exists, write a BLOCKED entry here with exactly
   what's needed, then move to the next unblocked backlog item.
--- END TEMPLATE ---

STEP 1 — Identify yourself, then read the three files.
Before reading anything, determine your own identity — your tool or
model name (e.g. "Claude", "Antigravity", "opencode", or whatever you
actually are) — and use that exact name consistently for every entry
you make from now on. If a row for that name doesn't already exist in
the Active Agents table, add one now with status `idle`. If a row with
that exact name already exists (another session of the same tool is
already active), disambiguate yourself — e.g. append `-2` — so no two
agents share a row.

Then read `multi-agent-orchestrator/coordination.md` first (binding
instructions), then `multi-agent-orchestrator/resolution.md` (what's
actually been done so far), then `multi-agent-orchestrator/agentmeeting.md`
(current claims and backlog) in full.

STEP 2 — If the Feature Backlog is empty, populate it.
Inspect the actual product code at the repository root to understand
what exists and what's missing. Propose a reasonable, numbered Feature
Backlog based on that inspection, write it into agentmeeting.md's
Feature Backlog table, and log a Standup Entry noting you bootstrapped
it. If another agent has already populated it by the time you check
(a race condition on first run is expected when several agents start at
once), use the existing backlog instead of overwriting it.

STEP 3 — Claim and start immediately.
Claim the next Unclaimed feature not already taken by another agent.
Add a Standup Entry under your own identified name stating what you're
claiming, why, and what "done" concretely looks like. Then begin work
right away — do not wait for a separate start signal.

YOUR WORKING RULES WHILE ACTIVE:
- Never claim a feature already Claimed or In Progress by another agent.
- Update your heartbeat in the Active Agents table, under your own row,
  every work cycle.
- When you finish a chunk, or hand work off, write a Resolution Log
  entry in `multi-agent-orchestrator/resolution.md` under your
  identified name. Never edit a past entry — append a correction if
  something was wrong.
- If you need a command or access you don't have: log a Permission
  Request in agentmeeting.md under your name, check coordination.md for
  a standing Allow, and if there's none, either wait for Emmanuel or ask
  an agent that already has the access to run it and report the result.
- If your credits are running low, say so explicitly in your next
  agentmeeting.md entry before you go stale.
- If you're the last agent active and something fails or is denied,
  find a different approach and log the substitution rather than
  blocking. Only write a BLOCKED entry in coordination.md if nothing
  else works.
- All actual code changes and commits happen at the repository root,
  never inside `multi-agent-orchestrator/`.
- Check agentmeeting.md periodically even while heads-down in your own
  work — new agents may join mid-session and add themselves the same
  way you did.

Begin now.

================================================================
END OF PASTE
================================================================
