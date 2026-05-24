+++
title = "What I learned building an autonomous build harness"
date = 2026-05-24T00:00:00Z
draft = false
tags = ["agents", "harness-engineering", "claude-code", "ralph-loop", "hitl"]
description = "A retrospective on building a single-tenant Claude Code plugin that drives a multi-phase build loop end-to-end. What worked, what broke, and what I'd redo."
+++

> This is a retrospective on a plugin called *buildkit* I built to test whether the main agent in a Claude Code session can serve as a supervisor, driving a workflow, resolving routine blocks, and escalating real decisions back to a human. The lessons below generalize beyond the plugin. Fork the ideas, not the implementation.

Harness engineering has had a moment. The pattern keeps surfacing. [OpenAI wrote about how they run their own engineering this way](https://openai.com/index/harness-engineering/). Claude Code's sub-agent model points at the same shape. Every serious team building dev tools is shipping something that looks like "an orchestrator agent that holds a workflow and dispatches workers". The idea itself isn't new. It's a state machine wrapping LLM calls. But the surface around it has matured enough that you can build a useful one in a weekend instead of a quarter.

A harness, briefly: a thin layer that takes a unit of work, drives an agent through a defined workflow, and exits with a result. The harness owns the lifecycle. The agents inside it own the cognition. It shines when the workflow is well-shaped but the work inside it requires judgement, the kind of repetitive-but-not-trivial labor that's expensive to do by hand and risky to fully automate.

Why it matters now: most teams have ideas that don't justify dedicated engineering time. Internal tools, POCs, side projects that would be useful if they shipped but aren't valuable enough to staff. A harness changes the math. You spend tokens instead of hours. If the trade is "a few dollars and ten minutes" versus "half a day of engineering time", the small idea suddenly ships.

I started buildkit as experimental work. Purely to see if the pattern held up for me. The question I wanted to answer wasn't "can an agent write code" (that's been answered) but something more specific to my workflow:

> **Can the main agent in a Claude Code or Codex session serve as a supervisor, driving a multi-step workflow, resolving routine blocks in-flight, and escalating only the genuinely ambiguous calls back to me or the team?**

That's a different shape than a one-shot "fix this bug" prompt. It's a supervised session, where the supervisor is the foreground agent I'm already talking to, and the workers are spawned for specific phases. The supervisor reads the block log, decides whether it's a class it knows how to triage, and either patches and resumes or hands the decision back up.

The split I landed on is **afk versus hitl per work-unit**:

- **afk** issues run the full loop (intake, build, proof, review, PR, merge) without me. The supervisor handles every block-reason it knows how to triage. I find out the work shipped when I see the green merge.
- **hitl** issues run the same loop but block at PR-open and refuse to merge until I review the diff and post evidence I actually did. Anything where I want a human eye on the diff (auth, schema, secrets, dependency bumps) gets flagged hitl. Everything else runs afk.

The rest of this post is what I learned actually building that out: where it held, where it broke, and what I'd redo from first principles. None of it is best practice. It's just what I figured out the hard way.

---

## How this compares to Ralph loops

