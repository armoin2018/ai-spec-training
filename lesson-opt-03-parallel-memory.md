# Optional Lesson 3: Parallel Memory Management for Multi-Subagent Systems
### Lab Guide · 20 Minutes

---

## The Core Idea

Running multiple agents in parallel is powerful. But the moment two agents try to update the same memory store, you've created a distributed systems problem. If you've built software long enough, you know what happens next: race conditions, lost writes, corrupt state, and hours of debugging.

This lesson teaches a **zone-based memory architecture** that eliminates write conflicts by design, enables efficient cross-agent knowledge sharing, and produces a clean handoff package at the end of every session.

---

## The Distributed Memory Problem, Made Concrete

Imagine 3 subagents running in parallel on a feature build:

- **Agent A** (backend) finishes first and writes: *"Used JSON column for notification channels"*
- **Agent B** (frontend) reads memory to understand the data shape, but Agent A's write hasn't been flushed yet
- **Agent C** (QA) needs to know the data model to write tests, reads stale memory, tests the wrong schema
- **Agent B** also discovers a schema issue and writes: *"Channels should be a junction table for flexibility"*

Now you have conflicting decisions in memory and a QA test suite that tests neither of them correctly.

**Root cause:** Agents can read and write to the same location concurrently. Solution: eliminate concurrent writes to shared locations.

---

## Zone-Based Memory Architecture

The fundamental rule: **during parallel execution, each agent writes only to its own zone.**

```
memory/
│
├── shared/                         # READ-ONLY for subagents
│   ├── project-context.md          # Stack, conventions, standards
│   ├── architecture.md             # System diagram, component boundaries
│   ├── open-decisions.md           # Pending decisions agents should know about
│   └── constraints.md              # Hard rules (security, compliance, style)
│
├── agents/                         # WRITE zone — one subdirectory per agent
│   ├── task-001-backend/
│   │   ├── progress.md             # Running log of what was done
│   │   ├── findings.md             # Discoveries, edge cases, observations
│   │   ├── decisions.md            # Decisions this agent made
│   │   └── blockers.md             # What is blocking progress (if anything)
│   ├── task-002-frontend/
│   │   ├── progress.md
│   │   ├── findings.md
│   │   ├── decisions.md
│   │   └── blockers.md
│   └── task-003-qa/
│       ├── progress.md
│       ├── findings.md
│       ├── decisions.md
│       └── blockers.md
│
└── merged/                         # WRITE by orchestrator only, after all agents done
    ├── session-summary.md          # What was accomplished this session
    ├── decisions.md                # All decisions, consolidated and deduplicated
    ├── findings.md                 # All findings, merged
    ├── blockers.md                 # Open blockers for next session
    └── handoff.md                  # Bootstrap document for the next session
```

**Access rules:**
| Layer | Subagents | Orchestrator |
|---|---|---|
| `shared/` | Read only | Read + Write (setup) |
| `agents/<task-id>/` | Write own zone only | Read all |
| `merged/` | No access | Read + Write (after merge) |

---

## Part A: Memory Initialization Skills

### `init-memory.sh` — Orchestrator runs this before spawning subagents

