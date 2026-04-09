# Exercise 2: Slash Commands for Each Spec Phase
### Lab Guide · 10 Minutes

---

## What You're Building

A personal **Spec Command Kit** — a set of reusable slash commands (prompt templates) that guide an AI agent through each phase of spec building.

By the end you'll have working versions of:

| Command | Purpose |
|---|---|
| `/brd` | Generate a Business Requirements Document |
| `/trd` | Generate a Technical Requirements Document |
| `/arch` | Draft an Architecture Overview |
| `/plan` | Generate a structured JSON task plan |
| `/review` | Run a spec compliance review |

These work in Claude Code, Copilot Chat, Cursor Composer, or any AI tool that accepts slash commands or custom prompts.

---

## Background: What Makes a Good Slash Command?

A slash command is a reusable prompt template. A good one:
- Gives the agent an **expert persona** for the task
- Provides a **structured output template** to fill in
- Defines explicit **inputs** (what the user provides) vs **outputs** (what the agent generates)
- Includes **quality criteria** — what does "done" look like?
- Is **self-contained** — no prior context should be required

---

## The Commands

### `/brd` — Business Requirements Document

```markdown
---
name: brd
description: Generate a Business Requirements Document from a feature description
---

You are a senior product manager and business analyst.

Given the feature or product idea below, generate a structured Business Requirements Document (BRD).

**Input:**
{{USER_INPUT}}

**Output the following sections:**

## Problem Statement
What specific problem is being solved? Who experiences this problem? What is the cost of not solving it?

## Target Users
Primary user personas. Include role, context, and key pain points.

## Goals & Success Metrics
3-5 measurable goals. For each, define a specific metric and how it will be measured.

## Scope
### In Scope
List what will be built in this iteration.
### Out of Scope
List what is explicitly excluded (to prevent scope creep).

## Constraints
Business, regulatory, timeline, or resource constraints.

## Assumptions
What are we assuming to be true that has not been confirmed?

## Open Questions
What decisions still need to be made before development starts?

---
Quality bar: A stakeholder who has never heard of this project should fully understand what is being built and why after reading this document.
```

---

### `/trd` — Technical Requirements Document

```markdown
---
name: trd
description: Generate a Technical Requirements Document from a BRD or feature description
---

You are a principal software engineer with expertise in system design and technical specification.

Given the requirements below, generate a structured Technical Requirements Document (TRD).

**Input:**
{{USER_INPUT}}

**Output the following sections:**

## Technical Summary
One paragraph: what is being built from an engineering perspective.

## Stack & Technology Decisions
List each major technology choice with a one-line justification.
Flag any choices that need team discussion.

## System Components
List the major components to be built (services, modules, APIs, data stores).
For each: name, responsibility, interface (inputs/outputs).

## API Contracts
For each API endpoint:
- Method + path
- Request payload schema
- Response payload schema
- Error states

## Data Models
Key entities, their fields, and relationships.
Highlight any schema migration concerns.

## Non-Functional Requirements
- Performance targets (e.g., p95 response time < 200ms)
- Security requirements (e.g., authentication, authorization, data encryption)
- Scalability considerations
- Availability / uptime targets

## Integration Points
External systems, third-party services, or internal services this interacts with.
Note dependencies and failure modes.

## Technical Risks
What technical unknowns or risks could delay delivery?
For each: severity (High/Medium/Low) and proposed mitigation.

---
Quality bar: A developer who has never seen this project should be able to implement each component from this document without needing additional specification.
```

---

### `/arch` — Architecture Overview

```markdown
---
name: arch
description: Generate a system architecture overview and diagram description
---

You are a solutions architect with expertise in distributed systems and developer tooling.

Given the technical requirements below, produce an architecture overview.

**Input:**
{{USER_INPUT}}

**Output the following sections:**

## Architecture Style
What architectural pattern does this system follow? (e.g., layered, event-driven, microservices, monolith)
Why is this the right choice for this context?

## System Diagram (Text Description)
Describe the system as a diagram in plain text, in this format:
- List each component in brackets: [Component Name]
- Show data flow with arrows: [A] → [B]
- Group related components in sections

## Component Responsibilities
For each major component:
- **Name:**
- **Responsibility:** (one sentence)
- **Technology:**
- **Communicates with:**

## Data Flow
Walk through the primary user journey step-by-step, naming which component handles each step and what data moves between them.

## Key Design Decisions
3-5 architectural decisions that shaped this design. For each:
- Decision: what was chosen
- Alternatives considered
- Rationale

## Deployment Topology
Where does each component run? How is it deployed?

## Scaling Approach
What scales horizontally? What has scaling constraints? What are the bottlenecks?
```

