**English** · [한국어](./README.ko.md)

# The Agentic Harness

*On building a model-neutral agent harness that improves itself, on your own machine.*

> A good agent harness is not a pile of prompts or a roster of agents. It is a **compounding system**: a model-neutral scaffold with durable memory, capabilities you can evaluate and swap, and a way to upgrade itself without losing continuity. Build that, and the model inside becomes a replaceable part.

Two things matter more than the rest, and most people building with AI shortchange both. A harness should get better over time, its capabilities a registry you version and swap rather than a frozen prompt, so that finding a better way anywhere lifts the whole system at once. And it should keep a knowledge base it maintains for itself: durable memory, curated, loaded only when needed, kept apart from the working conversation. That is the line between an assistant that re-learns you every session and one that compounds.

None of this is only for people who write code. Anyone who wants to use AI well runs into the same walls.

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

---

## 1. Why a harness

Most people use an agent: open a chat, type, read. For one-off tasks that is enough. But the moment you want more, the ground shifts. You start wanting consistency across sessions, an agent that knows who you are, specialists instead of one overstretched generalist, sub-tasks that coordinate among themselves, a setup that improves as you go. At that point you have stopped using a model and started building a system around it. The harness is that system, and it is the part you own.

Start from restraint. The simplest thing that works usually wins, and a single well-aimed call with good retrieval beats an elaborate multi-agent setup on reliability and cost more often than people expect. So treat everything below as an escalation you earn, not a place to begin. None of what follows compounds because it is novel. It compounds because it gives you continuity, structure, and a kind of growth a bare model never manages on its own.

---

## 2. The one mental model

If you keep one idea from all of this, keep this one.

> **An agent is a context window running in an execution environment.**

The **context window** is the entire world the agent reasons over: its instructions, the tools within reach, the data it has pulled in, the conversation so far. Two "different agents" are really two different ways of filling a window, and something useful follows from that. A main agent, a sub-agent, and a separate independent agent are the same kind of thing; they differ only in who manages their context and how they were summoned, not in nature. So "should this be a sub-agent or a fresh session?" collapses into one real question: what should this window hold, and who manages it?

Underneath sits a single atom: a model given retrieval, tools, and memory, free to write its own queries, choose its own tools, and decide what to keep. That is the unit you compose with. Every helper in a harness is one of these, narrowed to a single job.

### Context is finite, and it decays

The other half of the model, the environment, is governed by something beginners miss: context is finite, and it gets worse as it fills. Two forces are at work here. There is a budget of attention, and every token you add spends from it, so more context buys you less and less. Then there is decay: as a window fills, the model's grip on any particular thing inside it loosens.

So the goal is never *more* context. What you are after is the smallest set of high-signal tokens that still gets the outcome you want. In practice your job is to decide what earns a place in the window, what stays outside, and what has to be remembered once the window is gone. Almost every choice further on, isolation, loading on demand, summarized hand-backs, a knowledge base that opens only when asked, is one answer to that single job.

Caching says the same thing in the language of money. The stable front of a context can be cached and reread for almost nothing, so keeping the unchanging parts small, identity, rules, catalogs, is both steadier and cheaper. Design so the only thing that churns is the task itself.

---

## 3. Workflows and agents

One line runs through the whole harness, and it is worth drawing carefully. A **workflow** is a system where the model and its tools run along fixed paths laid down in code; the model only fills in the steps. An **agent** is a system where the model directs its own process and decides which tools to use and when. The path is the model's to choose.

A good harness is deliberately both, which is the part most builders miss. Its spine, the router, the fan-out engine, the checks between steps, is a workflow: fixed paths with gates, small programmatic checks that confirm one step is sound before the next begins. Its leaves, the individual helpers, are agents that improvise inside their slot. You pin the orchestration in code precisely so each helper can improvise without danger. Knowing which parts are fixed and which are free is the difference between a system you can reason about and one you can only hope works.

The pieces you compose with are familiar by now: prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer. Three of them carry most of the weight here. Routing is the front half of a team lead, orchestrator-workers is the team itself, and evaluator-optimizer does double duty as the quality gate and the improvement loop.

---

## 4. Anatomy of a harness

A harness is a handful of surfaces, each with one job. Conflating them is the most common mistake. Keep them apart, and call each by its real name.