```bash
#!/usr/bin/env bash
# init-memory.sh
# Usage: ./init-memory.sh <plan.json>
# Creates the memory directory structure for a parallel run.
# Writes shared context, creates agent zones.

set -euo pipefail

PLAN_FILE="${1:-plan.json}"
MEMORY_ROOT="${2:-./memory}"
DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)

if [[ ! -f "$PLAN_FILE" ]]; then
  echo "Error: Plan file '$PLAN_FILE' not found." >&2
  exit 1
fi

PROJECT=$(jq -r '.project' "$PLAN_FILE")
PLAN_ID=$(jq -r '.plan_id' "$PLAN_FILE")

echo "Initializing memory for: $PROJECT ($PLAN_ID)"
echo ""

# Create directory structure
mkdir -p "$MEMORY_ROOT/shared"
mkdir -p "$MEMORY_ROOT/merged"

# Write shared context header
cat > "$MEMORY_ROOT/shared/session-info.md" << EOF
# Session: $PROJECT
- Plan ID: $PLAN_ID
- Started: $DATE
- Memory root: $MEMORY_ROOT

## This Session's Tasks
$(jq -r '.phases[].tasks[] | "- [\(.id)] \(.title) → Agent: \(.agent)"' "$PLAN_FILE")
EOF

echo "  ✓ Shared context initialized"

# Create agent memory zones
jq -r '.phases[].tasks[].id' "$PLAN_FILE" | while read -r task_id; do
  AGENT=$(jq -r --arg id "$task_id" '.phases[].tasks[] | select(.id == $id) | .agent' "$PLAN_FILE")
  TITLE=$(jq -r --arg id "$task_id" '.phases[].tasks[] | select(.id == $id) | .title' "$PLAN_FILE")
  ZONE="$MEMORY_ROOT/agents/$task_id"

  mkdir -p "$ZONE"

  # Initialize each memory file with header
  cat > "$ZONE/progress.md" << EOF
# Progress: [$task_id] $title
Agent: $AGENT
Started: $DATE

## Work Log
EOF

  cat > "$ZONE/findings.md" << EOF
# Findings: [$task_id] $title
Agent: $AGENT

## Discoveries and Observations
EOF

  cat > "$ZONE/decisions.md" << EOF
# Decisions: [$task_id] $title
Agent: $AGENT

## Decisions Made During This Task
<!-- Format: ## Decision: [short title]
**Chose:** X
**Over:** Y
**Because:** reason
**Implications:** what this means for other agents -->
EOF

  cat > "$ZONE/blockers.md" << EOF
# Blockers: [$task_id] $title
Agent: $AGENT

## Current Blockers
<!-- Leave empty if none -->
EOF

  echo "  ✓ Agent zone created: $task_id ($AGENT)"
done

echo ""
echo "Memory initialized at: $MEMORY_ROOT/"
echo "Subagents may now be spawned."
```

---

### `subagent-memory-header.sh` — Prepend to each subagent's context

```bash
#!/usr/bin/env bash
# subagent-memory-header.sh
# Usage: ./subagent-memory-header.sh <task-id> <memory-root>
# Outputs a memory context block to prepend to a subagent's system prompt.

TASK_ID="$1"
MEMORY_ROOT="${2:-./memory}"

ZONE="$MEMORY_ROOT/agents/$TASK_ID"

cat << EOF
## Your Memory Zone

You have a private memory zone at: $ZONE/

Write to these files during your work:
- \`$ZONE/progress.md\` — update as you complete steps
- \`$ZONE/findings.md\` — record anything interesting you discover
- \`$ZONE/decisions.md\` — document every significant decision you make
- \`$ZONE/blockers.md\` — write blockers here immediately if you get stuck

READ from shared context (DO NOT write to these):
- \`$MEMORY_ROOT/shared/project-context.md\` — project stack and conventions
- \`$MEMORY_ROOT/shared/architecture.md\` — system design reference
- \`$MEMORY_ROOT/shared/constraints.md\` — hard rules you must follow

DO NOT write to any other agent's zone.
DO NOT write to \`$MEMORY_ROOT/merged/\`.
DO NOT read from other agents' zones — work from shared context only.

## Memory Update Protocol
After each significant step, append a timestamped entry to progress.md:
\`\`\`
### [HH:MM] Step completed
What was done and the outcome.
\`\`\`
EOF
```

---

## Part B: The Memory Merge Skill

### `merge-memory.sh` — Orchestrator runs this after all subagents complete

