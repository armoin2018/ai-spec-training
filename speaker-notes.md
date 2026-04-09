# Speaker Notes: From Prompt to Product — Building AI Apps by Spec
### Full Presenter Script · 90-Minute Workshop

---

## Pre-Session Setup (Before Participants Arrive)

Have these ready:
- Terminal open with your AI tool of choice (Claude Code recommended)
- Exercise starter files in a clean directory
- Slide deck on screen
- Exercise lab guides printed or shared digitally

---

## Slide: Welcome / Title

**[Timing: 3 min]**

"Good morning / afternoon everyone. Welcome to *From Prompt to Product: Building AI Apps by Spec*.

Before I get into the content, I want to ask: raise your hand if you've had this experience — you describe what you want to an AI coding tool, it builds something, and it's... kind of right? And then you clarify, and it's still not quite right? And by the time you're on the fourth or fifth iteration, you realize you've spent more time explaining yourself than you would have spent just writing the code?

[pause for hands / laughs]

Yeah. That's almost everyone. That experience isn't a skills problem or an AI problem — it's a *process* problem. And today, we're going to fix it.

In the next 90 minutes we're going to cover what agentic AI actually is, why the informal 'vibe coding' approach breaks down at scale, and how you can build a structured framework called *Spec Building* that turns AI from a frustrating autocomplete into a reliable development partner. We'll do three hands-on exercises, and you'll leave with working artifacts you can use on a real project tomorrow.

Let's get into it."

---

## Slide: What We'll Cover

**[Timing: 1 min — quick scan]**

"Here's our roadmap for today. Six topic areas, three exercises, tight pacing. I'll flag when we're transitioning to exercises and you'll have lab guides for each. Any questions before we start?"

---

## Slide: Beyond Autocomplete — What Is Agentic AI?

**[Timing: 4 min]**

"So what do we actually mean by *agentic* AI? This is a word that gets thrown around a lot right now so let's pin it down.

Most of you have used AI assistants that work like a smarter search engine — you ask, it answers, you go do the thing. That's the old model.

Agentic AI is fundamentally different. You give it a *goal*, and it figures out the plan to achieve it. It uses tools — it can read and write files, run code, call APIs, search the web. It executes multiple steps without you babysitting each one. It can even spawn other agents to work in parallel.

The mental model shift here is important: stop thinking of your AI tool as a very fast typist and start thinking of it as a junior developer who works at machine speed. That changes how you interact with it. You wouldn't walk up to a junior dev every 30 seconds and say 'okay now do this, now do this' — you'd give them a spec and let them run.

That's what we're building toward today."

---

## Slide: The Agentic Toolscape

**[Timing: 3 min]**

"Let's quickly map the landscape. These are the major tools you're likely to encounter or already be using.

GitHub Copilot has become ubiquitous — inline suggestions, chat in the IDE, PR reviews. It's powered by GPT-4o and increasingly it's acting more like an agent, especially in Copilot Workspace.

Cursor and Windsurf are AI-first IDEs. Rather than adding AI to an existing editor, they're built around the idea of a Composer or Cascade — where you describe a change and the AI makes it across multiple files.

Claude Code is Anthropic's CLI agent. It runs in your terminal, has direct access to your file system and shell, and is particularly strong at planning and multi-step tasks. That's what we'll use most heavily today.

What's critical here is that all of these tools operate on the same primitives — the same building blocks we'll cover shortly. Once you understand those, you can use any of these tools effectively."

---

## Slide: The Agentic Loop

**[Timing: 3 min]**

"Here's what's actually happening under the hood when an agent does work.

You provide an intent. The agent has a system prompt — a persona and instructions. It reasons about what steps to take. Then it calls tools — maybe it reads a file, runs a script, calls an MCP server. It gets results back, reasons about the next step, calls more tools, and eventually returns an artifact to you.

Notice this loop. The agent reasons, acts, observes results, reasons again. Each step consumes tokens — from your context window.

This is why *context management* becomes critical. The more steps in the loop, the more tokens consumed. If you fill up the context window with noise — repeated explanations, unnecessary files, redundant history — you cut into the budget the agent needs to actually do the work.