| Surface | Concept | Its one job |
|---|---|---|
| **Identity** | system prompt at the *right altitude* | who the agent is, how it speaks |
| **Your profile** | durable preference memory | who you are, how you work |
| **Router** | a *routing* workflow | classify intent, send it to the right specialist |
| **Rules** | system-prompt instructions | the hard limits on output and behavior |
| **Hooks** | *enforced configuration* | force a rule in code, not in prose |
| **Skills** | *progressive disclosure* | a capability loaded only when invoked |
| **Workers** | *sub-agents*, *context isolation* | scoped work in a clean window |
| **Knowledge base** | *structured note-taking*, *memory* | durable knowledge, loaded just in time |

Four of these are routinely misread, so they are worth slowing down on.

**Right altitude.** The system prompt lives between two failure modes: brittle hardcoded if-this-then-that logic on one side, and vague high-level hand-waving that assumes a shared context which isn't there on the other. Identity and rules sit in that band, and they are not free. A longer instruction file costs you twice: it eats context, and it lowers how reliably the model follows any single line in it. Imports do not rescue you, since an imported file still loads in full. Keep the rules layer lean, or it starts working against you.

**Rules versus hooks, the enforcement line.** This is the most consequential distinction in the whole harness. Instruction files and memory are context, not enforced configuration. A rule the model reads is a rule the model can decide to ignore. A hook is code that fires at a fixed moment and cannot be talked out of it. To actually block something, no matter what the model concludes, you need the hook, not the paragraph. Rules are the answer key. Hooks are what make the agent actually sit the exam.

**Skills and the economics of installing everything.** A skill describes a capability and when to use it, and it loads in layers: a sliver of metadata at all times, the body only when the skill is triggered, the deep reference only when something genuinely needs it. The effect is liberating. You can keep a large library of capabilities on hand for almost nothing, because reference material sits on disk costing you nothing until it is read. Trimming your capability library "to save context" optimizes the wrong thing.

**Workers and context isolation.** A worker is a sub-agent with a clean window. It can explore as widely as it needs, then hand back only a short, distilled summary to whoever called it. Two things come out of this: work can run in parallel, and the noisy, heavy parts stay walled off so they never pollute the caller's window. That is the whole reason a worker exists. It is not a new kind of agent, just a way to keep heavy, noisy work out of the caller's attention.

---

## 5. Context engineering

The discipline under all of it is context engineering: curating, every turn, the set of tokens the model should reason over. You are not writing one prompt and walking away. On every turn, for every agent, something decides what belongs in the window, and the harness is the machine that makes those decisions well.

The techniques you lean on have names worth knowing.

- **Just-in-time retrieval.** Rather than pre-loading data, an agent holds lightweight pointers, a file path, a saved query, a link, and pulls the actual data in at runtime, only when it needs it.
- **Progressive disclosure.** The agent finds what is relevant by exploring, and each step turns up the context that shapes the next.
- **Compaction.** Near the limit, summarize the conversation and start a fresh window from the summary. Carefully, though: compact too hard and you lose the quiet detail whose importance only surfaces later. More summarizing is not automatically better.
- **Structured note-taking.** The agent writes notes to durable memory outside the window and pulls them back when needed. This is the bridge to the knowledge base further on.
- **Sub-agent isolation.** Specialized workers take focused tasks into clean windows while the main agent keeps hold of the plan.

Underneath, these are one stance: treat the window as scarce and curated, and keep anything durable outside it.

---

## 6. Teams and the orchestrator-worker pattern

One generalist doing everything is a thousand-line function. Split the work by domain instead. For personal work that might be building, design, data, writing, research, knowledge, learning, planning, thinking, and a shared utility pool, though yours will differ. Give each domain a **lead**, and let leads coordinate workers. This is the orchestrator-worker shape: a coordinator on top, specialists underneath.

### The lead is a coordinator, not a doer

A lead reads the request, sets the strategy, spawns the workers it needs, then gathers their results and decides whether more work is required. So a lead is router, strategist, synthesizer, and gate at once. The move teams forget is the last one: the lead also owns the decision to stop. A lead plans and delegates. The instant it starts doing the work itself, it has stopped being a lead.

### Workers are specialists that cannot recruit