```bash
#!/usr/bin/env bash
# merge-memory.sh
# Usage: ./merge-memory.sh <memory-root>
# Consolidates all agent memory zones into merged/ directory.
# Deduplicates decisions, collects blockers, writes handoff document.

set -euo pipefail

MEMORY_ROOT="${1:-./memory}"
MERGED="$MEMORY_ROOT/merged"
DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
SESSION_DATE=$(date -u +%Y-%m-%d)

mkdir -p "$MERGED"

echo "======================================"
echo "  Memory Merge — $DATE"
echo "======================================"
echo ""

# ── 1. Merge all findings ──────────────────────────────────
echo "# All Agent Findings — $SESSION_DATE" > "$MERGED/findings.md"
echo "" >> "$MERGED/findings.md"

for AGENT_DIR in "$MEMORY_ROOT/agents"/*/; do
  TASK_ID=$(basename "$AGENT_DIR")
  FINDINGS="$AGENT_DIR/findings.md"

  if [[ -f "$FINDINGS" ]]; then
    CONTENT_LINES=$(grep -v "^#" "$FINDINGS" | grep -v "^$" | wc -l)
    if [[ "$CONTENT_LINES" -gt 0 ]]; then
      echo "## $TASK_ID" >> "$MERGED/findings.md"
      grep -v "^# " "$FINDINGS" | grep -v "^## Discoveries" >> "$MERGED/findings.md"
      echo "" >> "$MERGED/findings.md"
      echo "  ✓ Merged findings from $TASK_ID"
    else
      echo "  - No findings from $TASK_ID"
    fi
  fi
done

# ── 2. Merge all decisions ─────────────────────────────────
echo "# All Agent Decisions — $SESSION_DATE" > "$MERGED/decisions.md"
echo "" >> "$MERGED/decisions.md"
echo "> Review for conflicts before starting the next session." >> "$MERGED/decisions.md"
echo "" >> "$MERGED/decisions.md"

for AGENT_DIR in "$MEMORY_ROOT/agents"/*/; do
  TASK_ID=$(basename "$AGENT_DIR")
  DECISIONS="$AGENT_DIR/decisions.md"

  if [[ -f "$DECISIONS" ]]; then
    CONTENT_LINES=$(grep -v "^#" "$DECISIONS" | grep -v "^$" | grep -v "^<!--" | wc -l)
    if [[ "$CONTENT_LINES" -gt 0 ]]; then
      echo "## From $TASK_ID" >> "$MERGED/decisions.md"
      grep -v "^# " "$DECISIONS" | grep -v "^## Decisions Made" | grep -v "^<!-- Format" >> "$MERGED/decisions.md"
      echo "" >> "$MERGED/decisions.md"
      echo "  ✓ Merged decisions from $TASK_ID"
    fi
  fi
done

# ── 3. Collect blockers ────────────────────────────────────
echo "# Open Blockers — $SESSION_DATE" > "$MERGED/blockers.md"
echo "" >> "$MERGED/blockers.md"

BLOCKER_COUNT=0
for AGENT_DIR in "$MEMORY_ROOT/agents"/*/; do
  TASK_ID=$(basename "$AGENT_DIR")
  BLOCKERS="$AGENT_DIR/blockers.md"

  if [[ -f "$BLOCKERS" ]]; then
    CONTENT_LINES=$(grep -v "^#" "$BLOCKERS" | grep -v "^$" | grep -v "^<!-- Leave" | wc -l)
    if [[ "$CONTENT_LINES" -gt 0 ]]; then
      echo "## $TASK_ID" >> "$MERGED/blockers.md"
      grep -v "^# " "$BLOCKERS" | grep -v "^## Current Blockers" | grep -v "^<!-- Leave" >> "$MERGED/blockers.md"
      echo "" >> "$MERGED/blockers.md"
      BLOCKER_COUNT=$((BLOCKER_COUNT + 1))
      echo "  ⚠️  Blockers found in $TASK_ID"
    fi
  fi
done

if [[ "$BLOCKER_COUNT" -eq 0 ]]; then
  echo "No blockers reported." >> "$MERGED/blockers.md"
  echo "  ✓ No blockers"
fi

# ── 4. Build progress summary ──────────────────────────────
echo "# Session Progress Summary — $SESSION_DATE" > "$MERGED/session-summary.md"
echo "" >> "$MERGED/session-summary.md"
echo "## Tasks This Session" >> "$MERGED/session-summary.md"

for AGENT_DIR in "$MEMORY_ROOT/agents"/*/; do
  TASK_ID=$(basename "$AGENT_DIR")
  PROGRESS="$AGENT_DIR/progress.md"

  if [[ -f "$PROGRESS" ]]; then
    # Get last progress entry (most recent step)
    LAST_ENTRY=$(grep "### " "$PROGRESS" | tail -1)
    STATUS="✅ Complete"
    [[ -f "$AGENT_DIR/blockers.md" ]] && grep -q "^[^#<]" "$AGENT_DIR/blockers.md" && STATUS="⚠️ Blocked"
    echo "- $TASK_ID: $STATUS — $LAST_ENTRY" >> "$MERGED/session-summary.md"
  fi
done

echo "" >> "$MERGED/session-summary.md"
echo "## Merge completed: $DATE" >> "$MERGED/session-summary.md"

echo ""
echo "======================================"
echo "  Merge complete. Files written:"
echo "    $MERGED/findings.md"
echo "    $MERGED/decisions.md"
echo "    $MERGED/blockers.md"
echo "    $MERGED/session-summary.md"
[[ "$BLOCKER_COUNT" -gt 0 ]] && echo "" && echo "  ⚠️  $BLOCKER_COUNT task(s) have blockers — review before next session"
echo "======================================"
```

---

## Part C: The Handoff Document

At session end, the orchestrator writes a handoff that bootstraps the next session:

### Handoff template (written by orchestrator after merge)

