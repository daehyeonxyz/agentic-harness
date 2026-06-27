**English** · [한국어](./README.ko.md)

# The Agentic Harness

*A field guide to the principles, boundaries, and operating philosophy of building a model-neutral AI agent harness on your own machine.*

> A good agent harness is not a pile of prompts or a roster of agents. It is a **compounding system**: a model-neutral scaffold with durable memory, capabilities you can evaluate and swap, and a way to upgrade itself without losing continuity. Build that, and the model inside becomes a replaceable part.

---

## What this is

This is not a tutorial, not product documentation, and not a pitch. It is a field guide to the **operating philosophy** of building a model-neutral, self-improving agent harness on your own machine. "Field guide" in the literal sense: a book for recognizing things in the wild and making good judgments about them, not a step-by-step manual. If you came looking for copy-paste setup, this is the wrong document, and that is on purpose.

It externalizes one mental model so you can reuse it: context as a scarce resource, memory as durable state, agents as context windows running inside execution environments, and capabilities as slots you can version and swap.

The goal is twofold. For you, the reader, it marks the shift from *using* AI to *engineering how you use it*. For me, it is a thinking artifact: a way to make an architecture legible, durable, and portable across tools, models, and runtimes.

**Who it is for.** Anyone who wants to use AI well, whether or not you write code. The principles are universal: spend attention wisely, keep durable knowledge outside the conversation, delegate to specialists, check results, stay free of any single vendor. The deeper mechanics (the dispatch contract, hooks, the capability registry) describe how a builder makes that philosophy concrete. You do not need to implement them to absorb the ideas behind them.

**The two claims that matter most**, stated up front because the rest of the guide is in service of them:

1. A harness should **improve over time**. Its capabilities are a versioned, swappable, evaluable registry, not a frozen prompt. When you find a better way to do something, anywhere, you slot it in and the whole system gets better at once.
2. A harness should **keep a knowledge base it maintains for itself**. Durable memory, curated and loaded only when needed, kept separate from the working conversation. This is the difference between an assistant that re-learns you every session and one that compounds.

Most people who build with AI underinvest in both. This guide is mostly about why, and what to do instead.

---

## Contents

