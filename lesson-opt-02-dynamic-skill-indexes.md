# Optional Lesson 2: Dynamic Agents via Skill-Managed Indexes
### Lab Guide · 15 Minutes

---

## The Core Idea

Static agent setups tell the agent *what to know*. Dynamic agents let the agent *discover what it needs*.

The difference is the difference between giving someone a 300-page manual and giving them a well-indexed library they can search. One fills their head with noise. The other lets them find the signal.

A **skill-managed index** is a structured catalog of your project's resources. The agent queries it before loading any context — getting only what's relevant for the current task, within a specified token budget.

---

## Why This Matters at Scale

### Month 1
```
Agent loads: user.ts, user.service.ts, user.controller.ts
Context used: ~4K tokens. Clean and fast.
```

### Month 6
```
Agent loads (because you haven't updated the instructions):
user.ts, user.service.ts, user.controller.ts, product.ts, order.ts,
payment.ts, auth.ts, notification.ts, email.ts...
Context used: ~40K tokens. Half is irrelevant.
Quality: noticeably degraded at the edges.
```

### With Dynamic Indexes
```
Agent queries: lookup-resources("users, authentication")
Index returns: user.ts (~800 tokens), user.service.ts (~1200 tokens)
Context used: ~2K tokens. Precise. Relevant.
Quality: consistently high regardless of project size.
```

---

## The Resource Index Schema

```json
{
  "index_version": "2.0",
  "project": "my-saas-app",
  "last_updated": "2026-04-07T14:00:00Z",
  "resources": [
    {
      "id": "user-model",
      "path": "src/models/user.ts",
      "type": "model",
      "tags": ["users", "authentication", "database", "entities"],
      "description": "TypeScript interfaces and Zod schemas for the User entity. Includes: User, CreateUserDTO, UpdateUserDTO, UserRole enum.",
      "token_estimate": 420,
      "last_modified": "2026-03-20",
      "related": ["user-repository", "user-service"]
    },
    {
      "id": "user-repository",
      "path": "src/repositories/user.repository.ts",
      "type": "repository",
      "tags": ["users", "database", "queries"],
      "description": "Database access layer for users. Methods: findById, findByEmail, create, update, delete, findByRole.",
      "token_estimate": 680,
      "last_modified": "2026-03-22",
      "related": ["user-model", "user-service"]
    },
    {
      "id": "user-service",
      "path": "src/services/user.service.ts",
      "type": "service",
      "tags": ["users", "business-logic", "authentication"],
      "description": "Business logic for user management. Methods: register, login, updateProfile, changePassword, deactivateAccount.",
      "token_estimate": 1100,
      "last_modified": "2026-03-25",
      "related": ["user-model", "user-repository", "auth-service"]
    },
    {
      "id": "auth-service",
      "path": "src/services/auth.service.ts",
      "type": "service",
      "tags": ["authentication", "jwt", "sessions", "security"],
      "description": "JWT generation, validation, and session management. Methods: generateTokens, verifyToken, refreshSession, revokeSession.",
      "token_estimate": 890,
      "last_modified": "2026-04-01",
      "related": ["user-service", "auth-middleware"]
    },
    {
      "id": "auth-middleware",
      "path": "src/middleware/auth.middleware.ts",
      "type": "middleware",
      "tags": ["authentication", "authorization", "http", "security"],
      "description": "Express middleware for route authentication. Exports: requireAuth, requireRole, optionalAuth.",
      "token_estimate": 340,
      "last_modified": "2026-04-01",
      "related": ["auth-service"]
    },
    {
      "id": "payment-service",
      "path": "src/services/payment.service.ts",
      "type": "service",
      "tags": ["payments", "stripe", "billing", "subscriptions"],
      "description": "Stripe integration for payment processing. Methods: createPaymentIntent, confirmPayment, createSubscription, cancelSubscription.",
      "token_estimate": 1450,
      "last_modified": "2026-04-03",
      "related": ["stripe-config", "audit-service"]
    },
    {
      "id": "api-conventions",
      "path": "docs/api-conventions.md",
      "type": "documentation",
      "tags": ["api", "conventions", "rest", "error-handling", "pagination"],
      "description": "Team API design standards: REST conventions, error response format, pagination patterns, versioning.",
      "token_estimate": 780,
      "last_modified": "2026-02-14",
      "related": []
    },
    {
      "id": "db-schema",
      "path": "src/database/schema.sql",
      "type": "schema",
      "tags": ["database", "schema", "tables", "relationships"],
      "description": "Full database schema with table definitions, indexes, and constraints. Current tables: users, sessions, payments, subscriptions, audit_log.",
      "token_estimate": 2200,
      "last_modified": "2026-04-05",
      "related": ["user-model"]
    }
  ]
}
```

---

## Part A: Build the Lookup Skill

### `lookup-resources.sh`