A worker does one thing in an isolated window with a scoped set of tools, and returns a distilled summary. The boundary that matters: a worker cannot spawn more workers. Enforce that in capability, not just in writing. If workers can spawn workers, coordination becomes a tree nobody controls. Workers report. Only leads coordinate.

### Slots, names, and a litmus test

Inside a team, define functional **slots**, one current provider each. This is what makes the system upgradable, the subject of section 9: to improve a capability, you swap its provider. But slots only stay legible when they are cleanly bounded, and the usual culprit is not the number of tools but their overlap. A team can run a large set of distinct, well-separated tools without trouble, and choke on a small handful that blur into each other. The test is simple: if a careful person can't say for certain which tool to use, the model won't either. Fix slot names and boundaries before you add another worker. Grouping related capabilities under a shared prefix is the standard fix.

### When to collapse the team

Now the counterweight, because this is easy to oversell. Multi-agent is not free, and for many tasks it is the wrong tool. A team burns far more tokens than a single agent, and most of the difference in how well these systems perform comes down to how much they spend. It earns that cost on heavy parallel work, on problems too big for one window, on juggling many complex tools. It does not earn it on tightly-coupled work that needs shared context and real-time coordination, which describes most coding. So bake effort-scaling into the lead, one worker for a quick lookup, the full team only for genuinely large work, and know when to fold the team back into a single agent. The advice that holds up is the same everywhere: start with one agent, and reach for a manager that calls specialists as tools only when one agent's instructions and tools stop fitting.

---

## 7. The dispatch contract

Fan-out without a contract is chaos. The value of a team is not "more agents." It is more agents under a contract that makes their output trustworthy and composable. The contract:

- **Brief completely.** Every delegation carries an objective, an output format, the tools or sources to use, and the boundaries. Skip these and workers duplicate each other, leave gaps, and fail to find what they needed. A vague delegation is the most reliable way to waste effort.
- **Hand off by reference, not by paste.** Pass a path or an identifier and let the worker load it. Pasting the whole brief bloats the window and loses the trail of where things came from.
- **Name the model at dispatch.** A worker inherits the session's model unless you say otherwise. Naming it per call is how you set the cost and accuracy you want for that one task. It is also the lever behind section 9.
- **One owner per file, parallelize only what is separate.** In parallel work, each file has exactly one writer. Never point two agents at the same file at once. Parallel is for work that is genuinely independent.
- **Return a distilled summary in a fixed shape.** Every worker hands back the same structure, what changed, what it did, errors, next steps, with a short status, and keeps it small. Many workers each returning a long report would eat the very context you split them off to protect. The point was to isolate the heavy work, not to make it free, so the size of the return is part of the contract.
- **Bound the depth, cap the count.** Workers are leaves. Coordination cost climbs fast as agents multiply. A few sharply-scoped workers beat a swarm of fuzzy ones.

### The fan-out engine

The router invokes an orchestrating routine when a task needs more than one worker or spans several phases:

```text
intake  ->  classify size  ->  write a delegation plan (to a file)  ->  dispatch  ->  gate  ->  report up
```

Two requirements that are easy to miss. The size step applies effort-scaling, so the lead does not reach for the whole team out of habit. And the plan is written to a file first, an analysis, the team with each worker's owned files, the order of work, because planning before spawning is what keeps a large fan-out coherent. A running ledger records what is done, so a run that is interrupted or compacted does not redo finished work.

### The gate is the point, and it is an evaluator-optimizer

The step most systems skip is the one that earns trust. When a worker returns, check its result against the brief before you accept it. This is the evaluator-optimizer in miniature: one pass produces the work, another judges it and feeds back, in a loop, and it is worth doing wherever the criteria are clear enough to judge against. Two refinements matter.

- **Judge the end state, not the process.** Don't grade whether the worker followed some prescribed path; grade whether it reached the right result. Good outcomes arrive by different routes.
- **Use a rubric, keep a person in the loop.** One judging pass scores the things that matter, accuracy, completeness, the quality of sources, efficiency, and a human looks at the edge cases. You can stand this up with a small set of real tasks. "We have no benchmark yet" is not a reason to skip it.

When the stakes are high, make the check adversarial: independent reviewers whose job is to refute, and you accept only what survives. Find it, then try to break it. That is what separates a result you trust from one that only sounds sure. Pair the automatic gates with human ones at the moments you cannot undo, before a plan runs, before anything ships. Automation should carry you right up to the irreversible step and then stop.

