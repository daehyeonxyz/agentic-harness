**English** · [한국어](./README.ko.md)

# The Agentic Harness

*On building an agent harness that improves itself, on your own machine, and stays tied to no single model.*

> A good agent harness is not a pile of prompts, nor a roster of agents. It is a system that grows as you use it: a skeleton tied to no single model, with durable memory, with capabilities you can evaluate and swap, and a way to repair itself without losing continuity. Build it, and the model inside becomes a part you can swap out anytime.

Two things matter more than everything else combined, and most people building with AI give both short shrift. First, a harness should get better over time. Don't nail capability down as a frozen prompt; keep it as a versioned list you can swap, so that when you find a better way somewhere, swapping in that one slot lifts the whole system at once. Second, a harness should keep a knowledge base it manages itself. It pulls durable memory in only when needed, and keeps it apart from the conversation you're having now. That is where an assistant that re-learns you from scratch every session parts ways with one that keeps building.

None of this is only for people who write code. Anyone who wants to use AI well hits the same walls.

---

## Contents

1. [Why a harness](#1-why-a-harness)
2. [If you keep one thing](#2-if-you-keep-one-thing)
3. [Workflows and agents](#3-workflows-and-agents)
4. [What a harness is made of](#4-what-a-harness-is-made-of)
5. [Context engineering](#5-context-engineering)
6. [Teams and the orchestrator-worker](#6-teams-and-the-orchestrator-worker)
7. [Rules for handing off work](#7-rules-for-handing-off-work)
8. [The knowledge base](#8-the-knowledge-base)
9. [A harness that improves itself](#9-a-harness-that-improves-itself)
10. [What you can enforce and what you can't](#10-what-you-can-enforce-and-what-you-cant)
11. [Not tied to one model](#11-not-tied-to-one-model)
12. [What people miss](#12-what-people-miss)
13. [Core principles](#13-core-principles)
14. [Closing](#14-closing)
15. [Glossary and sources](#15-glossary-and-sources)

---

## 1. Why a harness

Most people just use an agent. Open a chat, type a question, read the answer. For a one-off task that's enough. But the moment you want more, the story changes. Consistency that carries across sessions, an agent that knows who you are, specialists per domain instead of one overstretched generalist, sub-tasks that keep in step, a setup that gets better the more you use it. Once you're here, you're no longer using a model; you're building a system around it. That system is the harness, and it's the part you own.

Start from restraint. The simplest approach usually wins. A single well-aimed call with good retrieval beats an elaborate multi-agent setup on reliability and cost more often than you'd think. So read everything below not as a list to lay down from the start, but as steps you add one at a time as the need shows up. These things accumulate not because they're novel, but because they give you the continuity, structure, and growth that a model alone can't.

---

## 2. If you keep one thing

If you take away only one thing from this, take this.

> **An agent is a context window running inside an execution environment.**

The context window is the whole world an agent thinks with: its instructions, the tools in hand, the data it pulled in, the conversation so far. "Two different agents" is really two different ways the same window got filled. So a main agent, a sub-agent, and a separate standalone agent are the same thing at heart. What differs is only who manages that context and how it got summoned. Even "should this be a sub-agent or a fresh session?" comes down to one question: what should this window hold, and who manages it?

Underneath sits a single basic unit. A model holding retrieval, tools, and memory writes its own queries, picks its own tools, and decides what to keep. That's the unit you build with. Every worker in a harness is this one unit, narrowed to a single job.

### Context is finite, and it gets worse as it fills

The other half of the model, the environment, is governed by a fact beginners often miss: context is finite, and it gets worse as it fills. Two forces are at work. Attention has a fixed budget, so every token you add spends from it, and the value the context gives you shrinks along with it. On top of that, as the window fills, the model's grip on any one thing inside it loosens.

So the goal is never more context. The goal is the smallest, richest bundle of tokens that still gets you the result you want. In practice your job is to decide what goes into the window, what stays outside, and what has to be remembered after the window is gone. Almost every choice further on, isolating work, loading only on demand, summarizing before handing back, opening the knowledge base only when asked, is an answer to that one question.

Caching tells the same story in terms of cost. The parts of a context that rarely change can sit in cache and be reread for almost nothing. So keeping the slow-changing parts small, identity, rules, catalogs, is both steadier and cheaper. Design so that the only thing that changes each time is the task itself.

---

## 3. Workflows and agents

One line runs through the whole harness, and it's worth drawing with care. A workflow is a system where the model and its tools run along paths laid down in code; the model only fills in the blanks. An agent is a system where the model drives its own process, deciding which tools to use and when. Choosing the path is the model's job.

A good harness is deliberately both, and this is the part most people miss. The skeleton, the router, the fan-out engine, the checks between steps, is a workflow: fixed paths with gates, small checks that confirm one step is sound before the next. The leaves, the individual workers, are agents that move freely within their slot. You pin the larger flow down in code precisely so each worker can improvise without danger. Knowing which parts are fixed and which are free is what separates a system you can reason about from one you only hope works.

The pieces you build with are familiar by now. Prompt chaining, routing, parallelization, orchestrator-worker, and the evaluate-and-fix loop (evaluator-optimizer). Three of them carry almost all the weight. Routing is the front half of a team leader, orchestrator-worker is the team itself, and the evaluate-improve loop serves as both the quality gate and the improvement mechanism.

---

## 4. What a harness is made of

A harness splits into a handful of parts, each with one job. Lumping them together is the most common mistake. Keep them apart, and call each by its real name.

| Part | What it is | Its one job |
|---|---|---|
| **Identity** | a system prompt at the right altitude | who the agent is, how it speaks |
| **Your profile** | durable preference memory | who you are, how you work |
| **Router** | a routing workflow | classify intent, send it to the right specialist |
| **Rules** | system-prompt instructions | the clear limits on output and behavior |
| **Hooks** | configuration enforced in code | enforce a rule in code, not in prose |
| **Skills** | progressive disclosure | a capability loaded only when called |
| **Workers** | sub-agents, context isolation | scoped work in a clean window |
| **Knowledge base** | structured notes, memory | durable knowledge, loaded the moment it's needed |

Four of these are routinely misread, so let me take them one at a time.

**Right altitude.** The system prompt sits between two failures. On one side, rigid "if this, then that" logic packed in so tightly that the slightest deviation breaks it. On the other, vague high-level instruction that assumes a shared context which isn't there. Identity and rules live somewhere between the two, and they aren't free. A long instruction file costs you twice: it eats context, and it lowers how reliably the model follows any one line in it. Importing other files doesn't save you either, since an imported file still loads in full. Keep the rules layer light, or it starts working against you.

**Rules versus hooks, the line of enforcement.** This is the most important distinction in the whole harness. Instruction files and memory are context, not enforced configuration. A rule the model reads is a rule the model can decide not to follow. A hook is code that fires at a fixed moment and can't be talked out of it. To actually block something, whatever the model concludes, you need a hook, not a paragraph. If the rules are the answer key, the hook is what makes the agent actually sit the exam.

**Skills, and why you can install them all.** A skill writes down what a capability is and when to use it, and it loads in stages rather than all at once. A line of metadata is always present, the body loads only when that skill is called, and the deep reference only when something genuinely needs it. That's why you can hold a large library of capabilities for almost nothing. Reference material just lies on disk costing nothing until someone reads it. Trimming your library "to save context" fixes the wrong thing.

**Workers and context isolation.** A worker is a sub-agent with a clean window. It searches as widely as it needs, then hands back only a short, distilled summary to whoever called it. Two things follow. Work can run in parallel, and the noisy, heavy parts stay isolated so they don't pollute the caller's window. That's the only reason a worker exists. Not a new kind of agent, but a way to keep heavy, noisy work outside the caller's attention.

---

## 5. Context engineering

The work beneath all of this is context engineering: choosing, every turn, the bundle of tokens the model will look at. You don't write one prompt and walk away. Every turn, for every agent, something has to decide what goes into the window, and the harness is the machine that makes that decision well.

The techniques you'll lean on aren't hard to name.

- **Just-in-time retrieval.** Instead of loading all the data up front, the agent holds only light pointers (a file path, a saved query, a link) and pulls the data in when it actually needs it.
- **Progressive disclosure.** The agent searches for itself to find what's relevant, and each step turns up the context the next step will use.
- **Compaction.** Near the limit, summarize the conversation and open a fresh window from that summary. But be careful: cut too hard and you lose the small details that only matter much later. More cutting isn't always better.
- **Structured notes.** The agent writes to a store outside the window and reads it back when needed. This is the bridge to the knowledge base ahead.
- **Sub-agent isolation.** Specialist workers carry narrow jobs into clean windows while the main agent keeps hold of the overall plan.

Underneath, these five are the same stance. Treat the window as precious and fill it selectively; keep anything that must last outside it.

---

## 6. Teams and the orchestrator-worker

Handing everything to one generalist is a thousand-line function. Split the work by domain instead. For personal work that might be building, design, data, writing, research, knowledge, learning, planning, thinking, and a shared pool of utilities, though yours will differ. Give each domain a leader, and let the leader direct workers to get the work in line. This is the orchestrator-worker shape: one coordinator on top, specialists below.

### A leader doesn't do the work

A leader reads the request, sets the strategy, spins up the workers it needs, then gathers the results and decides whether more is needed. So routing, strategy, synthesis, and the final check all sit with one role. What teams often drop is the last piece: deciding when to stop is also the leader's job. A leader plans and delegates. The moment it puts its own hands on the work, it stops being a leader.

### A worker can't summon another worker

A worker does one job in an isolated window with a narrow set of tools, and hands back a distilled summary. One line must be drawn clearly: a worker can't spin up another worker. Enforce that with permissions, not prose. Once workers start calling workers, coordination spreads beyond anyone's control. Workers only report; only the leader coordinates.

### Keep slots cleanly divided

Define functional slots within a team, and give each slot one current owner of that job. This is what makes the system swappable (section 9), and lifting a capability means swapping that owner. But a slot stays clear only when its boundaries are clean, and the real cause of trouble isn't the number of tools but their overlap. Tools that are clearly distinct can run many at once without trouble, while tools that overlap fuzzily choke the team with just a few. The test is simple: if a careful person can't say for sure which tool to use, the model can't either. Before you add one more worker, fix the slot names and boundaries first. Grouping related capabilities under a shared prefix is the usual remedy.

### When to fold the team back

Here you have to look at the honest other side. Multi-agent isn't free, and for many jobs it's the wrong tool. A team burns far more tokens than a single agent, and most of the performance difference between these systems comes down to how much they spent. That cost earns its keep on heavy parallel work, on problems too big for one window, on juggling many complex tools. It doesn't earn its keep on tightly coupled work that needs shared context and real-time coordination. Most coding falls there. So build effort-scaling into the leader: one worker for a simple lookup, the full team only for genuinely large work. And know when to fold the team back into a single agent. The advice that holds everywhere is the same: start with one agent, and only reach for a manager that calls specialists as tools once that single agent's instructions and tools no longer fit.

---

## 7. Rules for handing off work

Fan-out without rules is chaos. A team's value isn't "more agents." It's many agents under rules that make their output trustworthy and joinable. The rules:

- **Brief it completely.** When you hand off work, include the objective, the output format, the tools or sources to use, and the boundaries. Leave these out and workers overlap, leave gaps, and fail to find what they needed. A vague handoff is the surest way to waste effort.
- **Point, don't paste.** Hand over a path or an identifier and let the worker load it. Paste the whole brief and the window swells, and you lose the trail of where things came from.
- **Name the model at handoff.** Unless you say otherwise, a worker inherits the session model. Naming the model per call is how you set the cost and accuracy that one task deserves. It's also the lever you'll use in section 9.
- **One owner per file; parallelize only what's separate.** When running in parallel, a file must have exactly one writer. Never point two agents at the same file at once. Parallel is for genuinely independent work.
- **Return a distilled summary in a fixed shape.** Every worker hands back the same frame (what changed, what it did, errors, next steps) with a short status, kept small. If each worker returns a long report, it eats back the very context you split them off to save. The point is to isolate heavy work, not to make it free, so the size of the return is part of the rules too.
- **Cap the depth, limit the count.** Workers are leaves. Coordination cost climbs fast as agents multiply. A few well-scoped workers beat a swarm of fuzzy ones.

### The fan-out engine

When a job can't be done by one worker or spans several stages, the router calls an orchestrating routine.

```text
intake  ->  gauge size  ->  write a delegation plan (to a file)  ->  hand off  ->  gate  ->  report up
```

Two things are easy to miss. The size step scales effort to the work, so the leader doesn't reach for the whole team out of habit. And the plan goes to a file first: the analysis, the team with each worker's owned files, and the order of work. Planning before you spin anything up is what keeps a large fan-out from coming apart. Keep a separate progress log, and a run that gets interrupted or compacted won't redo work it already finished.

### Check before you accept

The step most systems skip is exactly the step that earns trust. When a worker returns, check its result against the original brief before you accept it. This is a small evaluate-improve loop: one pass produces the work, another judges it and feeds back. Wherever the criteria are clear enough to judge against, it's worth doing. Two refinements matter.

- **Judge the result, not the process.** Don't grade whether the worker walked a prescribed path; grade whether it reached the right result. Good results arrive by different routes.
- **Use a rubric, and keep a seat for a human.** One judging pass scores the essentials, accuracy, completeness, quality of sources, efficiency, and a human looks at the tricky cases. A handful of real tasks is enough to start. "We don't have criteria yet" is no excuse to skip it.

When the stakes are high, make the check adversarial. Put independent reviewers whose job is to refute, and accept only what survives. Find it first, then try to break it. That's what separates a result you can trust from one that merely sounds confident. Pair automatic checks with human ones at the moments you can't undo, before a plan runs, before anything goes out. Automation should carry you right up to the irreversible step, then stop.

> One warning to keep in mind. Multi-agent systems throw up unexpected behavior. Change the leader a little and the workers can swing in directions you didn't predict. You can't tune one worker in isolation. Change the leader and you have to re-check the whole team. That's why the gate and team-level checks aren't optional polish.

---

## 8. The knowledge base

This is the sharpest difference between a harness and ordinary AI use, and the place most people never reach. The agent holds a durable knowledge base and manages it itself. Done well, the agent keeps accumulating instead of starting from zero every session. Four points.

**It's curation, not stuffing the prompt.** The base is an act of selection: each turn it remakes and serves the smallest, richest bundle it can. Because context blurs and attention runs dry. Cramming long-term knowledge into the instruction file is exactly the thing this base exists to replace.

**Load on demand, and prefer reading over indexing.** The base stores light pointers, and the agent loads what it needs as it goes. On the old question of how to search, the fashionable answer, a semantic index over embeddings, is usually faster but less accurate, more work to maintain, and harder to see into. For a personal, ever-growing store that lives on your own machine, an agent reading curated files on demand beats a precomputed vector index. Let the agent search and read first; add a semantic index later, only when you really need the speed. Building the index first tunes for a speed you don't need yet.

**This is memory, not context.** Draw this distinction before anything else. Context is the finite working bundle inside the window, and it blurs. Memory is durable knowledge kept outside the window and pulled in when needed. The method is plain note-taking: the agent writes to a store outside the window and reads it back later. The right posture is to assume interruption at any time. The window can reset whenever, so what matters has to survive that reset, and every compaction and summary. This is what lets a team weather a swapped worker or an emptied window. Knowledge lives at the team level, not inside any one worker, so it outlasts the worker that wrote it.

**The agent curates its own context, and sharpens on recurring work.** Across sessions it gathers the stable things, your preferences, the workflows that recur, the conventions, the landmines you've already stepped on, and writes down what it learns from your corrections. Keep the index small (a short table of contents up front, details pulled in on request), and mark knowledge as durable only after it's been checked end to end.

One principle ties the whole thing together. Keep the authoritative, always-applies rules in a single source of truth under version control, and treat the self-curated base for what it is. It's a helpful layer of recall, not where binding rules live. An enforced source on one side, a self-curated recall layer on the other. That separation is what turns "a knowledge base the agent manages" from a slogan into something that actually holds up.

One last thing. Don't hand-edit the durable store yourself. You can't hold all its cross-references in your head, so fix one spot and consistency quietly breaks somewhere else. You only point at what to keep; leave the tidying to the agent. As sessions stack up, the knowledge thickens, and you can look back at how you saw the same problem a year ago.

---

## 9. A harness that improves itself

A harness that can't improve is a snapshot, and snapshots rot. Make capability a list you can swap and score rather than a fixed bundle, and run two loops to grow it. One pulls capability in from outside; the other grows it from within. Neither is exotic.

### Treat capability as a list

Read a slot as one row: the slot, its current owner, where that owner came from, and a version. Progressive disclosure is what makes this manageable. Normally only metadata loads, so a large, swappable bundle keeps rolling instead of bloating to a halt. The model is part of this list too. Capability is set at the moment of handoff (model and tools bound per call), and mixing tiers is the norm. Putting a strong model on the leader and cheaper models on the worker tasks reliably beats running one expensive model on everything. So a "capability slot" is really a versioned list of specialists whose model and tools get picked at handoff time.

### The loop that pulls from outside

When the list you have can't solve a problem, the system mustn't just shrug.

```text
notice the gap  ->  search what you already have  ->  scout the outside world  ->
verify candidates  ->  decide (reuse / upgrade / build)  ->  integrate  ->  retry
```

Two principles keep this safe and good. The first is reuse first. Before importing or building, search the list you've already installed. Most "missing" capability is already there, just forgotten. Having it costs almost nothing; the real cost is finding it. The second is to gate imports, trust included. Everything you bring in passes conservative checks: is it redistributable, does it leak data, is it current, is it pinned to a fixed version so an upstream change can't break it silently? Then you rename it to your convention and record where it came from. The trust check is not optional. Installing a skill is installing software. A bad one can steer the model into calling tools or running code in directions that have nothing to do with what it claimed, and one that fetches outside URLs is worse, since what it fetches can carry instructions of its own. Reuse first, pin the version, check for trust. That's the supply-chain hygiene of a system like this.

Wire this loop into the router as a hard rule. "I can't do this" can only be said after the gap-filling procedure has fully run. Never-stuck isn't an attitude; it's a procedure.

### The loop that grows from within

Capability grows from the inside too, and again the engine is evaluation-driven iteration. The model can refine itself. Give it a prompt and how it failed, and it pinpoints why and offers something better. Have an agent rewrite its own tool descriptions, and every agent that uses those tools afterward gets noticeably faster. So the loop runs: a worker writes down what it learned, that memory is now and then distilled into a sharper spec, and a pattern that passes the check across enough real work moves up a level. Either its version rises in the list, or it gets promoted into a shared capability. Something found in one corner becomes a capability used everywhere.

### Swapping without breaking the flow

This part is worth stating precisely, because the shape is clean. Upgrading a slot means swapping its owner. Everything that routed to that slot now goes to the better one, with not a single line changed on the caller's side. Migrating live work leans on the coordinator and durable memory: summarize the finished stages, push the essentials to external memory, then spin up fresh workers with clean windows to pick up from there. Because the state lives outside any worker's window, you can swap, upgrade, or redirect a worker mid-stream without breaking the flow. It's the same move by which long-running systems roll out upgrades with no downtime. Memory is what lets you swap capability while the thing is still running.

Leaders, team memory, and capability slots are, when you get down to it, the orchestrator-worker pattern plus note-taking memory plus a list that discloses in stages, swaps models in, and filters on evaluation. Naming the parts is what lets you keep trusting the whole as it shifts underfoot.

---

## 10. What you can enforce and what you can't

A harness will claim to enforce its own rules. If you don't get clear on how far that claim is true, you'll lean on a guarantee you don't have. There are two mechanisms, and they aren't equal.

The first is code, in the form of hooks. A hook runs at a fixed point and can't be argued with. This is real enforcement. Prose is not. Use hooks for the things that must hold: protecting credentials and signing keys, injecting the standing rules at session start, blocking the plainly dangerous.

The second is prose, which is most of the rules themselves. Name the model, one owner per file, return in the fixed shape. Most of this is, in fact, prose asking the agents to comply, and they usually do. Usually isn't always.

Enforce in code what you can, and say plainly that the rest runs on trust. Some rules really are hard to enforce by machine. Per-agent, per-file constraints often exceed what a single global hook can see, and enforcing them would need a custom dispatcher. Don't paper over that limit; name it plainly. A harness that knows how far it's only prose is far safer than one that mistakes its prose for iron.

Then turn the harness on itself. The cheapest honest check is to fan out a read-only self-audit. One worker looks at the structure, one takes inventory, one tries to break the rules, and you fix what they surface. A system that can audit and repair its own drift is one you keep for the long haul. It's just the orchestrator-worker and the evaluate-improve loop, pointed inward.

---

## 11. Not tied to one model

The models grow alike, and the runtimes wrapping them grow alike too. Tie your harness to one product's conventions and you'll rebuild it every time you switch. So write the harness once in a neutral form, and attach a thin adapter per runtime. A common convention for this is already taking shape: a plain instruction file, kept apart from the human-facing README, that any runtime can read. The nearest file wins, and an explicit user instruction overrides the file. One runtime reads that file directly; another points its own instruction file at it. The adapter carries only that runtime's commands, and no new philosophy.

Keep one identity, one profile, one router, one knowledge base as the single source of truth, and write instructions in plain action verbs. "Hand off a worker," "pass by reference," words each runtime can map onto its own primitives. One thing quietly kills a harness: a single source of truth means no duplication. The instant a fact lives in two places, the two diverge, and the divergence happens silently. Prefer a pointer over a copy. Point and say "the roster lives here," and you'll spend your time on real problems instead of chasing two truths that disagree.

---

## 12. What people miss

The places builders reliably go wrong.

1. **No evaluation, so you grade by feel.** A quality gate with no evaluation behind it is just a vibe. Evaluation can start tiny: a few real tasks, a scoring rubric, one person on the tricky cases. The effect is big enough that "we don't have criteria yet" doesn't hold up.
2. **Reaching for an agent or a framework too early.** Simplest thing first. A single call with retrieval is often more reliable and cheaper than autonomy. A team is a next step you justify once the need is confirmed.
3. **Believing more context is better.** It isn't. As the window fills, recall blurs, and every token spends from a fixed attention budget. Stuffing the instruction file, preloading the whole base, never compacting, all of it erodes reliability before you even hit the limit.
4. **Confusing memory with context.** Context is the finite window. Memory is durable knowledge kept outside it and pulled in when needed. Long-term knowledge belongs in memory, not the prompt.
5. **Expecting written rules to be enforced.** Instruction files are context, not configuration. Only a hook actually blocks. Know what guarantee you really have.
6. **Treating multi-agent as a free upgrade.** It uses far more tokens, and most of the performance gap is just that spend. For tightly coupled work that needs shared context, like most coding, it's the wrong tool. When the work wants shared context, fold back to a single agent.
7. **Neglecting tool and capability design.** Tools deserve at least as much care as prompts. The common failure is capabilities that overlap and are vaguely named. Fix boundaries and names before adding more workers.
8. **Over-trimming the capability library.** Installing many skills is nearly free. Reference material can sit on disk at no cost until it's read. Stock capability generously, and load it only when needed.
9. **Vague handoffs.** A worker with no objective, format, tools, or boundaries overlaps and leaves gaps. Brief it completely, or don't fan out at all.
10. **Tuning one worker in isolation.** Because the coupling is hard to predict, changing the leader ripples through the whole team. Re-check the whole team every time you touch the coordination.
11. **Reaching for a vector index by reflex.** For an ever-growing store, reading curated sources on demand beats a precomputed embedding index. Add semantic search later, only when you need the speed.
12. **Trusting imported capability.** A skill is software. A bad one can hijack tools or smuggle in instructions through a fetched URL. Reuse before importing, pin what you bring in, and don't trust it until it's been checked.

---

## 13. Core principles

If everything else evaporates, keep these.

1. Invest in the harness, not the prompt. Structure compounds as it stacks; prompts don't.
2. An agent is a context window inside an execution environment. Decide what goes in the window and who manages it.
3. Simplest solution first. Add autonomy only when a confirmed need demands it.
4. Aim for the smallest, richest bundle of tokens, not the most. Spend attention sparingly, and expect context to blur.
5. Memory is not context. Durable knowledge lives outside the window and loads when needed.
6. A leader routes, plans, synthesizes, checks, and decides when to stop. Workers run isolated and self-directed, and can't summon another worker.
7. Brief completely, pass by reference, name the model, one owner per file, return a distilled summary. These rules are what make fan-out joinable.
8. The gate is evaluation. Judge the result against a rubric, adversarially when it matters, before results flow on.
9. Match fan-out size to the work, and fold the team back when the work wants shared context. Calling the whole team is a bet, not a default.
10. Capability is a versioned, swappable, evaluation-filtered list. Upgrade by swapping the owner. Bind model and tools at handoff.
11. Reuse first, then import with the version pinned and trust checked, then build. Load on demand and a large library is nearly free.
12. Never stuck. "Can't do it" only after searching for a better capability.
13. Promote what passes the check. Move a proven pattern from one worker to the whole system.
14. Keep a knowledge base the agent manages. Authoritative rules in the enforced source, recall in the self-curated layer.
15. Enforce in code what you can. Be honest about what is only prose.
16. Stay untied to any one model. Write once, adapt thinly. The model is a part you can swap.

---

## 14. Closing

A harness, done well, stops feeling like configuration and starts feeling like an extension of how you think. It knows who you are and sends your intent to the right specialist. When it's stuck it goes looking for a better way, keeps what worked, and manages its own knowledge, sharpening on the work you actually do. Tied to no single model, it outlives the one you built it on, and the one after that.

If I reach for one image, tree rings fit. You're not shipping a product; you're growing something. Each session lays down a faint ring, the list deepens and the knowledge connects, and over time all of it tightens around how you actually work. One day you look at the cross-section, and the whole trace of how you came to think the way you do is there, held and carried by a system that is unmistakably yours.

Build the harness. Check it honestly. Let it grow with you.

---

## 15. Glossary and sources

**Core terms** used in this piece: *augmented LLM*; *workflow* and *agent*; *prompt chaining* and *gates*; *routing*; *parallelization*; *orchestrator-workers*; *evaluator-optimizer*; *context engineering*; *attention budget*; *context rot*; *just-in-time retrieval*; *progressive disclosure*; *compaction*; *structured note-taking*; *sub-agent context isolation*; *right altitude*; *tools as a contract*; *namespacing*; *end-state evaluation*; *effort scaling*; *emergent behavior*; *enforced configuration (hooks)*; *agentic search versus semantic search*; *memory tool*; *AGENTS.md*; *manager pattern*.

**Primary sources** (the official engineering writing and docs behind the ideas here; worth reading directly):

- Anthropic, [*Building Effective Agents*](https://www.anthropic.com/engineering/building-effective-agents) (Schluntz and Zhang): workflows versus agents, and the building-block patterns.
- Anthropic, [*Effective Context Engineering for AI Agents*](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents): context engineering, attention budget, context rot, just-in-time retrieval, progressive disclosure, compaction, structured note-taking, right altitude.
- Anthropic, [*How We Built Our Multi-Agent Research System*](https://www.anthropic.com/engineering/multi-agent-research-system): orchestrator-worker in practice, the token economics, effort scaling, end-state evaluation, emergent behavior.
- Anthropic, [*Writing Effective Tools for Agents*](https://www.anthropic.com/engineering/writing-tools-for-agents): tools as a contract, namespacing, and tool overlap as the most common failure.
- Anthropic, [*Building Agents with the Claude Agent SDK*](https://claude.com/blog/building-agents-with-the-claude-agent-sdk): sub-agent context isolation and distilled returns, agentic search versus semantic search.
- Anthropic, [*Agent Skills*](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) docs: progressive disclosure, installing many for almost nothing, and skills as a supply-chain risk.
- Anthropic, [*Memory tool*](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool) and [*context editing*](https://platform.claude.com/docs/en/build-with-claude/context-editing) docs: persistent memory, the memory directory, assume-interruption, persistence across compaction.
- Claude Code, [*Memory*](https://code.claude.com/docs/en/memory), [*Hooks*](https://code.claude.com/docs/en/hooks), and [*Sub-agents*](https://code.claude.com/docs/en/sub-agents) docs: instruction-file cost and adherence, "context, not enforced configuration," and PreToolUse blocking.
- OpenAI, [*A Practical Guide to Building Agents*](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) (PDF): when to build an agent, single agent versus the manager pattern, guardrails.
- OpenAI Codex, [docs](https://developers.openai.com/codex) including [*Memories*](https://developers.openai.com/codex/memories) and the [*AGENTS.md* guide](https://developers.openai.com/codex/guides/agents-md): layered memory (a recall layer versus rules that must always apply), per-agent model and tool settings.
- [*AGENTS.md*](https://agents.md/) (Agentic AI Foundation, Linux Foundation): the open instruction-file format, nearest-file-wins, user instruction overrides.

*The vocabulary and concepts here are borrowed from the work above; the way they're woven together, and the harness itself, are my own. I deliberately name none of my own products here, and no third-party repositories. Take what serves you, keep your source of truth singular, and let it build up slowly.*