We'll spend significant time on this later."

---

## Slide: What Is Vibe Coding?

**[Timing: 2 min]**

"Vibe coding. I love this term because it's perfectly descriptive. It's when you just describe what you want and let the AI figure out the rest. No structure, no constraints, just vibes.

And here's the thing — it works! For prototypes, for quick explorations, for 'I want to see if this is even possible,' vibe coding is fantastic. The AI is great at getting you to 60% in ten minutes.

The problem is the remaining 40%. And the problem with maintaining what you built. And the problem with picking it back up tomorrow when the context is gone."

---

## Slide: Why Vibe Coding Breaks Down

**[Timing: 4 min]**

"Let me walk you through what I call the Degradation Spiral, because most of you have lived it.

You ask the AI to build something. It makes reasonable assumptions — because you didn't specify, it filled in the blanks. Fine. It's close.

You clarify. But you're clarifying in the chat, which means those clarifications are now part of your context. The model has to hold all of that in memory while also doing the work.

You keep going. The context fills up. At around 60-70% context utilization, models start to get a little fuzzy at the edges. They forget earlier constraints. They contradict decisions they made three iterations ago.

And now you're spending more tokens explaining what you already explained than you are on new work. Tokens cost money at scale. And more importantly, they cost you *time*.

There's also the reproducibility problem. When everything lives in a conversation, you can't hand it off. You can't pick it up in a new session without re-explaining everything. You can't share it with a colleague. It's all in your head.

Spec Building solves all of this."

---

## Slide: The Spec Advantage

**[Timing: 2 min]**

"Look at the comparison here. Everything about vibe coding that breaks down — implicit intent, context bloat, single-session limitation — is solved by moving to specs.

A spec is a structured file, or set of files, that captures everything the AI needs to do good work without requiring you to be present in the conversation explaining yourself.

This is the mindset shift: instead of *talking* to the AI in real time, you're *writing contracts* that the AI executes."

---

## Slide: Spec Building Defined

**[Timing: 3 min]**

"Let me give you a working definition. A spec is a structured document — or set of documents — that gives an AI agent everything it needs to execute a task reliably, without requiring you to be present.

Three layers:

First, *what* you want built. Your requirements — what problem are you solving, what does success look like, what's in scope.

Second, *how* it should be built. Technical constraints, patterns to follow, architecture decisions.

Third, *who* does the work. Which agents, with what personas, under what guardrails.

When you hand a developer a spec, they can go run. A well-written AI spec should do the same thing."

---

## Slide: The Spec → Plan → Execute Pipeline

**[Timing: 3 min]**

"Here's the pipeline we'll be building toward. Each of these stages is a distinct artifact.

Business Requirements captures the *why* and the *what* at a business level.

Technical Requirements translates that into engineering terms — stack, APIs, data models.

Design and Architecture visualizes the system structure.

The Plan — which we'll build in Exercise 3 — is a structured JSON document that breaks all of this into discrete tasks, assigns agents, and identifies what can run in parallel.

Then Scaffold, Build, and Assess are your execution phases.

This is our map for the rest of the session."

---

## Slide: Meet the Building Blocks

**[Timing: 2 min]**

"Before we go deeper, I want to give you a vocabulary. Because one of the things that makes agentic AI confusing is that everyone uses different words for the same concepts.

These are the six building blocks. Every agentic system you'll encounter — Claude Code, Cursor, Copilot, doesn't matter — is built from some combination of these six things.

[read table briefly]

Agent, Instructions, Context, Prompts, Skills, MCP. We'll go through each one."

---

## Slide: Who — Agents & Personas

**[Timing: 3 min]**

"An agent is an AI model with a specific identity and purpose. A persona is what gives that agent its expertise and scope.

Think about the difference between asking a general LLM 'how do I fix this bug?' versus asking an agent that's been configured as 'a senior TypeScript developer who specializes in React performance optimization, never modifies test files without explicit permission, and always explains trade-offs before proposing solutions.'

That second version is going to give you dramatically better output on React performance problems. Because you've narrowed the scope. You've established expertise. You've set behavioral boundaries.