> One caution to keep in view. Multi-agent systems show emergent behavior: a small change to the lead can shift how the workers behave in ways you did not predict. You cannot tune one worker in isolation. Change the lead and you have to re-check the whole team. This is why the gate and team-level evaluation are not optional polish.

---

## 8. The knowledge base

This is the sharpest difference between a harness and ordinary AI use, and the one most people never reach. The agent keeps a durable knowledge base, and maintains it itself. Done well, it is what lets the agent compound instead of starting from zero every session. Four moves.

**It is context engineering, not prompt-stuffing.** The base is an exercise in curation: it hands over the smallest high-signal set it can, rebuilt every turn, because context decays and attention runs out. Cramming long-term knowledge into the instruction file is exactly the thing the base exists to replace.

**It loads just in time, and prefers reading over indexing.** The base stores lightweight pointers, and the agent loads what it needs at runtime. On the perennial question of how to retrieve, the fashionable answer, a semantic index over embeddings, is usually faster but less accurate, harder to maintain, and harder to see into. For a personal, evolving, local store, an agent that reads curated files on demand beats a precomputed vector index. Let the agent search and read first; add a semantic index later, and only if you actually need the speed. Building the index first tunes for a speed you do not need yet.

**It is memory, which is not the same as context.** Make this distinction before any other. Context is the finite working set in the window, and it rots. Memory is durable knowledge you keep outside the window and pull in when needed. The mechanism is plain note-taking: the agent writes things down to a store that lives outside the window, and reads them back later. The right posture is to assume interruption. The window can reset at any moment, so what matters has to survive that reset, across every compaction and summary. This is what lets a team outlast a swapped worker or a wiped window: the knowledge sits at the team's level, not inside any one worker, so it lives longer than the worker that wrote it.

**The agent curates its own context, and gets sharper at recurring work.** Across sessions it gathers the stable things, your preferences, the workflows that recur, the conventions, the traps you have already hit, and writes down what it learns from your corrections. Keep the index small, a short table of contents loading up front with the detail fetched on request, and mark knowledge durable only once it has been checked end to end.

One discipline holds the whole thing together. Keep the authoritative, always-applies rules in a single checked-in source of truth, and treat the self-curated base as what it is: a helpful recall layer, not the place where binding rules live. That separation, an enforced source of truth on one side and a self-curated recall layer on the other, is what turns "a knowledge base the agent maintains" from a slogan into something you can defend.

A metaphor to hold it: **tree rings.** The base grows by adding, a faint ring each session. It thickens. You can read its history outward, asking what you thought about something a year ago. But the rings only mean something if you curate the sources and ask the questions while the agent does the writing. You do not hand-edit the durable store yourself, because you do not hold all of its cross-references in your head, and you will quietly break consistency. You point. It integrates.

---

## 9. Continuous improvement

A harness that cannot improve is a snapshot, and snapshots rot. Make capability a registry you can swap and grade, not a fixed set, and run two loops to keep it growing: one pulls capability in from outside, the other grows it from within. Neither is exotic.

### Capability as a registry

Treat each slot as a row: the slot, its current provider, where that provider came from, and a version. Progressive disclosure is what makes this affordable, since only metadata is ever loaded by default, so a large and swappable library stays viable instead of turning into bloat. The model is part of the registry too. Capability is resolved at dispatch, model and tools bound per call, and mixing tiers is the normal thing to do: a strong model running the lead with cheaper models handling the worker tasks reliably beats putting one expensive model on everything. So a capability slot is really a versioned registry of specialists whose model and tools are chosen at the moment you dispatch.

### Loop one: never stuck

When the current registry cannot solve a problem, the system must not shrug.

```text
detect a gap  ->  search what you already have  ->  scout the wider world  ->
verify candidates  ->  decide (reuse / upgrade / build)  ->  integrate  ->  retry
```

Two disciplines keep it safe and good. The first is **reuse first**: exhaust the installed registry before importing or building, because most "missing" capability is already there and forgotten. Having it costs almost nothing; finding it is the real cost. The second is to **gate the import, trust included.** Anything you bring in passes conservative checks: is it redistributable, does it leak data, is it current, and is it pinned to a fixed version so an upstream change cannot break you silently? Then you rename it into your convention and record where it came from. The trust gate is not optional. Installing a skill is installing software. A bad one can steer the model into calling tools or running code in ways that have nothing to do with what it claimed to do, and one that reaches out to external URLs is worse, because whatever it fetches can carry instructions of its own. Reuse first, pin the version, gate for trust: that is the supply-chain hygiene of a system like this.