```bash
#!/usr/bin/env bash
# lookup-resources.sh
# Usage: ./lookup-resources.sh "tag1,tag2,tag3" [max_tokens] [index_file]
#
# Queries the resource index and returns matching resources within a token budget.
# Output format is designed to be passed directly to an agent as context instructions.

set -euo pipefail

QUERY_TAGS="${1:-}"
MAX_TOKENS="${2:-15000}"
INDEX_FILE="${3:-resource-index.json}"

if [[ -z "$QUERY_TAGS" ]]; then
  echo "Usage: ./lookup-resources.sh \"tag1,tag2\" [max_tokens] [index_file]" >&2
  exit 1
fi

if [[ ! -f "$INDEX_FILE" ]]; then
  echo "Error: Index file '$INDEX_FILE' not found." >&2
  exit 1
fi

echo "======================================"
echo "  Resource Lookup"
echo "  Query tags: $QUERY_TAGS"
echo "  Token budget: $MAX_TOKENS"
echo "======================================"
echo ""

# Find matching resources, sort by token_estimate ascending (load small ones first)
# Then apply budget — greedy pack within limit
MATCHES=$(jq --arg tags "$QUERY_TAGS" '
  .resources | map(
    . as $r |
    ($tags | split(",") | map(ltrimstr(" ") | rtrimstr(" "))) as $query_tags |
    select(
      $r.tags | map(. as $t | $query_tags | index($t) != null) | any
    )
  ) | sort_by(.token_estimate)
' "$INDEX_FILE")

# Apply token budget
SELECTED=$(echo "$MATCHES" | jq --argjson budget "$MAX_TOKENS" '
  reduce .[] as $r (
    {"items": [], "used": 0};
    if (.used + $r.token_estimate) <= $budget
    then {"items": (.items + [$r]), "used": (.used + $r.token_estimate)}
    else .
    end
  )
')

TOTAL_TOKENS=$(echo "$SELECTED" | jq '.used')
COUNT=$(echo "$SELECTED" | jq '.items | length')

echo "Matched resources (within ${MAX_TOKENS} token budget):"
echo ""

echo "$SELECTED" | jq -r '.items[] |
  "  [\(.id)] \(.path)",
  "  Type: \(.type) | Est. tokens: \(.token_estimate)",
  "  Tags: \(.tags | join(", "))",
  "  \(.description)",
  ""'

echo "======================================"
echo "  Resources selected: $COUNT"
echo "  Tokens allocated: $TOTAL_TOKENS / $MAX_TOKENS"
echo "======================================"
echo ""
echo "Files to load:"
echo "$SELECTED" | jq -r '.items[].path'
```

---

## Part B: Build the Index Update Skill

Keeping the index fresh is critical. This skill integrates with your git workflow:

### `update-index.sh`

```bash
#!/usr/bin/env bash
# update-index.sh
# Usage: ./update-index.sh [index_file]
# Scans the project for new/changed files and updates their index entries.
# Run this on git commit or as a pre-commit hook.

set -euo pipefail

INDEX_FILE="${1:-resource-index.json}"

if [[ ! -f "$INDEX_FILE" ]]; then
  echo "Index not found. Creating new index at $INDEX_FILE..."
  echo '{"index_version": "2.0", "last_updated": "", "resources": []}' > "$INDEX_FILE"
fi

# Get files changed since last commit (or all files if new index)
CHANGED_FILES=$(git diff --name-only HEAD 2>/dev/null || find src docs -name "*.ts" -o -name "*.md" -o -name "*.sql")

UPDATED=0
ADDED=0

while IFS= read -r file; do
  [[ -z "$file" || ! -f "$file" ]] && continue

  # Estimate tokens (rough: word count * 1.3 for code, * 1.1 for markdown)
  EXTENSION="${file##*.}"
  WORD_COUNT=$(wc -w < "$file" 2>/dev/null || echo "0")
  if [[ "$EXTENSION" == "md" ]]; then
    TOKEN_EST=$(awk "BEGIN {print int($WORD_COUNT * 1.1)}")
  else
    TOKEN_EST=$(awk "BEGIN {print int($WORD_COUNT * 1.3)}")
  fi

  MODIFIED=$(date -r "$file" -u +%Y-%m-%d 2>/dev/null || date -u +%Y-%m-%d)

  # Check if entry exists
  EXISTS=$(jq --arg path "$file" '.resources | map(select(.path == $path)) | length' "$INDEX_FILE")

  if [[ "$EXISTS" -gt 0 ]]; then
    # Update existing entry's token estimate and modified date
    jq --arg path "$file" \
       --argjson tokens "$TOKEN_EST" \
       --arg modified "$MODIFIED" \
       '(.resources[] | select(.path == $path)) |= . + {
         token_estimate: $tokens,
         last_modified: $modified
       }' "$INDEX_FILE" > tmp_index.json && mv tmp_index.json "$INDEX_FILE"
    echo "  Updated: $file (~${TOKEN_EST} tokens)"
    UPDATED=$((UPDATED + 1))
  else
    # New file — add a stub entry (human fills in description and tags)
    NEW_ID=$(echo "$file" | sed 's|/|-|g; s|\.|--|g')
    jq --arg id "$NEW_ID" \
       --arg path "$file" \
       --arg ext "$EXTENSION" \
       --argjson tokens "$TOKEN_EST" \
       --arg modified "$MODIFIED" \
       '.resources += [{
         "id": $id,
         "path": $path,
         "type": $ext,
         "tags": ["NEEDS_TAGS"],
         "description": "NEEDS_DESCRIPTION",
         "token_estimate": $tokens,
         "last_modified": $modified,
         "related": []
       }]' "$INDEX_FILE" > tmp_index.json && mv tmp_index.json "$INDEX_FILE"
    echo "  Added (stub): $file — ⚠️  add tags and description"
    ADDED=$((ADDED + 1))
  fi

done <<< "$CHANGED_FILES"

# Update index timestamp
jq --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" '.last_updated = $ts' "$INDEX_FILE" > tmp_index.json && mv tmp_index.json "$INDEX_FILE"

echo ""
echo "Index update complete: $UPDATED updated, $ADDED new stubs"
echo "Run: jq '.resources[] | select(.tags[] == \"NEEDS_TAGS\")' $INDEX_FILE"
echo "to find entries that need human review."
```