The system prompt example on this slide is real — and it takes about two minutes to write. The ROI is enormous."

---

## Slide: Guardrails — Instructions

**[Timing: 3 min]**

"Instructions are the rules. What the agent must do, what it must never do, what format its output should take.

The most important thing I can tell you about instructions: put them in files, not in the chat.

In Claude Code, there's a file called `CLAUDE.md` at the root of your project. Anything in that file is automatically loaded into every session. GitHub Copilot has `.github/copilot-instructions.md`. Whatever tool you're using, find the equivalent.

This is how you get persistent behavior. If you have to tell your AI tool 'by the way, we use tabs not spaces' every single session, that's a sign your instructions are living in the wrong place."

---

## Slide: Where — Context

**[Timing: 3 min]**

"Context is everything the agent knows about where it's working. The codebase, documentation, prior decisions, the current state of the world.

Here's the key tension: context is your most valuable resource and your most limited one. You only get one context window, and it has a finite size.

Most developers' instinct when a task fails is to load *more* context into the model. Give it more files. More history. More information. And often that makes things worse, not better.

The skill of context management is knowing what to include and what to exclude. We'll cover specific strategies shortly."

---

## Slide: What — Prompts

**[Timing: 3 min]**

"Prompts are the actual task request — the 'what do I want right now.'

The difference between a good prompt and a bad prompt isn't length. It's specificity and scope.

'Clean up the code' is a bad prompt. It gives the AI enormous latitude and no success criteria.

'Refactor getUserById to use the repository pattern, consistent with how getProductById is implemented in product.service.ts, without changing the function signature' is a good prompt. Specific, scoped, has an explicit success criterion.

The slash commands we build in Exercise 2 are essentially prompt templates — reusable, well-formed prompts for common tasks."

---

## Slide: How — Skills vs MCP

**[Timing: 5 min — important conceptual distinction]**

"This is one of the slides I want you to really internalize, because there's a lot of confusion in the field about the difference between Skills and MCP.

Skills are reusable scripts that the agent can execute in your environment. They run on your machine, in your terminal, as shell scripts or Python or Node. A skill might be a script that parses a JSON plan, runs your test suite and returns a summary, generates a report, or compacts the conversation history. You write skills. You own skills. They're internal.

MCP — Model Context Protocol — is a standardized way for agents to connect to external services. Slack, GitHub, Jira, databases, APIs. An MCP server runs as a separate process and exposes capabilities to the agent. You don't write MCP servers for Slack — someone else did, and you install and configure them.

The rule of thumb I use: Skills for doing, MCP for connecting.

If I need to process a JSON file — that's a skill. If I need to post to Slack — that's MCP. If I need to check availability in my calendar — that's MCP. If I need to run my test suite and parse the output — that's a skill.

They can absolutely work together. An orchestrator agent might call an MCP server to fetch data, pipe it through a skill for processing, and then call another MCP to post the result."

---

## Slide: Subagents & Parallel Execution

**[Timing: 3 min]**

"Here's where things get really interesting.

When you have a plan with independent tasks — writing tests for module A doesn't depend on implementing module B — there's no reason to run them sequentially. You can spawn multiple agents and run them in parallel.

Think about the throughput implication. If you have four independent tasks that each take 10 minutes, running them sequentially takes 40 minutes. Running them in parallel takes 10 minutes.

More importantly: each subagent gets a focused context. Instead of one giant context holding everything, each subagent has a lean context with just what it needs for its specific task. That improves quality and reduces cost.

This is what Exercise 3 is going to demonstrate. We'll build a plan, extract the tasks, and spawn them in parallel."

---

## Slide: Context Window Reference

**[Timing: 2 min]**

"Quick reference card for context windows across the major models. I want you to notice a few things.

Gemini has the biggest windows — 1 million tokens. Claude and GPT-4o are in the 128K-200K range. Copilot in an IDE is actually more limited because of how it handles the surrounding code context.

But here's what I want you to take away from this table: bigger window doesn't mean better. 1 million tokens of context is genuinely useful for specific tasks — like analyzing a giant codebase. But for most development work, 200K is more than enough if you're managing it well.