Wire the loop into the router as a hard rule. "I can't do this" may only be said after the gap-escalation has run. Never stuck is a step, not a good intention.

### Loop two: self-improvement

Capability also grows from the inside, and again the engine is evaluation-driven iteration. The model can do the optimizing itself: give it a prompt and the way it failed, and it will work out why and propose something better. Have an agent rewrite its own tool descriptions, and every agent that uses them afterward gets measurably faster. So the loop runs: a worker records what it learned, that memory is distilled now and then into a sharper specification, and any pattern that clears the gate across enough real work gets promoted, version-bumped in the registry or lifted into a shared capability. Something found in one corner becomes a capability everywhere.

### Upgrade and migration without losing the thread

This part is worth stating precisely, because the shape is clean. **Upgrading a slot** means swapping its provider; everything that routed to that slot now routes to the better one, and no caller changes. **Migrating live work** leans on the coordinator and durable memory: summarize the finished phases, push the essentials to external memory, then spawn fresh workers with clean windows that pick up from there. Because the state lives outside any worker's window, you can replace, upgrade, or re-point a worker mid-stream without dropping the thread, the same trick that lets long-running systems roll out upgrades with no downtime. Memory is what makes capability swappable while the thing is still running.

Leads, team memory, and capability slots are, in the end, just the orchestrator-worker pattern plus note-taking memory plus a registry you can disclose progressively, swap models in, and gate on evaluation. Naming the parts is what lets you keep trusting the whole as it shifts under you.

---

## 10. Enforcement and honesty

A harness will claim to enforce its contract. Be precise about what that claim is worth, or you will lean on a guarantee you do not actually have. There are two mechanisms, and they are not equal.

The first is code, in the form of hooks. A hook runs at a fixed point and cannot be argued with. This is real enforcement. Prose is not. Use hooks for the things that truly must hold: protecting credentials and signing keys, injecting the standing rules at the start of a session, blocking the obviously destructive.

The second is prose, which is most of the contract itself. Name the model, one owner per file, return in the fixed shape: most of this is, in practice, something you ask the agents to honor, and they honor it most of the time. Most of the time is not always.

Enforce in code what you can, and say out loud that the rest runs on honor. Some rules are genuinely hard to enforce by machine. Per-agent, per-file constraints usually sit beyond what a single global hook can see, and would need a custom dispatcher to hold. Name that limit rather than paper over it. A harness that knows where it is only prose is far safer than one that mistakes its prose for iron.

Then turn the harness on itself. The cheapest honest test is to fan out a read-only self-audit, one helper checking structure, one taking inventory, one trying to break the contract, and then fix what they surface. A system that can audit and repair its own drift is one you can keep. It is just the orchestrator-worker and evaluator-optimizer patterns, pointed inward.

---

## 11. Model neutrality

The models converge, and the runtimes wrapping them converge too. Tie your harness to one product's conventions and you will rebuild it at every switch. So write the harness once, in a neutral form, and ship a thin adapter per runtime. A shared convention is already forming for this: a plain instruction file, kept separate from the human-facing README, that any agent runtime can read, with the nearest file winning and an explicit user instruction overriding the file. One runtime reads it directly; another just points its own instruction file at it. The adapter carries the runtime's dispatch verbs and nothing else. No new philosophy lives in it.

Keep one identity, one profile, one router, and one knowledge base as the source of truth, and write instructions in plain action verbs, dispatch a worker, hand off by reference, that each runtime can map onto its own primitives. One corollary quietly kills harnesses: a single source of truth means no duplication. The instant a fact lives in two places, the two drift, and the drift is silent. Prefer a pointer, the roster lives here, over a copy, and you will spend your debugging on real problems instead of on two truths that disagree behind your back.

---

## 12. What people overlook

The shortlist. Each is a place builders reliably go wrong.