---

### `/plan` — JSON Task Plan

```markdown
---
name: plan
description: Generate a structured JSON task plan ready for parallel agent execution
---

You are a technical project manager and systems architect.

Given the requirements, architecture, or feature description below, generate a structured task plan in JSON format. The plan will be executed by AI agents, so tasks must be:
- Independently executable (no hidden dependencies)
- Atomic (one clear deliverable per task)
- Explicitly assigned to an agent persona
- Flagged for parallel execution where possible

**Input:**
{{USER_INPUT}}

**Output a JSON object in exactly this schema:**

```json
{
  "plan_id": "unique-id",
  "project": "Project or feature name",
  "created": "ISO date",
  "phases": [
    {
      "phase": "Phase name (e.g., Scaffold, Build, Test)",
      "tasks": [
        {
          "id": "task-001",
          "title": "Short task name",
          "description": "Detailed description of what must be built or done",
          "agent": "Agent persona (e.g., backend-engineer, frontend-engineer, qa-engineer)",
          "inputs": ["list of files or data this task needs"],
          "outputs": ["list of files or artifacts this task produces"],
          "dependencies": ["task-ids this task depends on, or empty array if independent"],
          "parallel": true,
          "estimated_complexity": "Low | Medium | High",
          "success_criteria": "How to verify this task is complete"
        }
      ]
    }
  ]
}
```

Rules:
- Tasks with empty `dependencies` arrays CAN be run in parallel
- Each task must have at least one output artifact
- `success_criteria` must be verifiable (testable or reviewable)
- Do not combine unrelated work into a single task
- Maximum 5 tasks per phase — break into more phases if needed
```

---

### `/review` — Spec Compliance Review

```markdown
---
name: review
description: Review implementation against original spec for compliance and quality
---

You are a senior engineering lead conducting a spec compliance review.

Given the original specification and the current implementation (or description of the implementation), assess compliance and quality.

**Input:**
Original Spec: {{SPEC}}
Implementation: {{IMPLEMENTATION}}

**Output the following sections:**

## Compliance Summary
Overall compliance rating: [Full / Partial / Non-Compliant]
Brief explanation (2-3 sentences).

## Requirements Coverage
For each requirement in the spec, mark: ✅ Met | ⚠️ Partially Met | ❌ Not Met
Include a one-line note for anything that is partial or missing.

## Quality Assessment
- **Code quality:** (e.g., naming, structure, readability)
- **Test coverage:** (what's tested, what's missing)
- **Error handling:** (complete, partial, missing)
- **Documentation:** (present, adequate, missing)

## Issues Found
List each issue with:
- Severity: Critical | Major | Minor
- Description: What's wrong
- Recommendation: How to fix it

## What's Working Well
2-3 things done correctly that should be preserved.

## Recommended Next Steps
Ordered list of actions before this can be considered complete.
```

---

## How to Install These Commands

### Claude Code

1. Create a directory: `mkdir -p .claude/commands`
2. Save each command as a `.md` file: `brd.md`, `trd.md`, `arch.md`, `plan.md`, `review.md`
3. In a Claude Code session, run them with `/brd`, `/trd`, etc.

### GitHub Copilot

Paste the prompt body into Copilot Chat as your message. Or save as a `.prompt.md` file in `.github/prompts/` for reuse via the Copilot prompt picker.

### Cursor / Windsurf

Use these as Composer prompts. You can save frequently used prompts as custom commands in Cursor's settings.

### Any AI Tool

Copy the prompt body and paste at the start of your session. Replace `{{USER_INPUT}}` with your actual input.

---

## Practice Run

Use this scenario to test your commands:

**Scenario:** *A SaaS company wants to add a feature that lets users export their account data in CSV format (GDPR compliance).*

1. Run `/brd` with this scenario
2. Take the BRD output and run `/trd`
3. Take the TRD output and run `/plan`
4. Review what the `/plan` output looks like — is it ready for Exercise 3?

---

## Calibration Tips

**If the output is too generic:** Add domain context to your system prompt before running the command. "We are building a B2B SaaS product using Node.js, PostgreSQL, and React."

**If the output is too long:** Add a constraint: "Keep the BRD to one page. Prioritize clarity over completeness."

**If the output misses your specific patterns:** Add a line to the command: "Follow the repository pattern established in our existing code."

---

*Exercise 2 complete. Artifacts to keep: your 5 slash command files*
*Next: Exercise 3 — we'll use the `/plan` output to spawn parallel subagents*