```markdown
# Session Handoff
Generated: {{DATE}}
Project: {{PROJECT}}
Plan ID: {{PLAN_ID}}

## ✅ Completed This Session
{{List from session-summary.md — completed tasks}}

## ⚠️ In Progress / Incomplete
{{List of tasks that are partial or blocked}}

## Key Decisions (Read Before Next Session)
{{Summary of decisions from merged/decisions.md that affect future work}}

## Critical Findings
{{High-signal items from merged/findings.md that future agents need to know}}

## Open Blockers
{{Contents of merged/blockers.md}}

## Recommended Next Steps (In Order)
1. {{Most urgent next action}}
2. {{Second action}}
3. {{Third action}}

## Files Modified This Session
{{List of files that were created or changed}}

## What the Next Session Should NOT Do
{{Anti-recommendations — e.g., "Don't re-investigate the channels column decision — it's final"}}
```

---

## Part D: The Exercise

### Step 1 — Initialize a memory structure

Using the plan.json from Exercise 3 (or create a new one), initialize the memory:

```bash
chmod +x init-memory.sh
./init-memory.sh plan.json ./memory
```

Verify the structure was created:
```bash
find ./memory -type f | sort
```

### Step 2 — Simulate two agents writing

Open two terminal windows. In each, simulate an agent writing to its zone:

**Terminal 1 (simulating task-001):**
```bash
cat >> ./memory/agents/task-001/progress.md << EOF

### [10:15] Schema designed
Created NotificationPreferences table. Used JSON column for channels.
EOF

cat >> ./memory/agents/task-001/decisions.md << EOF

## Decision: Channel Storage Format
**Chose:** JSON column in NotificationPreferences table
**Over:** Separate junction table (notification_channels)
**Because:** Current channel options are stable (3 values). JSON adds flexibility with minimal cost.
**Implications:** Frontend team: channels field returns an array of strings, not a join.
EOF
```

**Terminal 2 (simulating task-003):**
```bash
cat >> ./memory/agents/task-003/progress.md << EOF

### [10:18] Component skeleton built
NotificationPreferencesPanel renders toggle group for each notification type.
Needs channel data shape from backend to complete API integration.
EOF

cat >> ./memory/agents/task-003/findings.md << EOF

## Finding: Channel Data Shape Unclear
The component needs to know if channels come back as an array ["email", "sms"]
or as an object { email: true, sms: false }. Blocked on this.
EOF

cat >> ./memory/agents/task-003/blockers.md << EOF

## Blocker: Channel Data Shape
Need backend to confirm whether channels field is:
- Array: ["email", "sms"] (current preference)
- Object: { email: true, sms: false }
This affects how the toggle component maps state.
EOF
```

### Step 3 — Run the merge

```bash
chmod +x merge-memory.sh
./merge-memory.sh ./memory
```

### Step 4 — Observe the result

Read `./memory/merged/decisions.md`. Notice that Agent A's decision about using a JSON column is now visible — exactly the information Agent B needed to resolve their blocker.

Read `./memory/merged/blockers.md`. A human or orchestrator can now resolve this blocker and write the answer to `shared/constraints.md` before the next parallel run.

This is the value: **agents don't need to communicate directly**. The memory merge creates a structured information exchange that happens safely, after parallel execution completes.

---

## Preventing Decision Conflicts

What if two agents make conflicting decisions? Add a conflict detection step to the merge:

```bash
# detect-conflicts.sh — scan merged decisions for keywords that suggest contradiction
echo "Scanning for potential conflicts..."

DECISIONS_FILE="./memory/merged/decisions.md"

# Look for agents making decisions about the same topic
jq -n --rawfile content "$DECISIONS_FILE" '
  $content | split("\n") |
  map(select(startswith("**Chose:**"))) |
  group_by(.) |
  map(select(length > 1)) |
  .[]
'
# (This is a simplification — a real implementation would use semantic similarity)

# Simpler: flag any decision that mentions the same entity
grep -i "channels\|schema\|model\|service" "$DECISIONS_FILE" | \
  grep "^##\|^Chose\|^Over" | \
  head -20
```

---

## Takeaways

- **Zone isolation eliminates write conflicts** — by design, not by hoping
- **Shared context is read-only** — prepare it before spawning, don't mutate during
- **The merge step is the communication step** — agents exchange knowledge through structured handoff, not direct messaging
- **Blockers surface immediately** — don't wait for agents to fail silently
- **Handoff documents survive context resets** — the next session starts informed, not blank

---

*Optional Lesson 3 complete.*
*Artifacts to keep: `init-memory.sh`, `merge-memory.sh`, `subagent-memory-header.sh`, memory directory template*