Quality tends to degrade at the far edges of very large windows. And bigger windows mean bigger bills. Budget management matters regardless of which model you're using."

---

## Slide: Request Size vs Response Size

**[Timing: 2 min]**

"Your context window has two sides: what goes in, and what comes out.

What goes in: system prompt, instructions, conversation history, files, tool results. This is where most of your context gets consumed.

What comes out: the model's response, generated code, plans.

The important thing to understand is that your 'input' isn't just your message. Every file you've loaded, every tool result that came back, every prior message in the conversation — all of that is input tokens on the next request. Context accumulates."

---

## Slide: Budget Allocation Strategy

**[Timing: 3 min]**

"I want you to think about your context window like a project budget with line items. You have a total. You allocate to different categories. You leave a reserve for unexpected costs.

Here's an example allocation for a 200K token window. System prompt and persona: about 2K. CLAUDE.md instructions: maybe 3K. Current file and active context: 20K. Conversation history: 30K. Tool results: 15K.

That leaves roughly 130K as your working budget. Keep 20K in reserve — models get weird when they're approaching their limit. So your effective working space is about 110K.

For most development tasks, 110K is plenty. The goal is to structure your work so you never have to blow that budget."

---

## Slide: Keeping Within Budget

**[Timing: 4 min]**

"Four practical strategies.

First: put budget constraints in your instructions. Explicitly tell the agent 'do not load more than three files into context at once' or 'summarize completed work before starting the next task.' Models follow these instructions if you give them.

Second: compacting. This is asking the agent to summarize what's happened so far, forget the full transcript, and continue with just the summary. In Claude Code, `/compact` does this. It's like creating a checkpoint — you preserve the important information and free up context space.

Third: AI offloading to skills. Instead of loading a large file into context for the model to parse, write a skill that does the parsing and returns only the relevant result. The model only sees the output, not the 500-line input file.

Fourth: memory files. Important decisions, project facts, ongoing context — put these in `CLAUDE.md` or dedicated memory files rather than repeating them in the chat. They load automatically, they're versioned with your code, and they survive context resets.

These four strategies together can reduce your context consumption by 50-70% on a typical project."

---

## [TRANSITION TO EXERCISE 1]

**[Timing: 1 min]**

"Alright, enough theory. Let's build something.

For the next 15 minutes, we're going to do Exercise 1. You're going to build an orchestrator system prompt with budget constraints baked in, and a prompt optimizer that rewrites vague user inputs into spec-quality prompts before they hit the main agent.

Pull up your Exercise 1 lab guide. I'll be walking around. Let's go."

---

## [EXERCISE 1 DEBRIEF — 3 min]

"Great. Before we move on — who wants to share their orchestrator prompt? What constraints did you add?

[field 1-2 responses]

Notice how different everyone's prompts are. That's correct — because you each have different contexts and constraints in mind. The point isn't to memorize a template; it's to develop the habit of writing explicit constraints rather than assuming the model will infer them.

Let's keep moving."

---

## Slide: The 7 Phases of Spec Building

**[Timing: 4 min]**

"Now that you understand the building blocks and budget management, let's talk about the actual process of Spec Building.

Seven phases. Each one is a discrete document or artifact. Each one feeds the next.

Business Requirements answers: what problem are we solving, for whom, and how do we know we've succeeded?

Technical Requirements translates that into engineering: what stack, what APIs, what data models, what non-functional requirements like performance and security?

Design and Architecture gives us the system diagram, data flow, component boundaries.

The Plan is where we really get powerful — a structured JSON that breaks everything into tasks, assigns agents, and identifies what's independent enough to run in parallel.

Then Scaffold, Build, and Assess are the execution phases. Scaffold creates the skeleton. Build fills it in. Assess checks it against the original requirements.

The key insight: by the time you're in the Build phase, the AI has a complete, structured specification. It's not guessing at intent. It's executing a contract."

---

## Slide: Phase Breakdown

**[Timing: 3 min — quick scan, hit highlights]**

"Let me hit the highlights for each phase.