---

## Part C: Wire the Index Into Agent Instructions

Add this section to any agent's system prompt or `CLAUDE.md`:

```markdown
## Dynamic Resource Loading

Before loading any files into context, you MUST query the resource index.

### How to use the index
1. Identify the relevant tags for your current task
   - For user-related work: "users", "authentication"
   - For payment work: "payments", "billing", "stripe"
   - For API work: "api", "rest", "http"
   - When unsure, start broad and narrow down

2. Call the lookup skill:
   ```
   ./lookup-resources.sh "tag1,tag2" 15000
   ```

3. Load ONLY the files returned by the lookup — nothing else.

4. If you need a file not in the index results, explain why and request it explicitly.

### Token budget for context loading
- Task context budget: 15,000 tokens
- This covers ~3-5 average source files
- If your query returns more than the budget, prioritize files with the lowest token estimate

### Never do this
- Do not load entire directories
- Do not load files "just in case"
- Do not ask the user to provide files — check the index first
```

---

## Part D: The Exercise

### Step 1 — Create your index

Copy the example `resource-index.json` above (or build your own from a real project). If using a real project, run `update-index.sh` to bootstrap it from your git history.

### Step 2 — Run some lookups

Test your `lookup-resources.sh` with different task scenarios:

```bash
# Task: "Fix a bug in user login"
./lookup-resources.sh "authentication, users" 10000

# Task: "Add a new payment method"
./lookup-resources.sh "payments, stripe, billing" 15000

# Task: "Update the API error format"
./lookup-resources.sh "api, conventions, error-handling" 8000

# Task: "Write a database migration for a new table"
./lookup-resources.sh "database, schema" 20000
```

**Observe:** Do the right files come back? Are irrelevant files excluded? Does the budget constraint work?

### Step 3 — Test dynamic vs. static loading

With your AI tool, run the same task twice:

**Static approach:** Load 5 files you *think* are relevant before starting.

**Dynamic approach:** Give the agent the `resource-index.json` and `lookup-resources.sh`. Tell it to query the index before loading anything.

Compare:
- Tokens used
- First response quality
- Whether the agent loaded irrelevant context

### Step 4 — Add one real entry

Add one real resource from a project you actually work on. Write a genuine `description` and `tags`. Run the lookup against it.

---

## Advanced Pattern: Contextual Index Selection

For large projects, maintain multiple domain-specific indexes and select the right one based on the task:

```bash
# select-index.sh
TASK_DESCRIPTION="$1"

# Classify the task into a domain
DOMAIN=$(echo "$TASK_DESCRIPTION" | tr '[:upper:]' '[:lower:]')

if echo "$DOMAIN" | grep -qE "payment|billing|stripe|invoice"; then
  echo "resource-indexes/payments.json"
elif echo "$DOMAIN" | grep -qE "auth|login|session|jwt|password"; then
  echo "resource-indexes/auth.json"
elif echo "$DOMAIN" | grep -qE "api|endpoint|route|controller"; then
  echo "resource-indexes/api.json"
else
  echo "resource-indexes/general.json"
fi
```

Usage:
```bash
INDEX=$(./select-index.sh "Add payment retry logic")
./lookup-resources.sh "payments, retry, error-handling" 15000 "$INDEX"
```

---

## Takeaways

- **Indexes are living documents** — maintain them or they lose value
- **Tag quality determines lookup quality** — invest in good tags and descriptions
- **Budget enforcement in the lookup skill** creates predictable context consumption
- **Dynamic loading scales with your project** — static loading degrades as it grows
- **The index itself is a context artifact** — keep it lean (under 5K tokens for most projects)

---

*Optional Lesson 2 complete.*
*Artifacts to keep: `resource-index.json`, `lookup-resources.sh`, `update-index.sh`*
