# Optional Lesson 1: Object-Oriented Structure for Instructions
### Lab Guide · 15 Minutes

---

## The Core Idea

You already know that copy-pasting code is bad. You use inheritance, composition, and interfaces to build maintainable systems. Your agent instructions deserve the same treatment.

This lesson teaches you to build a **layered instruction architecture** where:
- Universal rules live once, in a base layer
- Role-specific behaviors extend the base without duplicating it
- Domain constraints are mixed in only when relevant
- A skill resolves the full instruction set at runtime

---

## The Flat File Problem (Before)

Most teams end up here:

```
# backend-agent-instructions.md  (347 lines)

Never log secrets.
Add JSDoc to all exports.
Follow the repository pattern.
Use typed errors.
Never skip error handling.
[...250 more lines mixing universal rules with backend-specific rules...]
```

```
# frontend-agent-instructions.md  (298 lines)

Never log secrets.       ← duplicate
Add JSDoc to all exports. ← duplicate
Follow the component pattern.
Use typed props.
Never skip error handling. ← duplicate
[...200 more lines...]
```

When you need to update "never log secrets" — you find it in 6 files, update 4, miss 2. An agent uses a stale rule. A production incident follows.

---

## The OO Architecture (After)

```
instructions/
├── base/
│   ├── universal.md          # Rules ALL agents follow, always
│   ├── code-quality.md       # Style, naming, documentation standards
│   └── security.md           # Security rules, secret handling, input validation
│
├── roles/
│   ├── backend-engineer.md   # Server-side patterns, DB, APIs
│   ├── frontend-engineer.md  # Component patterns, state, accessibility
│   ├── qa-engineer.md        # Testing philosophy, coverage, no mocks
│   └── tech-lead.md          # Review standards, escalation, architecture
│
└── domains/
    ├── payments.md            # PCI concerns, Stripe patterns, audit trails
    ├── auth.md                # OAuth, session handling, JWT
    ├── data-privacy.md        # GDPR, PII handling, retention
    └── api-design.md          # REST conventions, versioning, pagination
```

An agent's configuration declares what to include:

```json
{
  "agent": "payments-api-engineer",
  "includes": [
    "instructions/base/universal.md",
    "instructions/base/code-quality.md",
    "instructions/base/security.md",
    "instructions/roles/backend-engineer.md",
    "instructions/domains/payments.md",
    "instructions/domains/auth.md"
  ],
  "overrides": [
    "Never add console.log statements in payment-processing code. Use the AuditService instead.",
    "All payment-related functions must have a corresponding entry in the audit log."
  ]
}
```

---

## Part A: Write the Base Layer

### `instructions/base/universal.md`

```markdown
# Universal Agent Rules

These rules apply to every agent in this system without exception.

## Communication
- Before modifying any existing code, read it first.
- If a task is ambiguous, state your assumption explicitly before proceeding.
- End each response with either "Next: [what you'll do]" or "Done: [what was completed]".

## Safety
- Never delete code. Comment it out and add a note explaining why.
- Never modify files outside the task's specified scope without flagging it.
- If you encounter an error you cannot diagnose, stop and report it — do not guess.

## Context Discipline
- Summarize completed work before starting a new task (keeps context clean).
- Do not load more than 3 files into context at once unless explicitly required.
- If you've already been given information, do not ask for it again.
```

### `instructions/base/security.md`

```markdown
# Security Rules

## Secrets and Credentials
- Never log, print, or include secrets, API keys, tokens, or passwords in output.
- Never hardcode credentials — use environment variables.
- If you find hardcoded credentials in existing code, flag them as a critical issue immediately.

## Input Validation
- All user-supplied input must be validated before use.
- SQL queries must use parameterized queries or an ORM — never string concatenation.
- File paths from user input must be sanitized to prevent path traversal.

## Dependencies
- Do not add new external dependencies without checking if the project already has a library that serves the same purpose.
- Flag any dependency that has known CVEs or hasn't been updated in >2 years.
```

### `instructions/roles/backend-engineer.md`

```markdown
# Backend Engineer Role

@extends: base/universal.md, base/code-quality.md, base/security.md

## Patterns
- Follow the repository pattern established in /src/repositories.
- Business logic belongs in services (/src/services), not controllers or routes.
- Database queries go through the repository layer — never raw SQL in controllers.

## Error Handling
- Every async function must have try/catch or equivalent error handling.
- Errors should be typed (use the project's AppError class, or create one if absent).
- Return structured error responses: { success: false, error: { code, message } }

## Testing
- Write a unit test for every service method you create.
- Integration tests go in /tests/integration.
- Do not mock the database in integration tests.

## API Design
- Follow RESTful conventions unless the project uses a different documented standard.
- Document all endpoints with JSDoc or the project's standard (check /docs/api-conventions.md if it exists).
- Validate request payloads before processing — use zod, joi, or the project's validator.
```

### `instructions/domains/payments.md`

```markdown
# Payments Domain Rules

These rules apply to any agent working with payment-related code.

## Compliance
- PCI DSS: Never store full card numbers, CVVs, or magnetic stripe data.
- All payment processing must happen server-side — never in the browser.
- Payment state must be idempotent — duplicate requests must not double-charge.

## Audit Trail
- Every payment action must produce an audit log entry via AuditService.
- Audit entries must include: timestamp, user_id, action, amount, status, idempotency_key.
- Never use console.log for payment events — use AuditService.log() exclusively.

## Error Handling
- Payment failures must return descriptive error codes (not generic 500 errors).
- Failed payment attempts must be logged but must NOT expose Stripe error details to end users.
- Implement retry logic for network failures using exponential backoff (max 3 retries).

## Patterns
- All Stripe calls go through /src/services/stripe.service.ts — never call Stripe SDK directly from a controller.
- Webhook handlers must verify signatures before processing any event.
```