BRD: This is the business case. Problem, users, success metrics, scope. Most developers skip this and go straight to tech. Don't. Thirty minutes here saves days later.

TRD: Stack, API contracts, data models. This is where you make the technical decisions that cascade through everything else. Document your decisions *and your reasoning*.

Design: The diagram. The data flow. You need to see the system before you build it.

Plan: We'll build one of these in Exercise 3. JSON format, tasks with agent assignments, parallel opportunities flagged.

Scaffold: Your AI sets up the directory structure, creates stub files, configures tooling. Think of it as the AI creating its own workspace.

Build: Execution phase. Each task in the plan is assigned to an agent.

Assess: Did we build the right thing? Run it against the BRD. Check test coverage. Review the code."

---

## [TRANSITION TO EXERCISE 2]

**[Timing: 30 sec]**

"Exercise 2. We're going to build slash commands for each phase. You get 10 minutes. Lab guides are out — go."

---

## [EXERCISE 2 DEBRIEF — 2 min]

"Let's look at one or two of these. What did your `/brd` command look like?

[field a response]

The thing I want you to notice is how a well-structured prompt command essentially teaches the AI what a good BRD looks like. You're not asking it to figure that out — you're giving it a template and asking it to fill in the blanks from your input.

This is the leverage point of prompt engineering: front-load the structure, so the AI's energy goes into content, not figuring out format."

---

## Slide: Exercise 3 Overview

**[Timing: 2 min — framing]**

"Exercise 3 is the capstone. We're going to put everything together.

You'll write a plan.json — a structured task list for a small feature. Then you'll run a skill that uses `jq` to extract each task. Then your orchestrator will spawn those tasks as parallel subagents.

This is the pattern used in real production agentic systems. Once you've done it once, you can template it for any project.

Ten minutes. Let's go."

---

## [EXERCISE 3 DEBRIEF — 2 min]

"Who got theirs running end to end?

[field responses]

Even if you didn't finish — the important thing is you've seen the architecture. Plan as data. jq as extractor. Orchestrator as spawner. Subagents as workers.

This pattern scales. Four tasks today, forty tasks in production. The orchestrator doesn't change."

---

## Slide: Key Takeaways

**[Timing: 3 min]**

"Before I let you go — the six things I want you to remember.

One: Vibe coding doesn't scale. It's a great way to explore. It's a terrible way to build. Specs do scale.

Two: The six building blocks — agent, instructions, context, prompt, skills, MCP — are the vocabulary of agentic development. Learn these and you can work in any tool.

Three: Context is currency. Every token matters. Budget it, compact it, offload heavy processing to skills.

Four: Skills are for doing, MCP is for connecting. Use the right tool for the job.

Five: Parallel subagents multiply throughput. Plan your work with independence in mind.

Six: Spec Building gives you repeatability. You can hand off, reproduce, and improve. That's what production-grade AI development looks like.

These aren't just workshop concepts — this is how real teams are shipping with AI right now."

---

## Slide: Your Starting Kit

**[Timing: 1 min]**

"Everything you built today is yours. The slash commands, the orchestrator prompt, the plan template. These are not exercises — they're production-ready starting points.

The most valuable thing you can do when you get back to your machine: put your orchestrator system prompt in your CLAUDE.md or copilot instructions file. Start your next project by running `/brd` before you write a single line of code.

Two days from now you'll be moving faster than you thought was possible."

---

## Slide: Resources

**[Timing: 1 min]**

"Resources are on the last slide and in your lab materials. The Anthropic prompt engineering guide is particularly worth bookmarking — it's excellent and free.

Any questions before we wrap?"

---

## Closing

**[Timing: 1 min]**

"Thank you all. This was a genuinely great group — the energy in Exercise 3 especially was fantastic.

If you want to connect, ask questions, or share what you build: [your contact details / LinkedIn / email].

Good luck. Go build something real."

---

*End of script. Total runtime: ~90 minutes including exercises and transitions.*
*Exercises: Ex1 = 18 min (15 + 3 debrief), Ex2 = 12 min (10 + 2 debrief), Ex3 = 12 min (10 + 2 debrief)*
