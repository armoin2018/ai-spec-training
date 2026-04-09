# Exercise 3: JSON Plan + Parallel Subagents
### Lab Guide · 10 Minutes

---

## What You're Building

The capstone exercise. You'll put the whole pipeline together:

1. **A `plan.json`** — a structured task plan for a small feature (4 parallel tasks)
2. **A `extract-tasks.sh` skill** — a shell script that uses `jq` to extract individual tasks from the plan
3. **An Orchestrator Prompt** — a system prompt that reads task JSON and spawns focused subagents
4. **A test run** — execute the extraction and observe the parallel spawn pattern

---

## The Architecture

```
plan.json
    │
    ▼
extract-tasks.sh (skill using jq)
    │
    ├── task-001.json  ──▶  Subagent A (backend-engineer)
    ├── task-002.json  ──▶  Subagent B (frontend-engineer)
    ├── task-003.json  ──▶  Subagent C (qa-engineer)
    └── task-004.json  ──▶  Subagent D (docs-writer)
         │
         ▼
    [Results merged by Orchestrator]
```

Each subagent gets a focused context: its own task JSON + the orchestrator's system prompt. No task sees another task's context.

---

## Prerequisites

- `jq` installed (`brew install jq` on macOS, `apt install jq` on Linux)
- A terminal and text editor
- Your orchestrator system prompt from Exercise 1 (optional — we'll provide a starter)

---

## Part A: Create the Plan

### Step 1 — Create `plan.json`

We'll use a concrete scenario: adding a **user notification preferences** feature to an existing web app.

Save this as `plan.json` in your working directory:

```json
{
  "plan_id": "notif-prefs-001",
  "project": "User Notification Preferences",
  "created": "2026-04-07",
  "description": "Allow users to configure which notifications they receive and via which channels (email, SMS, in-app)",
  "phases": [
    {
      "phase": "Build",
      "tasks": [
        {
          "id": "task-001",
          "title": "Create NotificationPreferences data model and migration",
          "description": "Design and implement the database schema for notification preferences. Create a NotificationPreferences table with columns: user_id (FK), notification_type (enum), channels (JSON array of 'email'|'sms'|'in_app'), enabled (boolean), updated_at (timestamp). Write the migration file and rollback. Update the User model to include a preferences relationship.",
          "agent": "backend-engineer",
          "inputs": ["src/models/User.ts", "src/database/migrations/"],
          "outputs": ["src/models/NotificationPreferences.ts", "src/database/migrations/XXXX_create_notification_preferences.ts"],
          "dependencies": [],
          "parallel": true,
          "estimated_complexity": "Low",
          "success_criteria": "Migration runs without errors. NotificationPreferences model includes all specified fields with correct types. User model has hasMany relationship to NotificationPreferences."
        },
        {
          "id": "task-002",
          "title": "Build notification preferences API endpoints",
          "description": "Create REST API endpoints for managing notification preferences. Required endpoints: GET /users/:id/notification-preferences (returns all preferences for user), PUT /users/:id/notification-preferences (bulk update, replaces all preferences), PATCH /users/:id/notification-preferences/:type (update a single preference type). All endpoints require authentication. Return 404 if user not found. Validate that notification_type is a valid enum value.",
          "agent": "backend-engineer",
          "inputs": ["src/routes/", "src/controllers/", "src/models/NotificationPreferences.ts"],
          "outputs": ["src/routes/notificationPreferences.ts", "src/controllers/notificationPreferences.ts"],
          "dependencies": ["task-001"],
          "parallel": false,
          "estimated_complexity": "Medium",
          "success_criteria": "All 3 endpoints return correct responses. Auth middleware is applied. Invalid inputs return 400 with descriptive error messages."
        },
        {
          "id": "task-003",
          "title": "Build notification preferences UI component",
          "description": "Create a React component NotificationPreferencesPanel that renders a list of notification types grouped by category (Marketing, Account Activity, Security). Each item shows a label, description, and toggle switches for each channel (email, sms, in-app). The component should call GET to load current preferences on mount, optimistically update state on toggle, and call PATCH to persist changes. Show a loading skeleton while fetching. Show an error toast if save fails.",
          "agent": "frontend-engineer",
          "inputs": ["src/components/", "src/api/users.ts"],
          "outputs": ["src/components/NotificationPreferencesPanel.tsx", "src/components/NotificationPreferencesPanel.test.tsx"],
          "dependencies": [],
          "parallel": true,
          "estimated_complexity": "Medium",
          "success_criteria": "Component renders correctly in loading, loaded, and error states. Toggle correctly calls PATCH endpoint. Passes accessibility audit (keyboard navigable, screen reader labels)."
        },
        {
          "id": "task-004",
          "title": "Write API integration tests for notification preferences endpoints",
          "description": "Write integration tests (using the project's existing test framework) covering: authenticated user can GET their preferences, unauthenticated request returns 401, valid PUT replaces all preferences, invalid notification_type in PATCH returns 400, PATCH updates single preference correctly. Use the existing test database setup. Do not mock the database — use real queries.",
          "agent": "qa-engineer",
          "inputs": ["src/routes/notificationPreferences.ts", "src/controllers/notificationPreferences.ts", "tests/"],
          "outputs": ["tests/integration/notificationPreferences.test.ts"],
          "dependencies": ["task-002"],
          "parallel": false,
          "estimated_complexity": "Medium",
          "success_criteria": "All 5 test cases pass. Tests cover happy path and error cases. No mocked database calls."
        }
      ]
    }
  ]
}
```

---

## Part B: Build the Task Extraction Skill

This shell script uses `jq` to:
1. Extract all tasks from the plan
2. Write each task to its own JSON file (for parallel handoff to subagents)
3. Identify which tasks can run in parallel vs. which have dependencies

### Step 1 — Create `extract-tasks.sh`

```bash
#!/usr/bin/env bash
# extract-tasks.sh
# Usage: ./extract-tasks.sh plan.json [output-dir]
# Extracts tasks from a plan JSON and writes each to a separate file.
# Prints a summary of parallel vs. sequential tasks.

set -euo pipefail

PLAN_FILE="${1:-plan.json}"
OUTPUT_DIR="${2:-./tasks}"

if [[ ! -f "$PLAN_FILE" ]]; then
  echo "Error: Plan file '$PLAN_FILE' not found." >&2
  exit 1
fi

mkdir -p "$OUTPUT_DIR"

echo "======================================"
echo "  Task Extraction"
echo "  Plan: $PLAN_FILE"
echo "======================================"
echo ""

# Count total tasks
TOTAL=$(jq '[.phases[].tasks[]] | length' "$PLAN_FILE")
echo "Total tasks found: $TOTAL"
echo ""

# Extract each task to its own file
jq -c '.phases[].tasks[]' "$PLAN_FILE" | while IFS= read -r task; do
  TASK_ID=$(echo "$task" | jq -r '.id')
  TITLE=$(echo "$task" | jq -r '.title')
  AGENT=$(echo "$task" | jq -r '.agent')
  PARALLEL=$(echo "$task" | jq -r '.parallel')
  DEPS=$(echo "$task" | jq -r '.dependencies | length')

  # Write individual task file
  echo "$task" > "$OUTPUT_DIR/$TASK_ID.json"

  # Print summary
  if [[ "$PARALLEL" == "true" ]]; then
    STATUS="⚡ PARALLEL (ready now)"
  else
    DEP_LIST=$(echo "$task" | jq -r '.dependencies | join(", ")')
    STATUS="⏳ SEQUENTIAL (waits for: $DEP_LIST)"
  fi

  echo "  [$TASK_ID] $TITLE"
  echo "    Agent: $AGENT | $STATUS"
  echo ""
done

echo "======================================"
echo "  Parallel tasks (spawn immediately):"
jq -r '.phases[].tasks[] | select(.parallel == true) | "  - " + .id + ": " + .title' "$PLAN_FILE"

echo ""
echo "  Sequential tasks (wait for dependencies):"
jq -r '.phases[].tasks[] | select(.parallel == false) | "  - " + .id + " (needs: " + (.dependencies | join(", ")) + ")"' "$PLAN_FILE"

echo ""
echo "Task files written to: $OUTPUT_DIR/"
echo "======================================"
```

### Step 2 — Make it executable and run it

```bash
chmod +x extract-tasks.sh
./extract-tasks.sh plan.json ./tasks
```

**Expected output:**
```
======================================
  Task Extraction
  Plan: plan.json
======================================

Total tasks found: 4

  [task-001] Create NotificationPreferences data model and migration
    Agent: backend-engineer | ⚡ PARALLEL (ready now)

  [task-002] Build notification preferences API endpoints
    Agent: backend-engineer | ⏳ SEQUENTIAL (waits for: task-001)

  [task-003] Build notification preferences UI component
    Agent: frontend-engineer | ⚡ PARALLEL (ready now)

  [task-004] Write API integration tests
    Agent: qa-engineer | ⏳ SEQUENTIAL (waits for: task-002)

======================================
  Parallel tasks (spawn immediately):
  - task-001: Create NotificationPreferences data model and migration
  - task-003: Build notification preferences UI component

  Sequential tasks (wait for dependencies):
  - task-002 (needs: task-001)
  - task-004 (needs: task-002)
======================================
Task files written to: ./tasks/
```

---

## Part C: Build the Subagent Orchestrator Prompt

This orchestrator reads a task JSON and configures itself as the appropriate specialist to execute it.

### Create `subagent-orchestrator.md`

```markdown
# Subagent Orchestrator System Prompt

## Role
You are an expert agent executor. You will be given a single task JSON object.
Your job is to execute that task completely and correctly — nothing more, nothing less.

## On Startup
1. Read the task JSON carefully
2. Adopt the persona of the specified `agent` role
3. Review all files listed in `inputs` before starting
4. Execute the task as described in `description`
5. Produce all artifacts listed in `outputs`
6. Verify your work against `success_criteria`
7. Report completion with a summary of what was done

## Agent Personas

**backend-engineer:**
You are a senior backend engineer. You write clean, well-structured server-side code.
You follow existing patterns in the codebase. You never skip error handling.
You add JSDoc comments to all exported functions.

**frontend-engineer:**
You are a senior frontend engineer specializing in React and TypeScript.
You write accessible, well-typed components. You handle loading, error, and empty states.
You co-locate tests with components.

**qa-engineer:**
You are a senior QA engineer who writes integration tests.
You never mock the database. You test happy paths and failure modes.
You write clear test descriptions that serve as documentation.

**docs-writer:**
You are a technical writer who creates clear, accurate developer documentation.
You explain the why, not just the what. You include examples for every concept.

## Constraints
- Only create or modify files listed in the task's `outputs`
- Do not modify files not listed in `inputs` unless absolutely required (flag it if so)
- If you encounter a blocker, document it and stop — do not guess past it
- When done, output: "TASK COMPLETE: [task-id] — [one sentence summary of what was built]"

## Token Budget
You have a focused context for this single task. Do not load files outside the specified inputs.
Your goal is surgical, accurate execution — not exploration.
```

---

## Part D: Run a Parallel Spawn (Conceptual Walkthrough)

In a real agentic system (Claude Code, AutoGen, CrewAI, etc.), you would spawn parallel subagents programmatically. Here's what that looks like conceptually and in Claude Code:

### Conceptual Pattern

```bash
# Step 1: Extract parallel tasks
PARALLEL_TASKS=$(jq -r '.phases[].tasks[] | select(.parallel == true) | .id' plan.json)

# Step 2: Spawn each as a parallel subagent
for TASK_ID in $PARALLEL_TASKS; do
  TASK_JSON=$(cat "./tasks/$TASK_ID.json")

  # In Claude Code: each agent invocation is an independent session
  # The subagent-orchestrator.md becomes the system prompt
  # The task JSON becomes the user message
  echo "Spawning subagent for: $TASK_ID"
  echo "Agent: $(cat ./tasks/$TASK_ID.json | jq -r '.agent')"
  echo "Task: $(cat ./tasks/$TASK_ID.json | jq -r '.title')"
  echo "---"
done
```

### In Claude Code (Agent SDK Pattern)

When using the Claude Agent SDK or Claude Code's subagent feature, the pattern is:

```javascript
// Pseudocode — illustrates the pattern
const parallelTasks = plan.phases
  .flatMap(p => p.tasks)
  .filter(t => t.parallel === true);

const results = await Promise.all(
  parallelTasks.map(task =>
    agent.spawn({
      systemPrompt: subagentOrchestratorPrompt,
      userMessage: JSON.stringify(task),
      tools: ['read_file', 'write_file', 'run_bash']
    })
  )
);
```

The key: `Promise.all` spawns all parallel tasks simultaneously. Each runs in its own context. They don't share memory or tokens. You get the results back when all complete.

---

## Putting It All Together

Here's the complete flow you've now built:

```
1. /plan command (Exercise 2)
         ↓
   plan.json
         ↓
2. extract-tasks.sh (this exercise)
         ↓
   tasks/task-001.json   tasks/task-002.json   tasks/task-003.json
         ↓                      ↓                      ↓
3. subagent-orchestrator.md applied to each task
         ↓
   Subagents execute in parallel (task-001 and task-003)
         ↓
   Results returned to orchestrator
         ↓
   Sequential tasks (task-002, task-004) triggered when dependencies complete
```

---

## Bonus Challenges

**Bonus 1: Add a dependency checker**
Extend `extract-tasks.sh` to verify that all dependency task IDs actually exist in the plan before writing the output files. Print a warning if a dependency references a non-existent task.

**Bonus 2: Generate a Mermaid diagram from the plan**
Write a `jq` command (or small script) that generates a Mermaid flowchart from your `plan.json` showing task dependencies:
```
graph TD
  task-001 --> task-002
  task-002 --> task-004
  task-003
```

**Bonus 3: Progress tracking**
Add a `status` field to each task in `plan.json` (`pending | in_progress | complete | failed`). Write a `progress.sh` script that reads all task files and prints a status board.

---

## Key Takeaways

- **Plans as data** make agent orchestration systematic and repeatable
- **jq** is a power tool for extracting structured data from JSON — learn it
- **Independence flags in your plan** enable parallel execution by design
- **Focused subagent context** improves quality and reduces token cost vs. one giant agent
- **This pattern scales:** 4 tasks today, 40 tasks in production — the orchestrator doesn't change

---

*Exercise 3 complete.*
*Full artifact set: `plan.json`, `extract-tasks.sh`, `tasks/`, `subagent-orchestrator.md`*
*Congratulations — you've built a complete Spec → Plan → Parallel Execute pipeline.*