1. [Why a harness](#1-why-a-harness)
2. [The one mental model](#2-the-one-mental-model)
3. [Workflows and agents](#3-workflows-and-agents)
4. [Anatomy of a harness](#4-anatomy-of-a-harness)
5. [Context engineering](#5-context-engineering)
6. [Teams and the orchestrator-worker pattern](#6-teams-and-the-orchestrator-worker-pattern)
7. [The dispatch contract](#7-the-dispatch-contract)
8. [The knowledge base](#8-the-knowledge-base)
9. [Continuous improvement](#9-continuous-improvement)
10. [Enforcement and honesty](#10-enforcement-and-honesty)
11. [Model neutrality](#11-model-neutrality)
12. [What people overlook](#12-what-people-overlook)
13. [Operating principles](#13-operating-principles)
14. [Closing](#14-closing)
15. [Glossary and sources](#15-glossary-and-sources)

---

## 1. Why a harness

Most people *use* an agent. They open a chat, type, and read. That is fine for one-off tasks. But the moment you want consistency across sessions, an agent that knows who you are, specialized helpers instead of one generalist, coordinated sub-tasks, and a system that gets better as you go, you are no longer using a model. You are building a system around it. That system is the harness, and it is the part you own.

The labs that build these systems start their advice with the move most people skip. Anthropic's first recommendation in *Building Effective Agents* is to "find the simplest solution possible, and only increase complexity when needed." A single model call with good retrieval often beats an autonomous multi-agent setup on both reliability and cost. So treat everything below as an escalation you earn, not a place to begin. The payoff is not novelty but compounding: the continuity, structure, and growth a raw model cannot give you alone.

---

## 2. The one mental model

If you keep one idea from this guide, keep this one.

> **An agent is a context window running in an execution environment.**

The **context window** is the entire world the agent reasons over: its instructions, the tools available to it, the data it has retrieved, the conversation so far. Two "different agents" are really two different ways of filling a window. A useful consequence follows. A main agent, a sub-agent, and a separate independent agent are the same kind of thing. They differ only in who manages their context and how they were summoned, not in nature. So the question "should this be a sub-agent or a fresh session?" reduces to one real question: what should this window contain, and who manages it?

The atom underneath all of it is what Anthropic calls the **augmented LLM**: a model "enhanced with augmentations such as retrieval, tools, and memory," which generates its own queries, picks its own tools, and decides what to keep. That is the unit you compose with. Every helper in a harness is an augmented LLM scoped to one job.

### Context is finite, and it decays

The second half of the model, the environment, is ruled by a fact beginners miss: **context is finite and gets worse as it fills.** Anthropic names two effects. The first is an **attention budget**: "every new token introduced depletes this budget," so context is a resource with diminishing returns. The second is **context rot**: "as the number of tokens in the context window increases, the model's ability to accurately recall information from that context decreases."

The goal, then, is never *more* context. It is, in their words, "the smallest possible set of high-signal tokens that maximize the likelihood of some desired outcome." Read practically: your job is to decide what deserves to enter the window, what must stay outside it, and what should be remembered after the window is gone. Almost every choice later in this guide (isolation, lazy loading, summarized hand-backs, a knowledge base that loads on demand) is one answer to that one job.

Caching makes the same point in money. Model APIs cache the stable front of a context and charge little to read it back, so keeping the steady parts (identity, rules, catalogs) small and unchanging is both more reliable and cheaper. Design so the only thing that churns is the task.

---

## 3. Workflows and agents

Anthropic draws a line that organizes the whole harness. A **workflow** is a system where "LLMs and tools are orchestrated through predefined code paths." The control flow is fixed in code, and the model fills in steps. An **agent** is a system where "LLMs dynamically direct their own processes and tool usage." The model decides the path.

A good harness is deliberately both, and that is the point most builders miss. Its spine (the router, the fan-out engine, the checks between steps) is a workflow: fixed code paths with what Anthropic calls *gates*, programmatic checks that verify progress between steps. Its leaves (the individual helpers) are agents that self-direct inside their slot. You hard-code the orchestration precisely so each helper can improvise safely. Knowing which parts are workflow and which are agent is what separates a system you can reason about from one you can only hope works.

The named building blocks you will compose are Anthropic's: *prompt chaining*, *routing*, *parallelization*, *orchestrator-workers*, and *evaluator-optimizer*. Three carry most of the weight here. Routing is the front half of a team lead. Orchestrator-workers is the team itself. Evaluator-optimizer is both the quality gate and the improvement loop.

---

## 4. Anatomy of a harness

A harness is a small number of surfaces, each with one job. Conflating them is the most common mistake. Keep them apart, and name each with the discipline's real term.

| Surface | Official framing | Its one job |
|---|---|---|
| **Identity** | system prompt at the *right altitude* | who the agent is, how it speaks |
| **Your profile** | durable preference memory | who you are, how you work |
| **Router** | a *routing* workflow | classify intent, send to the right specialist |
| **Rules** | system-prompt instructions | the hard constraints on output and behavior |
| **Hooks** | *enforced configuration* | force a rule in code, not in prose |
| **Skills** | *progressive disclosure* | a capability loaded only when invoked |
| **Workers** | *sub-agents*, *context isolation* | scoped work in a clean window |
| **Knowledge base** | *structured note-taking*, *memory* | durable knowledge, loaded just in time |

Four of these are routinely misunderstood, so they are worth slowing down on.

**Right altitude.** Anthropic frames the system prompt as a zone between two failures: "brittle hardcoded if-else logic" on one side, and "vague high-level guidance that falsely assumes shared context" on the other. Identity and rules live here, and they are not free. Claude Code's own docs warn that a longer instruction file "consumes more context and reduces adherence," and note that `@`-imports do not save context because imported files still load in full. The practical rule: a rules layer has both a context cost and an adherence cost. Keep it lean, or it works against you.

**Rules versus hooks: the enforcement line.** This is the single most consequential distinction in a harness, and the docs say it plainly. Instruction files and memory are "context, not enforced configuration. To block an action regardless of what [the model] decides, use a PreToolUse hook instead." A rule the model reads is a rule the model can drop. A hook is code that runs at a fixed moment and cannot be argued with. (One operational detail people get wrong: only exit code 2 blocks, and only on events that allow blocking. Exit 1 is treated as non-blocking, despite Unix habit.) In this guide's terms: rules are the answer key, and hooks are what make the agent actually sit the exam.

**Skills and the economics of installing everything.** A skill describes a capability and how to use it, loaded by **progressive disclosure**. The labs' model loads only about 100 tokens of metadata at all times, the skill body (a few thousand tokens) when it is triggered, and deeper material only on demand. The consequence is freeing: you can keep a large library of capabilities at almost no standing cost. Reference material can sit on disk, costing nothing until something reads it. Trimming your capability library "to save context" optimizes the wrong thing.

**Workers and context isolation.** A worker is a sub-agent with a clean window. It "might explore extensively, using tens of thousands of tokens or more," then "returns only a condensed, distilled summary (often 1,000 to 2,000 tokens)" to whoever called it. Its two benefits are parallel work and context isolation, so noisy or heavy work never pollutes the caller's window. That is the whole reason a worker exists: not a new kind of agent, but a way to keep heavy, noisy work out of the caller's attention budget.

---

## 5. Context engineering

Anthropic's name for the discipline under everything is **context engineering**: "the set of strategies for curating and maintaining the optimal set of tokens (information) during LLM inference," where "the curation phase happens each time we decide what to pass to the model." You are not writing one prompt. Every turn, for every agent, you decide what is in the window, and the harness is the machine that makes those decisions well.

The techniques you will lean on are all named in the literature.

- **Just-in-time retrieval.** Instead of pre-loading data, an agent keeps "lightweight identifiers (file paths, stored queries, web links)" and loads the data "into context at runtime using tools." Pull on demand, not up front.
- **Progressive disclosure.** The agent "incrementally discover[s] relevant context through exploration. Each interaction yields context that informs the next decision."
- **Compaction.** Near the limit, summarize the conversation and start a fresh window from the summary. With a warning the docs make explicit: "overly aggressive compaction can result in the loss of subtle but critical context whose importance only becomes apparent later." More summarizing is not strictly better.
- **Structured note-taking.** The agent "regularly writes notes persisted to memory outside of the context window," pulled back in when needed. This is the bridge to the knowledge base in section 8.
- **Sub-agent isolation.** "Specialized sub-agents can handle focused tasks with clean context windows," while the main agent holds the plan.

Underneath, the five are a single stance: treat the window as scarce and curated, and keep durable state outside it.

---

## 6. Teams and the orchestrator-worker pattern

One generalist doing everything is a thousand-line function. Split the work by domain (for personal work, something like building, design, data, writing, research, knowledge, learning, planning, thinking, and a shared utility pool, though yours will differ). Give each domain a **lead**, and let leads coordinate workers. That pattern has a name and a pedigree: Anthropic's **orchestrator-worker**, made concrete in a multi-agent research system.

### The lead is a coordinator, not a doer

In their system the lead "analyzes queries, develops strategy, and spawns subagents," then "synthesizes their results and decides whether more [work] is needed." So define a lead as router, strategist, synthesizer, and gate. Note the part teams forget: the lead also owns the decision to stop. A lead plans and delegates. The moment it starts doing the work itself, it has stopped being a lead.

### Workers are specialists that cannot recruit

A worker does one thing in an isolated window with a scoped set of tools, and returns a distilled summary. The important boundary: a worker cannot spawn more workers. Enforce that in capability, not just in writing. If workers can spawn workers, coordination becomes a tree no one controls. Workers report. Only leads coordinate.

### Slots, names, and a litmus test

Inside a team, define functional **slots**, with one current provider for each. This is what makes the system upgradable, the subject of section 9: to improve a capability, swap its provider. But slots only stay legible if they are cleanly bounded, and here the tool-design literature is sharp. Anthropic "spent more time optimizing our tools than the overall prompt" on a coding benchmark, and found that the top failure is overlap: "the issue isn't solely the number of tools, but their similarity or overlap." Some teams run more than fifteen distinct tools without trouble. Others choke on fewer that overlap. Their litmus test: "if a human engineer can't definitively say which tool should be used... an AI agent can't be expected to do better." Fix slot names and boundaries before you add another worker. Grouping related capabilities under shared prefixes (namespacing) is the standard remedy.

### When to collapse the team

Now the honest counterweight. Multi-agent is not free, and it is the wrong tool for many tasks. Anthropic measured that "agents typically use about 4x more tokens than chat interactions, and multi-agent systems use about 15x more tokens," with "token usage by itself [explaining] 80% of the variance" in performance. It pays for "heavy parallelization, information that exceeds single context windows, and interfacing with numerous complex tools." It does not pay for "domains that require shared context or tight real-time coordination, like most coding tasks," where agents cannot yet share context well. So put effort-scaling rules into the lead (one worker for a simple lookup, many only for genuinely large research), and know when to collapse the team back to a single agent. OpenAI's practical guide gives the same shape: start with one agent, and reach for a **manager pattern** (a lead calling specialists as tools) only when one agent's instructions and tools stop fitting.

---

## 7. The dispatch contract

Fan-out without a contract is chaos. The value of a team is not "more agents." It is more agents under a contract that makes their output trustworthy and composable. The contract, with each clause grounded:

- **Brief completely.** Every delegation carries an objective, an output format, the tools or sources to use, and explicit boundaries. The failure is documented: "without detailed task descriptions, agents duplicate work, leave gaps, or fail to find necessary information." A delegation missing any of these is the named cause of wasted effort.
- **Hand off by reference, not by paste.** Pass a path or an identifier and let the worker load it. Pasting the full brief bloats context and loses track of where things came from.
- **Name the model at the moment of dispatch.** Workers default to inheriting the session's model. The dispatching call names the model it wants, which encodes a cost and accuracy intent for that one task. This is also the lever in section 9.
- **One owner per file, and parallelize only separate work.** During parallel work a file has exactly one writer. Never run two agents editing the same files at once. Parallel is for work that is independent.
- **Return a distilled summary in a fixed shape.** Each worker returns the same structure (what changed, what was done, errors, next steps) plus a small status (done, done with concerns, blocked, needs context), and keeps it inside a budget of roughly one to two thousand tokens. The reason: "running many subagents that each return detailed results can consume significant context." The win is isolating the intermediate work, not making it free, so the return budget is part of the contract.
- **Bound the depth and cap the count.** Workers are leaves. Coordination cost climbs fast with the number of agents. A few well-scoped workers beat a swarm of vague ones.

### The fan-out engine

The router invokes an orchestrating routine when a task needs more than one worker or spans several phases:

```text
intake  ->  classify size  ->  write a delegation plan (to a file)  ->  dispatch  ->  gate  ->  report up
```

Two non-obvious requirements. The size step applies explicit effort-scaling against that 15x-token reality, so the lead does not reach for the whole team by reflex. And the plan is written to a file first (an analysis, the team with each worker's owned files, and the order of work), because planning before spawning is what keeps a large fan-out coherent. A running ledger records what is done, so a run that is interrupted or compacted does not redo finished work.

### The gate is the point, and it is an evaluator-optimizer

The step most systems skip is the one that earns trust. After a worker returns, verify its result against the brief before accepting it. This is Anthropic's **evaluator-optimizer** pattern, "one LLM call generates a response while another provides evaluation and feedback in a loop," with their caveat to use it only "when there are clear evaluation criteria" and iteration adds real value. Two refinements from the multi-agent work:

- **Judge the end state, not the process.** "Instead of judging whether the agent followed a specific process, evaluate whether it achieved the correct final state," because agents reach good outcomes by different routes.
- **Use a rubric, and keep a human in the loop.** A single judging call scores accuracy, completeness, source quality, and efficiency, and a person reviews the edge cases. You can start this at about twenty real tasks. The excuse "we have no benchmark yet" does not hold.

When the stakes are high, make the evaluator adversarial: independent checkers whose job is to refute, accepting only what survives. "Find, then try to refute" is what separates a system you trust from one that merely sounds sure. Pair the machine gates with human gates at the moments that cannot be undone, before a plan runs, before anything is published. Automation should reach the irreversible step and stop for you.

> A caution the labs state directly: multi-agent systems have **emergent behavior**, where "small changes to the lead agent can unpredictably change how subagents behave." You cannot tune one worker in isolation. A change to the lead has to be re-checked against the whole team. That is why the gate and team-level evaluation are not optional polish.

---

## 8. The knowledge base

This is the harness's sharpest difference from ordinary AI use, and the one most people never reach. The agent keeps a durable knowledge base, and maintains it itself. Done well, it is what lets the agent compound instead of starting from zero every session. Four moves, each in the official vocabulary, each pulled back into a plain claim.

**It is context engineering, not prompt-stuffing.** Anthropic calls the discipline context engineering. What that buys you: the base delivers the smallest set of high-signal tokens it can, curated every turn, because context rots and attention is finite. Cramming long-term knowledge into the instruction file is the very thing the base exists to defeat.

**It loads just in time, and prefers reading over indexing.** The base stores "lightweight identifiers (file paths, stored queries, web links)," and the agent loads what it needs at runtime. On the perennial question of how to retrieve, the Claude Agent SDK guidance runs against fashion: semantic search over embeddings is "usually faster than agentic search, but less accurate, more difficult to maintain, and less transparent," so "start with agentic search, and only add semantic search if you need faster results." For a personal, evolving, local store, an agent that reads curated files on demand beats a precomputed vector index. Building the index first tunes for speed you do not need yet.

**It is memory, which is not the same as context.** Make this distinction before anything else. Context is the finite working set in the window, subject to rot. Memory is durable knowledge kept outside the window and pulled in when needed. The mechanism is structured note-taking, "the agent regularly writes notes persisted to memory outside of the context window," and, as a product, a **memory tool**: a small set of file operations that let an agent "store and retrieve information across conversations... build knowledge over time without keeping everything in the context window." Its operating instruction opens with the right posture: "ASSUME INTERRUPTION: your context window might be reset at any moment." Memory "persists important information across compaction boundaries so that nothing critical is lost." This is what lets a team survive a swapped worker, a reset window, and an aggressive summary. The knowledge lives at the team's level, outside any one worker's window, so it outlasts the worker that wrote it.

**The agent curates its own context, and gets sharper at recurring work.** Across sessions it accumulates "stable preferences, recurring workflows, conventions, and known pitfalls," and writes down what it learns from your corrections. Keep the index small (a short table of contents loads at the start, the detail loads on request), and only mark knowledge durable after it has been checked end to end.

One honesty hook holds the whole thing together. Keep authoritative, always-applies rules in the checked-in single source of truth, and treat the self-curated base as exactly what one runtime calls it: "a helpful local recall layer, not... the only source for rules that must always apply." That separation, an enforced source of truth on one side and a self-curated recall layer on the other, is what turns "agent-maintained knowledge base" from a slogan into an architecture you can defend.

A metaphor to hold it: **tree rings.** The base grows by adding, a faint ring each session. It thickens. You can read its history outward, asking what you thought about something a year ago. But rings only mean something if you curate the sources and ask the questions while the agent does the writing. You do not hand-edit the durable store yourself, because you do not hold all of its cross-references in your head and you will quietly break consistency. You point. It integrates.

---

## 9. Continuous improvement

A harness that cannot improve is a snapshot, and snapshots rot. Make capability a registry you can swap and grade, not a fixed set, and run two loops that keep it growing. One pulls capability in from outside. One grows it from inside. Neither is bespoke. Each is a named pattern.

### Capability as a registry

Treat each slot as a row: a slot, its current provider, where the provider came from, and a version. Because of progressive disclosure, "you can install many [capabilities] without context penalty," since only metadata is always loaded. That economics is what makes a large, swappable library viable rather than bloated. The model is part of the registry too. Capability is resolved at the moment of dispatch (the model and tools are bound per call), and mixing tiers is canonical and measured: Anthropic's setup of a strong model for the lead and a cheaper model for the workers "outperformed single-agent Claude Opus 4 by 90.2%." So a "capability slot" is a versioned registry of specialists whose model and tools are chosen at dispatch time.

### Loop one: never stuck

When the current registry cannot solve a problem, the system must not shrug.

```text
detect a gap  ->  search what you already have  ->  scout the wider world  ->
verify candidates  ->  decide (reuse / upgrade / build)  ->  integrate  ->  retry
```

Two disciplines keep it safe and good. The first is **reuse first**: exhaust the installed registry before importing or building, because most "missing" capability is already there and forgotten. The cost of having it is near zero. The cost is finding it. The second is to **gate the import, including for trust.** Anything you bring in passes conservative checks: is it redistributable, does it leak data, is it current, and is it pinned to a fixed version so an upstream change cannot break you silently? Then rename it into your convention and record where it came from. The trust gate is not optional. The security guidance for skills says to "treat installing [one] like installing software." A bad one can "direct [the model] to invoke tools or execute code in ways that don't match the [skill's] stated purpose," and one that fetches external URLs is riskier still, since fetched content can carry injected instructions. Reuse first, pin the version, gate for trust: that is the supply-chain hygiene of an agentic system.

Wire the loop into the router as a hard rule. "I can't do this" may only be said after the gap-escalation has run. Never stuck is a step, not a good intention.

### Loop two: self-improvement

Capability also grows from inside, and the mechanism is again the evaluator-optimizer, plus iteration driven by evaluation. The models can do the optimizing themselves: "given a prompt and a failure mode, [they] diagnose why the agent is failing and suggest improvements." In Anthropic's own system, an agent that rewrote tool descriptions produced "a 40% decrease in task completion time" for later agents. So the loop is: a worker records what it learned; that memory is periodically distilled into a sharper specification; and a pattern that passes the gate's rubric across enough real tasks is promoted, version-bumped in the registry or lifted into a shared capability. A pattern found in one corner becomes a capability everywhere.

### Upgrade and migration without losing the thread

This is the part worth being precise about, because it has a clean shape. **Upgrading a slot** is swapping its provider. Everything that routed to that slot now routes to the better one, with no caller changes. **Migrating live work** uses the coordinator plus durable memory: "summarize completed work phases and store essential information in external memory," then spawn fresh workers with clean windows that pick up from that memory. Because the state lives outside any worker's window, you can replace, upgrade, or re-point a worker mid-stream without dropping the thread, the same move that lets long-running systems do rolling upgrades with no downtime. Memory is what makes capability swappable while the system is running.

Said plainly, leads and team memory and capability slots are just the orchestrator-worker pattern, plus note-taking memory, plus a progressively-disclosed, model-swappable, evaluation-gated registry. Naming them this way is what lets you trust the system as it changes underneath you.

---

## 10. Enforcement and honesty

A harness will claim to enforce its contract. Be precise about what that means, or you will trust a guarantee you do not have. There are two mechanisms, and they are not equal.

The first is **code**, in the form of hooks. A hook runs at a fixed moment and cannot be talked out of it. This is real enforcement, and the docs are explicit that prose is not: instruction files are "context, not enforced configuration. To block an action regardless of what [the model] decides, use a PreToolUse hook." Use it for what must hold: protect credentials and signing keys, inject the standing rules at the start of a session, block the clearly destructive.

The second is **prose**, the contract itself. Most of the dispatch contract (name the model, one owner per file, return in the fixed shape) is, in practice, prose that agents are asked to honor. They honor it most of the time. Most of the time is not always.

Enforce in code what you can, and say plainly that the rest is an honor system. Some constraints are hard to enforce mechanically. Per-agent, per-file rules often exceed what a global hook can see, and would need a custom dispatcher to enforce. Name that limit out loud. A harness that knows where it is only prose is far safer than one that believes its prose is iron.

Then verify the harness against itself. The cheapest honest test: fan out a read-only self-audit (one helper checks structure, one inventories it, one tries to break the contract) and fix what they find. A system that can audit and repair its own drift is one you can keep. It is just the orchestrator-worker and evaluator-optimizer patterns pointed inward.

---

## 11. Model neutrality

The models converge, and the runtimes wrapping them converge too. Bind your harness to one product's conventions and you will rebuild it on every switch. So author the harness once, in a neutral form, and ship a thin adapter for each runtime. This is becoming a real standard with a real home: **AGENTS.md**, "a simple, open format for guiding coding agents... a README for agents," kept separate from the human README because "README.md files are for humans," and stewarded by the Agentic AI Foundation under the Linux Foundation, with nearest-file-wins precedence and explicit user prompts overriding. One runtime reads it directly. Another points its own instruction file at it. The adapter carries the runtime's dispatch verbs and nothing else. It carries no new philosophy.

Keep one identity, one profile, one router, and one knowledge base as the source of truth, and write instructions in plain action verbs ("dispatch a worker," "hand off by reference") that each runtime maps to its own primitives. One corollary kills harnesses quietly: a single source of truth means no duplication. The instant a fact lives in two places, they drift, and drift is silent. Prefer a pointer ("the roster lives here") over a copy, and you will spend your debugging on real problems instead of on two truths that quietly disagree.

---

## 12. What people overlook

The grounded shortlist. Each is a place builders reliably go wrong.

1. **No evaluation, so you grade by feel.** A quality gate without an evaluation loop is a vibe. Evaluation starts small: about twenty real tasks, a scoring rubric, a person on the edge cases. With effects this large, "no benchmark yet" is not an excuse.
2. **Reaching for an agent, or a framework, too early.** Simplest solution first. A single model call with retrieval is often more reliable and cheaper than autonomy. A team is an escalation, justified by measured need.
3. **Believing more context is better.** It is not. Recall decays as the window fills, and every token spends a finite attention budget. Stuffing the instruction file, pre-loading the whole base, or never compacting hurts reliability even below the limit.
4. **Confusing memory with context.** Context is the finite window. Memory is durable knowledge kept outside it and pulled in when needed. Long-term knowledge belongs in memory, not the prompt.
5. **Expecting written rules to be enforced.** Instruction files are context, not configuration. Only a hook blocks, and only on events that allow it. Know which guarantees you actually have.
6. **Treating multi-agent as a free upgrade.** It costs roughly fifteen times the tokens, and token spend explains most of the performance difference. It is the wrong tool for tightly-coupled work that needs shared context, like most coding. Collapse to one agent when the task wants shared context.
7. **Underinvesting in tool and capability design.** The labs spend more time on tools than on prompts. The top failure is overlapping, vaguely-named capabilities. Fix boundaries and names before adding helpers.
8. **Over-trimming the capability library.** Installing many skills is nearly free. Reference material can live on disk at no cost until it is read. Hoard capability, and load it lazily.
9. **Vague delegation.** A worker without an objective, a format, tools, and boundaries duplicates work and leaves gaps. Brief completely, or do not fan out.
10. **Tuning one worker in isolation.** Because of emergent coupling, a change to the lead ripples through the team. Re-check the whole team after any change to the coordination.
11. **Reaching for a vector index by reflex.** Reading curated sources on demand beats a precomputed embedding index for an evolving store. Add semantic search later, only for speed.
12. **Trusting imported capability.** A skill is software. A bad one can hijack tools or smuggle instructions through fetched URLs. So reuse before you import, pin what you bring in, and do not trust it until it is gated.

---

## 13. Operating principles

The compressed version. If the rest of this evaporated, keep these.

1. Invest in the harness, not the prompt. Structure compounds. Prompts do not.
2. An agent is a context window inside an execution environment. Decide what is in the window and who manages it.
3. Simplest solution first. Add autonomy only when measured need demands it.
4. Aim for the smallest set of high-signal tokens, not the most. Respect the attention budget. Expect context rot.
5. Memory is not context. Durable knowledge lives outside the window and loads on demand.
6. A lead routes, plans, synthesizes, gates, and decides when to stop. Workers self-direct in isolation and cannot recruit.
7. Brief completely, hand off by reference, name the model, one owner per file, return a distilled summary. The contract is what makes fan-out composable.
8. The gate is an evaluator. Judge the end state against a rubric, adversarially when it matters, before results flow on.
9. Right-size the fan-out, and collapse the team when the work wants shared context. Fifteen times the tokens is a bet, not a default.
10. Capability is a versioned, swappable, evaluation-gated registry. Upgrade by swapping a provider. Bind model and tools at dispatch.
11. Reuse first, then import (pinned and trust-gated), then build. A large library is nearly free if you load it lazily.
12. Never stuck. "I can't" only after the search for a better capability has run.
13. Promote what passes the gate. Lift proven patterns from one worker to the whole system.
14. Keep a knowledge base the agent maintains. Authoritative rules in the enforced source of truth, recall in the self-curated layer.
15. Enforce in code what you can. Be honest about what is only prose.
16. Stay model-neutral. Author once, adapt thinly. The model is a replaceable part.

---

## 14. Closing

A harness, done well, stops feeling like configuration and starts feeling like an extension of how you think. It knows who you are and sends your intent to the right specialist. When it is stuck it reaches for a better way, keeps what proves out, and curates its own knowledge so it grows sharper at the work you actually do. It stays free of any single model, so it outlives the one you built it on, and the next.

The tree-ring image is the right one to end on. You are not shipping a product. You are growing something. Each session leaves a faint mark. The registry deepens and the knowledge connects, and over time the contract tightens around how you actually work. One day you look at the cross-section and read, in its rings, the whole trace of how you came to think the way you do, held and continued by a system that is unmistakably yours.

Build the harness. Gate it honestly. Let it grow with you.

---

## 15. Glossary and sources

**Core terms**, as the labs use them: *augmented LLM*; *workflow* and *agent*; *prompt chaining* and *gates*; *routing*; *parallelization*; *orchestrator-workers*; *evaluator-optimizer*; *context engineering*; *attention budget*; *context rot*; *just-in-time retrieval*; *progressive disclosure*; *compaction*; *structured note-taking*; *sub-agent context isolation*; *right altitude*; *tools as a contract*; *namespacing*; *end-state evaluation*; *effort scaling*; *emergent behavior*; *enforced configuration (hooks)*; *agentic search versus semantic search*; *memory tool*; *AGENTS.md*; *manager pattern*.

**Primary sources** (the official engineering writing and docs behind this guide; read them directly):

- Anthropic, [*Building Effective Agents*](https://www.anthropic.com/engineering/building-effective-agents) (Schluntz and Zhang): workflows versus agents, and the building-block patterns.
- Anthropic, [*Effective Context Engineering for AI Agents*](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): context engineering, attention budget, context rot, just-in-time retrieval, progressive disclosure, compaction, structured note-taking, right altitude.
- Anthropic, [*How We Built Our Multi-Agent Research System*](https://www.anthropic.com/engineering/multi-agent-research-system): orchestrator-worker in practice, the token economics (roughly 4x and 15x, 80% of variance, 90.2%), effort scaling, end-state evaluation, emergent behavior, and the 40% result.
- Anthropic, [*Writing Effective Tools for Agents*](https://www.anthropic.com/engineering/writing-tools-for-agents): tools as a contract, namespacing, and tool overlap as the top failure.
- Anthropic, [*Building Agents with the Claude Agent SDK*](https://claude.com/blog/building-agents-with-the-claude-agent-sdk): sub-agent context isolation and distilled returns, agentic search versus semantic search.
- Anthropic, [*Agent Skills*](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) docs: progressive disclosure, installing many for free, and skills as a supply-chain risk.
- Anthropic, [*Memory tool*](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) and [*context editing*](https://platform.claude.com/docs/en/build-with-claude/context-editing) docs: persistent memory, the memory directory, "ASSUME INTERRUPTION," persistence across compaction.
- Claude Code, [*Memory*](https://code.claude.com/docs/en/memory), [*Hooks*](https://code.claude.com/docs/en/hooks), and [*Sub-agents*](https://code.claude.com/docs/en/sub-agents) docs: instruction-file cost and adherence, "context, not enforced configuration," PreToolUse and exit-code-2 blocking.
- OpenAI, [*A Practical Guide to Building Agents*](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) (PDF): when to build an agent, single agent versus manager pattern, guardrails.
- OpenAI Codex, [docs](https://developers.openai.com/codex) including [*Memories*](https://developers.openai.com/codex/memories) and the [*AGENTS.md* guide](https://developers.openai.com/codex/guides/agents-md): two-tier memory (a recall layer versus rules that must always apply), per-agent model and tool settings.
- [*AGENTS.md*](https://agents.md/) (Agentic AI Foundation, Linux Foundation): the open instruction-file format, nearest-file-wins, user prompts override.

*Quotations are exact. Figures (15x, 80%, 90.2%, 40%) come from the multi-agent research write-up. I name none of my own products here, and no third-party repositories, on purpose. The portable part is the philosophy and the precise vocabulary. Take what serves you, keep your source of truth singular, and let the rings thicken.*