---

## Part B: Build the Instruction Resolver

### `resolve-instructions.sh`

```bash
#!/usr/bin/env bash
# resolve-instructions.sh
# Usage: ./resolve-instructions.sh <agent-config.json> [output-file]
# Merges all referenced instruction files into a single resolved system prompt

set -euo pipefail

AGENT_CONFIG="${1:-agent-config.json}"
OUTPUT_FILE="${2:-resolved-instructions.md}"

if [[ ! -f "$AGENT_CONFIG" ]]; then
  echo "Error: Agent config '$AGENT_CONFIG' not found." >&2
  exit 1
fi

AGENT_NAME=$(jq -r '.agent' "$AGENT_CONFIG")
> "$OUTPUT_FILE"

echo "# Resolved Instructions: $AGENT_NAME" >> "$OUTPUT_FILE"
echo "# Generated: $(date -u +%Y-%m-%dT%H:%M:%SZ)" >> "$OUTPUT_FILE"
echo "" >> "$OUTPUT_FILE"

echo "Resolving instructions for: $AGENT_NAME"
echo ""

TOTAL_LINES=0
INCLUDE_COUNT=0

# Process includes in order (order matters — later files can override earlier ones)
jq -r '.includes[]' "$AGENT_CONFIG" | while read -r include_path; do
  if [[ -f "$include_path" ]]; then
    LINE_COUNT=$(wc -l < "$include_path")
    echo "  ✓ Including: $include_path ($LINE_COUNT lines)"

    echo "" >> "$OUTPUT_FILE"
    echo "---" >> "$OUTPUT_FILE"
    echo "<!-- Source: $include_path -->" >> "$OUTPUT_FILE"
    cat "$include_path" >> "$OUTPUT_FILE"
    echo "" >> "$OUTPUT_FILE"
  else
    echo "  ✗ WARNING: Include not found: $include_path" >&2
  fi
done

# Append overrides last (they always win)
OVERRIDE_COUNT=$(jq '.overrides | length' "$AGENT_CONFIG")
if [[ "$OVERRIDE_COUNT" -gt 0 ]]; then
  echo "" >> "$OUTPUT_FILE"
  echo "---" >> "$OUTPUT_FILE"
  echo "# Agent-Specific Overrides (highest priority)" >> "$OUTPUT_FILE"
  echo "" >> "$OUTPUT_FILE"
  jq -r '.overrides[]' "$AGENT_CONFIG" | while read -r override; do
    echo "  ✓ Override: $override"
    echo "- $override" >> "$OUTPUT_FILE"
  done
fi

FINAL_LINES=$(wc -l < "$OUTPUT_FILE")
ESTIMATED_TOKENS=$(awk "BEGIN {print int($FINAL_LINES * 15)}")

echo ""
echo "======================================"
echo "  Resolved: $OUTPUT_FILE"
echo "  Lines: $FINAL_LINES"
echo "  Est. tokens: ~$ESTIMATED_TOKENS"
echo "======================================"
```

---

## Part C: The Exercise

### Setup

Create the directory structure:
```bash
mkdir -p instructions/base instructions/roles instructions/domains
```

Fill in the files:
1. Copy the base files from Part A above
2. Create `instructions/roles/frontend-engineer.md` with 5 frontend-specific rules
3. Create your own `agent-config.json` for a specific task

### Your Agent Config

```json
{
  "agent": "my-specialized-agent",
  "includes": [
    "instructions/base/universal.md",
    "instructions/base/security.md",
    "instructions/roles/backend-engineer.md"
  ],
  "overrides": [
    "Write your most important domain-specific rule here.",
    "Add a second override that the base rules don't cover."
  ]
}
```

### Run the Resolver

```bash
chmod +x resolve-instructions.sh
./resolve-instructions.sh agent-config.json resolved.md
cat resolved.md
```

### Compare the Results

Run this prompt with two different system prompts:

**Test prompt:** *"I need to add an endpoint that accepts a payment method ID from the user and charges $50."*

1. First, use a generic "you are a helpful coding assistant" system prompt
2. Then, use your `resolved.md` output as the system prompt

**What to notice:**
- Does the specialized agent catch the security issues (user-supplied payment data, missing idempotency)?
- Does it reference your audit service?
- Does it suggest the correct architecture (controller → service → stripe.service)?

---

## Inheritance Rules (Quick Reference)

| Scenario | What to do |
|---|---|
| Rule applies to all agents | Put it in `base/universal.md` |
| Rule is role-specific | Put it in `roles/<role>.md` |
| Rule is domain-specific | Put it in `domains/<domain>.md` |
| An agent needs to break a base rule | Use `overrides` — and document why |
| Two agents share rules but aren't the same role | Create a new shared module in `base/` |
| A rule seems to fit multiple places | Prefer the most specific layer that covers all the cases |

---

## Takeaways

- **Composition > copy-paste.** Write rules once, reference many times.
- **Specificity wins.** More specific instructions (overrides) always beat general ones.
- **The resolver is just a skill.** It can be called as part of your agent-spawn workflow.
- **Token awareness matters.** Resolved instructions still consume context — measure them and budget accordingly.

---

*Optional Lesson 1 complete.*
*Artifacts to keep: `instructions/` directory, `resolve-instructions.sh`, your agent configs*
