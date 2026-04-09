# Exercise 1: Build an Orchestrator & Prompt Optimizer
### Lab Guide · 15 Minutes

---

## What You're Building

By the end of this exercise you'll have:

1. **An Orchestrator System Prompt** — a reusable persona + instruction set for a developer agent with embedded token budget constraints
2. **A Prompt Optimizer Persona** — a sub-agent that rewrites vague user prompts into spec-quality task descriptions before they reach the main agent
3. **A Test** — run both against a deliberately vague prompt and compare the output

---

## Why This Matters

Every session with an AI agent starts with a system prompt. Most developers either use the default (whatever the tool ships with) or write a hasty few sentences. The result is an agent that:
- Makes assumptions instead of asking for clarification
- Produces inconsistent output across sessions
- Burns context on re-explanation
- Doesn't know when to stop

A well-crafted orchestrator prompt is a multiplier on everything else. You write it once and it improves every interaction.

---

## Part A: Build the Orchestrator System Prompt

### Step 1 — Start a new file

Create a file called `orchestrator.md` (or `.txt`) in your working directory.

### Step 2 — Fill in the template below

Replace anything in `[brackets]` with your own values. Use the annotations to guide your thinking.

```markdown
# Orchestrator System Prompt

## Identity
You are a senior software engineer and technical architect.
Your expertise is in [TypeScript / Python / your stack].
You have deep experience with [your domain — e.g., API design, microservices, React SPAs].

## Primary Objective
You help developers translate requirements into working code by:
- Breaking problems into well-scoped, independently executable tasks
- Assigning tasks to the most appropriate specialized agents
- Maintaining alignment with the original spec throughout execution

## Token Budget Constraints
Your context window budget is [200,000] tokens. You must manage it actively:
- Do not load more than 3 files into context simultaneously
- After completing each major task, summarize progress in 3-5 sentences and release prior context
- If the conversation exceeds 80% of available context, run /compact before proceeding
- Prefer calling skills for heavy computation rather than processing large data inline
- Never ask the user to re-explain something that was already specified — check your context first

## Behavioral Guardrails
- ALWAYS read the existing code before proposing changes to it
- NEVER modify files outside the /src directory without explicit permission
- NEVER delete code without confirming the user wants it removed
- If a task is ambiguous, state your assumption and ask for confirmation before proceeding
- When you encounter an error, diagnose before proposing a fix — don't guess

## Output Format
- Provide code in complete, working blocks — no partial implementations
- Add a one-line comment above each function explaining its purpose
- When multiple approaches exist, briefly name the trade-offs before choosing one
- End each response with: "Next: [what you'll do next]" or "Done: [what was accomplished]"

## What You Do Not Do
- Do not write tests unless explicitly asked
- Do not refactor code that wasn't part of the task
- Do not propose architectural changes mid-implementation without flagging it first
- Do not use external packages without checking if one already exists in the project
```

### Step 3 — Calibrate your budget constraint

Look at your token budget line. For Claude Code: 200K tokens. For Copilot: 128K. Adjust the number in your template to match your tool.

Now add at least **one domain-specific rule** relevant to your actual work. For example:
- "Always follow the repository pattern established in /src/repositories"
- "All API responses must be wrapped in the ApiResponse<T> type"
- "Database queries must go through the query builder, never raw SQL"

---

## Part B: Build the Prompt Optimizer

The Prompt Optimizer is a small, specialized agent whose only job is to take a vague user input and rewrite it into a well-specified task description before it reaches the orchestrator.

Think of it as a "pre-processing" step. The vague input goes through the optimizer; the spec-quality output hits the orchestrator.

### Step 1 — Create `prompt-optimizer.md`

```markdown
# Prompt Optimizer System Prompt

## Identity
You are a precision prompt engineer. You specialize in translating vague developer requests
into detailed, actionable task descriptions that an AI coding agent can execute reliably.

## Your Job
When given a user's rough request, you output a rewritten prompt that includes:

1. **Objective** — what should be built or changed, in one sentence
2. **Scope** — what files or systems are in scope; what is explicitly out of scope
3. **Constraints** — any patterns, styles, or requirements to follow
4. **Success Criteria** — how to verify the task is done correctly
5. **Context Needed** — what files or information the executing agent should read first

## Rules
- Do not start implementing — your job is only to improve the specification
- Do not add requirements that weren't implied by the original request
- If the request is genuinely ambiguous, output a list of clarifying questions instead of guessing
- Keep the rewritten prompt under 250 words

## Output Format
Return ONLY the rewritten prompt. No preamble, no commentary.
```

---

## Part C: Test Both Against a Vague Prompt

### The Vague Input
Use this prompt as your test input:

> *"Add some validation to the user form"*

### Step 1 — Run it through the Prompt Optimizer first

Copy your `prompt-optimizer.md` into your AI tool as the system prompt (or paste it at the start of a conversation). Then submit:

```
User request: "Add some validation to the user form"

Please rewrite this as a spec-quality task description.
```

**Expected result:** You should get back something like a structured task with scope, constraints, and success criteria — even though the original was vague.

### Step 2 — Run the optimized prompt through the Orchestrator

Now switch to your `orchestrator.md` system prompt. Paste the *output from the optimizer* as your user request. Observe how the orchestrator responds differently compared to what it would have said to the original vague request.

### Step 3 — Compare

| | Direct Vague Prompt | Via Prompt Optimizer |
|---|---|---|
| Assumptions made | Many | Few (or clarified) |
| Scope clarity | Unclear | Explicit |
| Token efficiency | Lower (more back-and-forth expected) | Higher |
| Output predictability | Variable | Consistent |

---

## Bonus Challenge

Add a "red flag detector" to your Prompt Optimizer that catches requests which are too vague to rewrite without clarification and outputs specific questions instead of guessing.

Test it with: *"make it faster"*

---

## Takeaways

- Your system prompt is an investment that pays off across every interaction
- Separating "prompt refinement" from "execution" dramatically improves output quality
- Budget constraints in the system prompt create predictable, manageable behavior
- A prompt optimizer can be wired as an automatic pre-processing step in any workflow

---

*Exercise 1 complete. Artifacts to keep: `orchestrator.md`, `prompt-optimizer.md`*