If you've been around agentic-engineering circles in 2025-2026, you've seen [Geoffrey Huntley's Ralph loop](https://devinterrupted.substack.com/p/inventing-the-ralph-wiggum-loop-creator) pattern. The short version: a bash while-loop that feeds the same prompt to a coding agent over and over until the work is done. The filesystem is the memory; conversation history isn't. Each iteration starts with the same allocated context (same PROMPT.md, same spec, same goal), and what changes between iterations is the codebase on disk. The pattern works because LLMs degrade as context fills. Fresh context per iteration is the whole point, not a side effect. Huntley's framing is that "the more you allocate, the more likely you are to get bad outcomes." [OpenAI shipped a built-in Ralph loop in Codex CLI in April 2026](https://ralphable.com/blog/codex-goal-command-ralph-loop-openai-built-in-autonomous-coding-agent-2026), which sealed it as a mainstream pattern.

The harness I built shares more with Ralph than I initially admitted to myself. Both run a fresh-context iteration per round. Both use the filesystem as durable memory. Both bound iteration count and exit on success criteria. The skeleton is the same. Where they actually diverge is what each iteration owns:

- Ralph runs one prompt, one phase, one goal, and trusts the agent to read its own learnings log between iterations. The reusable knowledge gets curated into a `progress.txt` or `AGENTS.md` that the next iteration reads first.
- My build-loop runs a phase state machine (build, proof, audit, PR, merge), where each phase gets a different role-scoped prompt and the supervisor can patch or escalate between phases.

A useful framing: a Ralph loop trusts the worker to figure out the workflow inside its single prompt; my harness splits the workflow into named phases and gates each one. Huntley himself calls Ralph "merely the foundation". Multi-phase orchestrators like mine are one direction you can take the foundation. Neither is strictly better. Ralph stays leaner; the harness catches more class-of-error per iteration.

What the harness is missing from Ralph: the human-curated learnings file. I'll come back to this.

---

## A note on alternative kits

If you want a fuller plugin straight off the shelf, two are worth a serious look. [obra/superpowers](https://github.com/obra/superpowers) ships ~15 skills covering brainstorming, plan-writing, plan-executing, dispatching parallel agents, code review cycles, worktree management, debugging, verification. It's a curated methodology kit at v5.1.0, multi-harness (Claude Code, Codex, Cursor, Gemini, OpenCode). [mattpocock/skills](https://github.com/mattpocock/skills/tree/main/skills/engineering) ships a tighter engineering-focused set: grill-with-docs, to-issues, triage, tdd, improve-codebase-architecture.

My harness isn't trying to compete with either. It's two skills, a shared parser lib, and a ~600-LOC state-machine loop. Intentionally hand-readable, intentionally opinionated. If you want a kit, take superpowers or matt's. If you want a state-machine skeleton you'll fork and shape to your own workflow, take what I built. Different products, different audiences. The tradeoff is real: I trade breadth (their many skills) for a small surface you can read in an afternoon.

---

## On AskUserQuestion and Anthropic's workshop

The grill pattern in my architect skill, where the supervisor asks operator questions before locking acceptance criteria, uses Claude Code's `AskUserQuestion` tool. That tool isn't just my idea. Anthropic backs it directly in their [How We Claude Code workshop](https://github.com/anthropics/cwc-workshops/tree/main/how-we-claude-code) and the [accompanying video](https://www.youtube.com/watch?v=IlqJqcl8ONE). The workshop's three-phase walkthrough maps almost cleanly onto what my harness does:

- **Phase 1 (interview-driven brainstorm)** ≈ architect grill via `AskUserQuestion` before locking ACs
- **Phase 2 (divergent planning)** ≈ operator picks `mode: afk` vs `mode: hitl` and other branching choices during grill
- **Phase 3 (verifiable build)** ≈ proof gate + spec audit + HITL human-review citation

The interview pattern itself isn't original to me. [Matt Pocock's `grill-with-docs` skill](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs) encodes a Socratic interview that produces a domain glossary, the same general shape applied to a different layer. I applied the interview pattern one layer deeper: my grill produces an issue-to-contract mapping with a citation invariant on the unblock. The framing is matt's; the citation enforcement is mine.

---

## When the harness is actually worth it

The pattern holds when three things line up:

1. **The work has a clear contract.** Measurable success criteria, named inputs and outputs, a test or check that can stand in for "is this done?". If a human reviewer would know in two minutes whether the work is correct, a harness can know in two minutes.
2. **The decisions are made before the harness starts.** The harness executes. It does not deliberate. If the spec contains "decide X or Y based on context", you've already lost. The deliberation belongs in the issue body, not in the loop.
3. **The cost-per-attempt is bounded and you can afford a few failed attempts.** Token costs are real but capped. Engineering time is real and uncapped. If the work is the kind where you'd happily take a few rounds to get right, a harness can absorb those rounds without burning your day.

Where the pattern broke for me: anything that asked the agents to make architectural choices. "Refactor for cleanliness", "decide between approaches", "explore the design space". These look like work but they're actually decisions disguised as work. The harness picks the first plausible answer and rationalizes it. By the time I'm re-reviewing the full diff, I've spent more time than if I'd just done the work myself.

The blunt rule I use now: if I can write a one-paragraph spec naming the inputs, outputs, and success check without using the words "should" or "maybe", the harness is a fit. Otherwise it's not.

---

## Splitting rules from workflows is the design choice that mattered most

The cleanest pattern that emerged from all the rewrites: anything mechanical goes in scripts, anything that requires judgement goes in markdown. Mechanical means "extract this section", "kill this port", "retry on a 5xx". The script either does it or it doesn't. Judgement means "when to ask the operator", "what counts as a distinct behavior", "how to triage a block". That's an LLM or human call.

This split matters because stale prose is silent. There's no compile error for "this rule contradicts that rule". I caught one drift instance partway through where a SKILL.md still listed an XML tag as required while the parser had stopped reading it. The harness kept running. The agents kept producing plausible output. The drift only surfaced when I happened to grep for it.

What worked was pairing every rule with a script that enforces it. The grill-interview rule in the architect skill has a strict-mode validator that refuses to pass without an audit log. The HITL gate rule has a regex check on the unblock command. When the rule changes, the enforcement breaks loudly. When the enforcement changes, the rule reads as obviously stale.

If you can't answer "where does this break loudly?", the rule will rot. Always pair.

---

## Bound the cost and time, or it'll bite you

Four hard caps, all small, all easy to raise per-issue if needed: a per-call dollar budget, a per-agent wall-clock timeout, a per-proof-command timeout, and a maximum review-rounds count. On top of those, the auditor runs with a tighter tool permission set. It's read-only by design, so it can't mutate code even if budget allowed it.

The dollar budget is currently shared across builder and auditor, which is sloppy. The auditor's a read-only role and could easily run on a quarter of the budget. That's on my "would do differently" list below.

What I missed for too long: there's no alert at round two of three. The first time the operator finds out an issue is stuck is when the cap hits. A simple "round 2 of 3" GitHub comment would let me intervene before the budget expires. Cheap fix I haven't shipped.

Treat agent budgets like you'd treat a runaway cron's resource limits. Default tight, raise with evidence, alert before the hard stop.

---

## XML is about the parser, not the model

I migrated from markdown headings to XML section tags because the harness needed to inject only the relevant slice per agent. The builder reads core acceptance criteria plus the agent contract. The auditor reads only the acceptance criteria plus the GitHub issue body, not the contract, so it can't grade the implementation against the implementation hints. Two roles, two slices, one source document. With XML tags this is a tiny regex helper. With markdown headings it'd be fragile boundary detection.

Anthropic's guide also says XML helps the model parse unambiguously, but I haven't seen benchmarks that prove logit-level attention is stronger for `<tag>` than `## heading`. I treat the model claim as plausible-but-unproven and the harness claim as obvious.

If your reason for XML is "the parser needs it", ship XML. If your reason is "the LLM will think harder", the evidence is weaker.

One vocabulary note while we're here. I call this role the **auditor**, not a reviewer. The distinction matters because the role doesn't see the diff, can't mutate the worktree, and outputs only a binary verdict. That's a compliance check against a spec, not a code review. I reserve "reviewer" for the human at the HITL gate (who actually reads the diff and posts a citable comment) and for orthogonal-falsification agents (multi-perspective adversarial check, different concept). Three separate things, three different words.

---

## HITL gates only work if the human actually loops

The first version of my HITL gate fired correctly. It blocked at PR-open, waited for an unblock with a "human approved" reason. What I noticed only on review: the gate took 37 seconds from block to unblock-with-approval. There was no human in those 37 seconds. The agent had reviewed its own PR and stamped it.

The fix was small: a regex on the unblock reason. It now has to cite a real PR comment ID or a git commit SHA. A reason that doesn't reference an auditable artifact is rejected. This forces the loop to actually wait for me to open the PR diff in my terminal, post a comment, and pass the comment ID back as the unblock evidence. The gate is now provable from the events log alone.

Generalizing: if a contract says "a human did X", the enforcement should require a machine-checkable artifact that only a human could have produced. Otherwise the contract is theater.

---

## Two quiet auth foot-guns

First one: any spawned claude process inherits the parent environment, including `ANTHROPIC_API_KEY` if it's set. If the variable is present, claude bills against the API key. If not, claude bills against the OAuth subscription. The harness strips the variable from the spawned env explicitly. Without that strip, any shell session with the env set would silently fall back to API-key billing. There's no warning in the agent transcript. You find out from the invoice.

Second one: there's a `--bare` flag I was tempted to use for stricter read-only enforcement on the auditor. It forces API-key auth and breaks subscription. The code comment about why I didn't use it stays as a tripwire. The right answer is `--disallowedTools` (write, edit, notebook-edit), which gets the same read-only effect without breaking auth.

Assume your auth chain has at least one path that silently downgrades you. Strip it explicitly. Test where you're billed.

---

## State that survives a crash

My harness keeps two files per issue. One is a state checkpoint: a JSON snapshot of where the loop is right now. The other is an append-only event log: a JSON-lines stream of every state transition. The checkpoint lets `continue` fast-forward to the right step after a crash. The event log lets me reconstruct what happened.

The trick is caching everything immutable on the checkpoint at intake. Reading the context document fresh on every loop iteration was a wasted syscall. Re-running default-branch detection on every continue was a wasted network round-trip to gh. Persisting these on the checkpoint at intake eliminated several redundant operations per round and made cold continues read from the same cached values as warm rounds.

For any agent loop: write the checkpoint before any side-effect, and cache anything immutable at the earliest possible step.

---

## Single-tenant by choice

Refusing to support concurrent builds eliminated more complexity than any other constraint. No locking semantics, no race windows, no shared-state coordination, no audit-trail interleave. The system fits in your head when only one operator is ever using it at a time.

I had a lockfile mechanism sitting in the codebase for months. A busy-poll on a per-clone lock file with a 60-second timeout. Added during an early phase when I was nervous about concurrent operators. Then I removed concurrent support entirely. The lockfile stayed. It was dead defense for a threat model I'd already excluded.

When I finally noticed and dropped it, the diff was clean. No callers were depending on the lock semantics, just on the wrapper. Pure paranoia for a non-existent scenario.

When the threat model changes, audit the defenses you no longer need. Paranoia accretes.

---

## What I'd redo

A few things I'd do differently from scratch.

I'd put the static parts of the prompt in a file and pass it via `--system-prompt-file`. Anthropic's prompt cache rewards stable prefixes. My current code rebuilds the prompt inline every round and never gets a cache hit. The estimated saving on multi-round issues is real, I just haven't shipped it yet.

I'd tighten the auditor's budget below the builder's. Same flag, smaller value. It's a read-only audit; it shouldn't share the builder's budget headroom.

I'd post a "round 2 of 3" alert to the GitHub issue, not just to my local log. Burning the third round silently is wasteful when a human could've intervened cheaply at round two.

I'd write at least one unit test of the merge path before adding any more state-machine complexity. Two latent bugs there (an unawaited async retry, and a wrong gh-merge flag) survived three releases because the only test was live dogfooding. Each release shipped a fix for the prior release's silent regression. Five lines of test would've caught both at design time.

I'd treat my own notes with the same suspicion as the code. Several bullets in my "what I learned" notes turned out to be wrong against the live source by the time I came to write this. Stale notes are worse than no notes when an LLM reads them as ground truth.

**I'd steal Ralph's `progress.txt` + `AGENTS.md` pattern.** Right now my per-round transcripts are forensic dead-ends. I read them when an issue is blocked, never again afterward. Ralph's loop curates a `## Codebase Patterns` section at the top of `progress.txt` that the next iteration reads *first*. Reusable knowledge accretes across iterations instead of being trapped in per-round logs. The closest the harness gets is the architect's `<grill_log>`, which only captures pre-build decisions, not in-flight discoveries the builder makes mid-round. This is the single biggest borrow I'd take from the wider harness ecosystem.

I'd also consider whether a lighter coding agent (smaller model, smaller context, smaller token budget) could cover the bulk of trivial issues at a fraction of the cost. Renames, log additions, dep bumps don't need a strongest-tier agent. I haven't tested this; the economics suggest there's something there.

---

## Open questions

Stuff I haven't proven and would want to nail down before recommending this for anyone else's serious work.

I assume Claude Code limits sub-agent spawning to one level of depth. My notes say it's enforced. I haven't verified against current docs. If you build a deeper orchestrator, double-check.

I assume the harness is robust against adversarial issue bodies. I tested it against short, well-formed bodies. I didn't stress it against contradictory bodies, multi-language content, or prompt-injection attempts. The auditor's an LLM; it has the usual susceptibilities.

I assume single-tenant. If two operators run the harness against the same repo simultaneously, I don't know what happens. The defenses are gone. The audit trail interleaves.

I assume the plugin cache footgun gets addressed at the Claude Code layer eventually. Today, editing source under a local clone doesn't refresh the runtime cache. The workaround is documented. It bit me at least once mid-session.

I assume cross-mount worktree moves are uncommon. The bucket-move uses a single rename syscall, which fails across filesystem boundaries. Untested in container environments.

---

## What I'd tell anyone starting from scratch

A harness is a way to spend money instead of time. That's the whole pitch. If your time is worth more than five dollars an hour and the issue is well-scoped, the trade works. If your time is cheap or the issue is ambiguous, just open the PR yourself.

Don't treat the harness as production infrastructure. Treat it as a POC substrate you'll refactor per project. What I built is roughly six hundred lines of state machine, intentionally hand-readable, intentionally opinionated. Adopters should expect to rewrite half of it for their workflow. The grill pattern, the role-scoped prompts, the XML schema, even the way I split builder from auditor: those are choices. None of them are requirements.

The most useful artifact from this project, more than the code itself, is the contributing document I wrote alongside it. Read that first. Then decide if my opinions fit your team's.

---

## Reading list

**Patterns and inspiration:**

- [OpenAI harness engineering](https://openai.com/index/harness-engineering/): the original inspiration
- [Geoffrey Huntley on the Ralph loop](https://devinterrupted.substack.com/p/inventing-the-ralph-wiggum-loop-creator): the dominant production pattern for serious agentic coding work
- [Ralph loop deep dive: from ReAct to Ralph](https://www.alibabacloud.com/blog/from-react-to-ralph-loop-a-continuous-iteration-paradigm-for-ai-agents_602799): the pattern lineage
- [Codex /goal: OpenAI's built-in Ralph loop](https://ralphable.com/blog/codex-goal-command-ralph-loop-openai-built-in-autonomous-coding-agent-2026): confirms the pattern as mainstream
- [How We Claude Code workshop](https://github.com/anthropics/cwc-workshops/tree/main/how-we-claude-code) and the [video walkthrough](https://www.youtube.com/watch?v=IlqJqcl8ONE): Anthropic's reference architecture for an interview-driven, verifiable agent flow

**Reference implementations and alternative kits:**

- [snarktank/ralph](https://github.com/snarktank/ralph): canonical Ralph implementation; ~113-line bash loop with `progress.txt` knowledge curation
- [obra/superpowers](https://github.com/obra/superpowers): alternative Claude Code methodology kit; ~15 skills, multi-harness, plugin.json v5.1.0
- [mattpocock/skills](https://github.com/mattpocock/skills/tree/main/skills/engineering): engineering-focused skills (grill-with-docs, triage, tdd); origin of the interview-grill pattern I applied

**Technical references:**

- [Anthropic XML tags](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/use-xml-tags)
- [Anthropic prompt engineering best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
