+++
title = "What I learned from building an agent harness"
date = 2026-05-24T00:00:00Z
draft = false
tags = ["agents", "harness-engineering", "claude-code", "ralph-loop", "hitl"]
description = "A retrospective on building a single-tenant Claude Code plugin that drives a multi-phase build loop end-to-end. What worked, what broke, what I'd redo."
+++

> Retrospective on a Claude Code plugin I built called *buildkit*. The point: test whether the main agent in a session can act as a supervisor. Drive a workflow, resolve routine blocks, escalate the real decisions. The lessons generalize beyond the plugin. Fork the idea, not the code.

A harness is the thin layer outside the model: the environment, constraints, and feedback loops that let an agent do useful work without a human watching every token. [OpenAI calls this harness engineering](https://openai.com/index/harness-engineering/) and runs Codex on it. [Claude Code's sub-agent model](https://docs.claude.com/en/docs/claude-code/sub-agents) lands in the same neighborhood. The shape isn't new (it's an orchestration loop around model calls), but the tooling matured enough this year that you can build a useful one in a weekend.

A harness owns the lifecycle. The agents inside own the cognition. It pays off for tedious-but-non-trivial labor that's expensive to do by hand and risky to fully automate.

Most teams have ideas that don't justify engineering time: internal tools, POCs, side projects. A harness changes the math: tokens, not hours. A few dollars and ten minutes versus half a day, the small idea ships.

The question I wanted to answer:

> **Can the main agent in a Claude Code or Codex session serve as a supervisor: driving a multi-step workflow, resolving routine blocks in-flight, escalating only the genuinely ambiguous calls back to me or the team?**

Not a one-shot prompt. A supervised session. Foreground agent supervises, workers spawn per phase. Supervisor reads block logs, patches and resumes, or escalates.

The split I landed on is **afk versus hitl per work-unit**:

- **afk** issues run the full loop (intake, build, proof, review, PR, merge) without me. I find out the work shipped when I see the merge.
- **hitl** issues block at PR-open. They don't merge until I review the diff and post evidence. Auth, schema, secrets, dep bumps get flagged hitl. Everything else runs afk.

Below: where it held, where it broke, what I'd redo. Not best practice. Just what I figured out.

## When the harness is worth it

Three conditions:

1. **Clear contract.** Measurable success, named I/O, a test that means "done". If a reviewer would know in two minutes, the harness can too.
2. **Decisions made before the harness starts.** It executes, doesn't deliberate. "Decide X or Y based on context" already loses.
3. **Bounded cost, tolerant of failed attempts.** Token costs are capped, engineering time isn't.

It broke on architectural choices: "refactor for cleanliness", "decide between approaches", "explore the design space". Decisions disguised as work. Harness picks the first plausible answer and rationalizes it.

The rule I use: if I can write a one-paragraph spec naming inputs, outputs, and success check without "should" or "maybe", the harness fits. Otherwise no.

## How this compares to Ralph loops

[Geoffrey Huntley's Ralph loop](https://devinterrupted.substack.com/p/inventing-the-ralph-wiggum-loop-creator): a bash while-loop that feeds the same prompt to a coding agent until done. Filesystem is the memory; conversation history isn't. Same context every iteration, different codebase on disk. The pattern works because LLMs degrade as context fills past a certain threshold (Huntley's framing: minimize allocation to avoid the "Dumb Zone" of 60-70% context fill). [OpenAI shipped a built-in Ralph loop in Codex CLI in April 2026](https://ralphable.com/blog/codex-goal-command-ralph-loop-openai-built-in-autonomous-coding-agent-2026), making the pattern mainstream.

My harness shares more with Ralph than I first admitted: fresh-context iterations, filesystem memory, bounded iteration count. Identical skeleton. Where they diverge:

- Ralph runs one prompt and trusts the agent to read its own learnings log between iterations. Knowledge gets curated into a `progress.txt` or `AGENTS.md`.
- My build-loop runs a phase state machine (build, proof, audit, PR, merge). Each phase gets a role-scoped prompt. The supervisor patches or escalates between phases.

Ralph trusts the worker; my harness gates phases. Huntley calls Ralph "merely the foundation"; multi-phase orchestrators are one direction. Ralph stays leaner; the harness catches more error classes per iteration.

What my harness is missing from Ralph: the human-curated learnings file. More below.

## Alternative kits

If you want something off the shelf, two are worth a look. [obra/superpowers](https://github.com/obra/superpowers) ships ~15 skills (brainstorming, planning, parallel agent dispatch, code review, worktree management, debugging, verification). Curated, v5.1.0, multi-harness. [mattpocock/skills](https://github.com/mattpocock/skills/tree/main/skills/engineering) ships tighter engineering skills (grill-with-docs, to-issues, triage, tdd, improve-codebase-architecture).

My harness isn't competing with either. Two skills, a shared parser, ~600-LOC state-machine core (~830 LOC total). Hand-readable, opinionated. Take superpowers or matt's if you want a kit. Take mine if you want a skeleton to fork. Breadth versus a surface you can read in an afternoon.

## On AskUserQuestion and Anthropic's workshop

The architect's grill uses Claude Code's `AskUserQuestion`. Anthropic backs the pattern in their [How We Claude Code workshop](https://github.com/anthropics/cwc-workshops/tree/main/how-we-claude-code) and the [video walkthrough](https://www.youtube.com/watch?v=IlqJqcl8ONE). The workshop's three phases map onto my harness:

- **Phase 1 (interview brainstorm)** ≈ architect grill via `AskUserQuestion`
- **Phase 2 (divergent planning)** ≈ operator picks `mode` and branching choices
- **Phase 3 (verifiable build)** ≈ proof gate + spec audit + HITL citation

The interview pattern isn't mine. [Matt Pocock's `grill-with-docs`](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs) is a Socratic interview that stress-tests a plan against the project's domain language and maintains a `CONTEXT.md` glossary plus ADRs. I applied the interview shape one layer deeper: produce an issue-to-contract mapping with a citation invariant on the unblock. Framing is matt's; the citation enforcement is mine.

## Rules vs workflows is the design call that mattered most

Mechanical work goes in scripts. Judgement goes in markdown. Mechanical: "extract this section", "kill this port", "retry on 5xx". Judgement: "when to ask the operator", "what's a distinct behavior", "how to triage a block".

Stale prose is silent: no compile error catches contradictions. One SKILL.md still listed an XML tag as required after the parser stopped reading it. Harness kept producing plausible output. I found it by grepping.

Pair every rule with a script that enforces it. Grill-interview rule: strict-mode validator that fails without an audit log. HITL gate rule: regex on the unblock command. When the rule changes, enforcement breaks loudly. When enforcement changes, the rule reads as stale.

If you can't answer "where does this break loudly?", the rule will rot.

## Bound the cost and time

Four hard caps: per-call dollar budget, per-agent wall-clock timeout, per-proof timeout, max review rounds. Plus the auditor runs with a tighter tool permission set: read-only, can't mutate code regardless of budget.

The dollar budget is shared across builder and auditor. Sloppy. The auditor's read-only and could run on a quarter of the budget. On the redo list.

No alert at round two of three: operator finds out at the cap. A "round 2 of 3" GitHub comment would let me intervene early. Cheap, unshipped.

Treat agent budgets like a runaway cron's resource limits. Default tight, raise with evidence, alert before the hard stop.

## XML is about the parser, not the model

I moved from markdown headings to XML tags so the harness could inject one slice per agent:

```xml
<core_ac>
- [FAST] ...
- [ACCURATE] ...
</core_ac>

<agent_contract>
- proof_command: ...
- stable_interface: ...
</agent_contract>
```

Builder reads `<core_ac>` plus `<agent_contract>`. Auditor reads `<core_ac>` plus issue body, not `<agent_contract>` (so it can't grade implementation against implementation hints). Two roles, two slices, one document. XML makes this a tiny regex; markdown headings would need fragile boundary detection.

Anthropic's guide says XML helps the model parse unambiguously, but I haven't seen logit-level benchmarks proving `<tag>` beats `## heading`. Model claim: plausible-but-unproven. Harness claim: obvious.

Ship XML if your reason is "parser needs it". The "LLM thinks harder" reason is weaker.

Vocabulary note: I call this role the **auditor**, not a reviewer. It doesn't see the diff, can't mutate, outputs a binary verdict. That's compliance, not code review. "Reviewer" is the human at the HITL gate (who reads the diff and posts a citable comment) or the orthogonal-falsification agents (multi-perspective check). Three different things, three different words.

## HITL gates only work if the human actually loops

First HITL gate fired correctly: blocked at PR-open, waited for an unblock with "human approved". On review, the gate took 37 seconds from block to unblock-with-approval. No human in those 37 seconds. The agent reviewed its own PR and stamped it.

The fix was a regex on the unblock reason:

```
human approved (sha=<git-sha-7+>|comment=<pr-comment-id>)
```

Reasons without an auditable artifact get rejected. The gate is now provable from the events log alone.

Generalize: if a contract says "a human did X", enforcement should require a machine-checkable artifact only a human could have produced. Otherwise the contract is theater.

## Two quiet auth foot-guns

Any spawned claude inherits the parent env, including `ANTHROPIC_API_KEY`. If set, claude bills against the API key. If not, OAuth subscription. The harness strips the variable explicitly. Without that strip, any shell session with the env set silently falls back to API-key billing. No warning in the transcript. You find out from the invoice.

A `--bare` flag for stricter read-only on the auditor forces API-key auth and breaks subscription. The code comment about why I didn't use it stays as a tripwire. The right answer is `--disallowedTools` (write, edit, notebook-edit): same effect, no auth breakage.

Assume your auth chain has at least one path that silently downgrades you. Strip it. Test where you're billed.

## State that survives a crash

Two files per issue: a JSON state checkpoint and an append-only JSONL event log. The checkpoint lets `continue` fast-forward after a crash. The event log lets me reconstruct what happened.

Cache immutables on the checkpoint at intake. Reading context.md every iteration wasted a syscall; re-running default-branch detection wasted a gh round-trip. Persisting at intake cut redundant ops and made cold continues read from the same cache as warm ones.

Rule: write the checkpoint before any side-effect, cache immutables early.

## Single-tenant by choice

Refusing concurrent builds eliminated more complexity than any other constraint. No locks, no races, no shared-state coordination. The system fits in your head when one operator uses it at a time.

I had a lockfile mechanism for months: busy-poll on a per-clone lock file, 60-second timeout. Added when I was nervous about concurrent operators. Then I removed concurrent support entirely. The lockfile stayed. Dead defense for a threat model I'd already excluded.

When I dropped it, the diff was clean. No callers depended on the lock semantics. Pure paranoia for a non-existent scenario.

When the threat model changes, audit the defenses you no longer need. Paranoia accretes.

## What I'd redo

- **Use `--system-prompt-file` for prompt caching.** The cache rewards stable prefixes. My current code rebuilds the prompt inline every round and never gets a cache hit.
- **Tighten the auditor's budget below the builder's.** Same flag, smaller value. Read-only audits don't need the builder's headroom.
- **Post a "round 2 of 3" alert to the GitHub issue.** Burning the third round silently is wasteful when a human could've intervened cheaply.
- **Write at least one unit test of the merge path.** Two latent bugs (an unawaited async retry, a wrong gh-merge flag) survived three releases because the only test was live dogfooding. Five lines of test would've caught both at design time.
- **Treat my own notes with the same suspicion as the code.** Several "what I learned" bullets were wrong against the live source by the time I came to write this. Stale notes are worse than no notes when an LLM reads them as ground truth.
- **Steal Ralph's `progress.txt` + `AGENTS.md` pattern.** My per-round transcripts are forensic dead-ends: I read them when an issue is blocked, never again. Ralph curates a `## Codebase Patterns` section at the top of `progress.txt` that the next iteration reads first. Knowledge accretes across iterations instead of being trapped in per-round logs. The closest my harness gets is the architect's `<grill_log>`, which only captures pre-build decisions. Biggest borrow from the ecosystem.
- **Test a lighter coding agent for trivial issues.** Smaller model, smaller context, smaller budget. Renames, log additions, dep bumps don't need a strongest-tier agent. Unverified; the economics suggest there's something there.

## Open questions

Things I haven't proven and would want nailed down before recommending this for anyone else's serious work:

- **Sub-agent depth limit.** My notes say Claude Code enforces one level. Unverified against current docs.
- **Adversarial issue bodies.** Tested only against short, well-formed bodies. Not stress-tested against contradictions, multi-language content, or prompt-injection.
- **Concurrent operators.** Defenses gone. The audit trail interleaves. Unknown failure mode.
- **Plugin cache staleness.** Editing source under a local clone doesn't refresh the runtime cache. Workaround documented. Bit me at least once.
- **Cross-mount worktree moves.** The bucket-move uses a single rename syscall and fails across filesystem boundaries. Untested in containers.

## What I'd tell anyone starting from scratch

A harness spends money instead of time. If your time is worth more than a few dollars per useful change and the issue is well-scoped, the trade works. If your time is cheap or the issue is ambiguous, open the PR yourself.

Don't treat the harness as production infrastructure. Treat it as a POC substrate you'll refactor per project. What I built is ~600 lines of state machine, opinionated. Expect to rewrite half. The grill, role-scoped prompts, XML schema, builder-auditor split: choices, not requirements.

The most useful artifact from this project, more than the code, is the contributing document I wrote alongside. Read that first.

---

## Reading list

**Patterns and inspiration:**

- [OpenAI: harness engineering](https://openai.com/index/harness-engineering/)
- [Geoffrey Huntley on the Ralph loop](https://devinterrupted.substack.com/p/inventing-the-ralph-wiggum-loop-creator)
- [Mastering Ralph loops (LinearB)](https://linearb.io/blog/ralph-loop-agentic-engineering-geoffrey-huntley)
- [Ralph loop deep dive: from ReAct to Ralph](https://www.alibabacloud.com/blog/from-react-to-ralph-loop-a-continuous-iteration-paradigm-for-ai-agents_602799)
- [Codex /goal: OpenAI's built-in Ralph loop](https://ralphable.com/blog/codex-goal-command-ralph-loop-openai-built-in-autonomous-coding-agent-2026)
- [How We Claude Code workshop](https://github.com/anthropics/cwc-workshops/tree/main/how-we-claude-code) + [video](https://www.youtube.com/watch?v=IlqJqcl8ONE)
- [Martin Fowler: harness engineering for coding agent users](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html)

**Reference implementations and alternative kits:**

- [snarktank/ralph](https://github.com/snarktank/ralph): canonical Ralph implementation, ~113-line bash loop
- [obra/superpowers](https://github.com/obra/superpowers): alternative Claude Code methodology kit, ~15 skills, multi-harness
- [mattpocock/skills](https://github.com/mattpocock/skills/tree/main/skills/engineering): engineering skills, origin of the interview-grill pattern

**Technical references:**

- [Anthropic XML tags](https://platform.claude.com/docs/en/docs/build-with-claude/prompt-engineering/use-xml-tags)
- [Anthropic prompt engineering best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Claude Code sub-agents](https://docs.claude.com/en/docs/claude-code/sub-agents)