1. **No evaluation, so you grade by feel.** A quality gate with no evaluation behind it is just a vibe. Evaluation can start tiny: a small set of real tasks, a scoring rubric, a person on the edge cases. The effect is big enough that "no benchmark yet" does not hold up.
2. **Reaching for an agent, or a framework, too early.** Simplest thing first. A single call with retrieval is often more reliable and cheaper than autonomy. A team is an escalation you justify with measured need.
3. **Believing more context is better.** It is not. Recall fades as the window fills, and every token spends from a finite attention budget. Stuffing the instruction file, pre-loading the whole base, never compacting, all of it hurts reliability even before you hit the limit.
4. **Confusing memory with context.** Context is the finite window. Memory is durable knowledge kept outside it and pulled in on demand. Long-term knowledge belongs in memory, not the prompt.
5. **Expecting written rules to be enforced.** Instruction files are context, not configuration. Only a hook actually blocks. Know which guarantees you really have.
6. **Treating multi-agent as a free upgrade.** It costs far more tokens, and most of the performance gap is just that spend. It is the wrong tool for tightly-coupled work that needs shared context, like most coding. Fold back to one agent when the task wants shared context.
7. **Underinvesting in tool and capability design.** Tools deserve at least as much care as prompts. The usual failure is capabilities that overlap and are vaguely named. Fix boundaries and names before adding more helpers.
8. **Over-trimming the capability library.** Installing many skills is nearly free; reference material sits on disk at no cost until something reads it. Hoard capability, and load it lazily.
9. **Vague delegation.** A worker with no objective, format, tools, or boundaries duplicates work and leaves holes. Brief it completely, or do not fan out.
10. **Tuning one worker in isolation.** Because the coupling is emergent, a change to the lead ripples through the team. Re-check the whole team after any change to the coordination.
11. **Reaching for a vector index by reflex.** For an evolving store, reading curated sources on demand beats a precomputed embedding index. Add semantic search later, only for speed.
12. **Trusting imported capability.** A skill is software. A bad one can hijack tools or smuggle instructions in through fetched URLs. Reuse before you import, pin what you bring in, and do not trust it until it is gated.

---

## 13. Operating principles

The compressed version. If the rest of this evaporated, keep these.

1. Invest in the harness, not the prompt. Structure compounds. Prompts do not.
2. An agent is a context window inside an execution environment. Decide what is in the window and who manages it.
3. Simplest solution first. Add autonomy only when measured need demands it.
4. Aim for the smallest set of high-signal tokens, not the most. Respect the attention budget. Expect context to decay.
5. Memory is not context. Durable knowledge lives outside the window and loads on demand.
6. A lead routes, plans, synthesizes, gates, and decides when to stop. Workers self-direct in isolation and cannot recruit.
7. Brief completely, hand off by reference, name the model, one owner per file, return a distilled summary. The contract is what makes fan-out composable.
8. The gate is an evaluator. Judge the end state against a rubric, adversarially when it matters, before results flow on.
9. Right-size the fan-out, and fold the team back when the work wants shared context. A full team is a bet, not a default.
10. Capability is a versioned, swappable, evaluation-gated registry. Upgrade by swapping a provider. Bind model and tools at dispatch.
11. Reuse first, then import with the version pinned and trust gated, then build. A large library is nearly free if you load it lazily.
12. Never stuck. "I can't" only after the search for a better capability has run.
13. Promote what passes the gate. Lift proven patterns from one worker to the whole system.
14. Keep a knowledge base the agent maintains. Authoritative rules in the enforced source of truth, recall in the self-curated layer.
15. Enforce in code what you can. Be honest about what is only prose.
16. Stay model-neutral. Write once, adapt thinly. The model is a replaceable part.

---

## 14. Closing

A harness, done well, stops feeling like configuration and starts feeling like an extension of how you think. It knows who you are and sends your intent to the right specialist. When it is stuck it reaches for a better way, keeps what proves out, and curates its own knowledge so it grows sharper at the work you actually do. It stays free of any single model, so it outlives the one you built it on, and the next.

The tree-ring image is the right one to end on. You are not shipping a product. You are growing something. Each session leaves a faint mark. The registry deepens and the knowledge connects, and over time the contract tightens around how you actually work. One day you look at the cross-section and read, in its rings, the whole trace of how you came to think the way you do, held and continued by a system that is unmistakably yours.

Build the harness. Gate it honestly. Let it grow with you.
