+++
title = "What I learned from building an agent harness"
date = 2026-05-24T00:00:00Z
draft = false
tags = ["agents", "harness-engineering", "claude-code", "ralph-loop", "hitl"]
description = "A retrospective on building a single-tenant Claude Code plugin that drives a multi-phase build loop end-to-end. What worked, what broke, what I'd redo."
+++

> Retrospective on a Claude Code plugin I built called *buildkit*. The point: test whether the main agent in a session can act as a supervisor. Drive a workflow, resolve routine blocks, escalate the real decisions. The lessons generalize beyond the plugin. Fork the idea, not the code.

Most teams have ideas that don't justify dedicated engineering time: internal tools, POCs, side projects. A engineering harness could be useful for tasks like these.

A harness is the thin layer outside the model: the environment, constraints, and feedback loops that let an agent do useful work without a human watching every token. [OpenAI calls this harness engineering](https://openai.com/index/harness-engineering/) and runs Codex on it. [Claude Code's sub-agent model](https://docs.claude.com/en/docs/claude-code/sub-agents) lands in the same neighborhood.

A harness owns the lifecycle. The agents inside own the cognition. It pays off for tedious-but-non-trivial work that's expensive to do by hand and risky to fully automate.

The question I wanted to answer:

> **Can the main agent in a Claude Code or Codex session serve as a supervisor: driving a multi-step workflow, resolving routine blocks in-flight, escalating only the genuinely ambiguous calls back to me or the team?**

Not a one-shot prompt but a supervised session where the foreground agent supervises workers spawn per phase. The supervisor monitors and reads stage logs when blocked, patches and resumes, or escalates to a human.

The split I landed on is **afk versus hitl per work-unit**:

- **afk** issues run the full loop (intake, build, proof, review, PR, merge) without me. I find out the work shipped when I see the merge.
- **hitl** issues block at PR-open. They don't merge until I review the diff and post evidence.

Below i describe where it held, where it broke, what I'd redo. Not best practice. Just what I figured out.

## When the harness is worth it

Two conditions:

1. **Clear outcome.** When the task has a measurable success i.e a bug fix or hotfix. If a reviewer would know in two minutes, the harness can too.
2. **Reduce human loops with agent**: engineering time is a scarce resource that should be spent on more high value tasks, important but low urgency tasks can be delegated

The rule I use: if I can write a one-paragraph spec naming inputs, outputs, and success checklist without "could" or "maybe", the harness fits. Otherwise no.

## How this compares to Ralph loops

[Geoffrey Huntley's Ralph loop](https://devinterrupted.substack.com/p/inventing-the-ralph-wiggum-loop-creator): a bash while-loop that feeds the same prompt to a coding agent until done. Filesystem is the memory; conversation history isn't. Same context every iteration, different codebase on disk. The pattern works because LLMs degrade as context fills past a certain threshold (Huntley's framing: minimize allocation to avoid the "Dumb Zone" of 60-70% context fill). [OpenAI shipped a built-in Ralph loop in Codex CLI in April 2026](https://ralphable.com/blog/codex-goal-command-ralph-loop-openai-built-in-autonomous-coding-agent-2026), making the pattern mainstream.

My harness shares more with Ralph than I initially admitted: fresh-context iterations, filesystem memory, bounded iteration count. Identical wireframe however here is where they diverge:

- Ralph runs one prompt and trusts the agent to read its own learnings log between iterations. Knowledge gets curated into a `progress.txt` or `AGENTS.md`.
- My build-loop runs a phase state machine (build, proof, audit, PR, merge). Each phase gets a role-scoped prompt. The supervisor patches or escalates between phases.

Ralph trusts the worker; my harness gates phases. Huntley calls Ralph "merely the foundation"; multi-phase orchestrators are one direction. Ralph stays leaner; my harness catches more error classes per iteration.

What my harness is missing from Ralph: the human-curated learnings file. More below.

## Alternative kits

If you want something off the shelf, two are worth a look. [obra/superpowers](https://github.com/obra/superpowers) ships ~15 skills (brainstorming, planning, parallel agent dispatch, code review, worktree management, debugging, verification). Curated, v5.1.0, multi-harness. [mattpocock/skills](https://github.com/mattpocock/skills/tree/main/skills/engineering) ships tighter engineering skills (grill-with-docs, to-issues, triage, tdd, improve-codebase-architecture).

My harness isn't competing with either. It contains two skills, a shared parser, ~600-LOC state-machine core (~830 LOC total), intentionally hand-readable and opinionated. Use superpowers or matt's if you want a full kit. Take mine if you want a simple scaffold to fork.

## Using AskUserQuestion Tool

The architect's grill uses Claude Code's `AskUserQuestion`. Anthropic backs the pattern in their [How We Claude Code workshop](https://github.com/anthropics/cwc-workshops/tree/main/how-we-claude-code) and the [video walkthrough](https://www.youtube.com/watch?v=IlqJqcl8ONE). The workshop's three phases map onto my harness:

- **Phase 1 (interview brainstorm)** ≈ architect grill via `AskUserQuestion`
- **Phase 2 (divergent planning)** ≈ operator picks `mode` and branching choices
- **Phase 3 (verifiable build)** ≈ proof gate + spec audit + HITL citation

The interview pattern isn't mine. [Matt Pocock's `grill-with-docs`](https://github.com/mattpocock/skills/tree/main/skills/engineering/grill-with-docs) is a Socratic interview that stress-tests a plan against the project's domain language and maintains a `CONTEXT.md` glossary plus ADRs. I applied the interview shape one layer deeper to produce an issue-to-contract mapping with a citation invariant on the unblock. Framing is matt's; the citation enforcement is mine.

## Using Markdown files vs Scripts

I found that deterministic work is best authored as scripts/hooks. Judgement, knowledge and rules goes in markdown. Deterministic work typically involves tasks like "fetch from an API", "parse this data", "retry on 5xx". 

Judgement: "when to escalate to a human", "how to triage a blocked pipeline or intervene".

Stale prose is silent: no compile error catches contradictions or context rot in markdown files which is why there should be simple and more human readable.

I found it helpful to pair most rules with a deterministic script that enforces it. When the rule changes, enforcement in the scripts breaks loudly. When enforcement changes, the rule reads as stale.

If you can't answer "where does this break loudly?", the rule will most likely cause context rot.

## Bound the cost and time

I added four hard caps: per-run dollar budget, per-agent wall-clock timeout, per-proof timeout, max review rounds. Plus the auditor runs with a tighter tool permission set: read-only, can't mutate code regardless of budget.

The dollar budget was shared across builder and auditor which isnt the best design since the auditor is read-only and should be able to run on a quarter of the budget. This provides better cost control and has been flagged on the redo list.

I had no alert at round two of three operator finds out at the cap. A "round 2 of 3" notification system would have let me intervene early to better understand anomaly events.

## XML parsing

I moved from markdown headings to XML tags so the harness could inject one slice per agent:

```xml
<core_ac>
- [FAST] ... [latency]
- [ACCURATE] ...[correctness]
- [RELIABLE] ... [error rates]
</core_ac>

<agent_contract>
- proof_command: ...
- stable_interface: ...
</agent_contract>
```

Builder reads `<core_ac>` plus `<agent_contract>`. Auditor reads `<core_ac>` plus issue body, not `<agent_contract>` (so it can't grade implementation against implementation hints). Two roles, two slices, one document. XML makes this a tiny regex; markdown headings would need fragile boundary detection.

Anthropic's guide says XML helps the model parse unambiguously, but I haven't been able to benchmark whether `<tag>` beats `## heading`.

Vocabulary note: I call this role the **auditor**, not a reviewer. It doesn't see the diff, can't mutate, just outputs a binary verdict. That's more of a compliance check, not a full code review. "Reviewer" is the human during the PR at the HITL gate (who reads the diff and posts a citable comment).

## HITL gates only work if the human actually loops

Before i added HITL at the PR stage the agent reviewed its own PR and stamped it but now issues labeled `hitl` check for a regex on the unblock reason:

```
human approved (sha=<git-sha-7+>|comment=<pr-comment-id>)
```

Reasons without an auditable artifact get rejected. This is provable from the events log.


## Agent Authentication

Any spawned claude code session inherits the parent env, including `ANTHROPIC_API_KEY`. If set, claude bills against the API key otherwise it uses the OAuth subscription. The harness strips the variable explicitly. Without that strip, any shell session with the env set silently falls back to API-key billing with no warning.

I considered using the `--bare` flag for cleaner context but it forces API-key auth and breaks subscription. `--bare` reduces startup time by skipping auto-discovery of hooks, skills, plugins, MCP servers, auto memory, and CLAUDE.md. Without it, `claude -p` loads the same context an interactive session would, including anything configured in the working directory . I used `--disallowedTools` (write, edit, notebook-edit) which has a similar effect with no auth breakage.

## State that survives a crash

Two files are persisted per issue: a JSON state checkpoint and an append-only JSONL event log. The checkpoint lets `continue` fast-forward after a crash. The event log lets supervisor reconstruct what happened.

## Single-tenant by choice

Refusing concurrent builds eliminated complexity in the harness. No locks, no races, no shared-state coordination. This helped keep the design simple and avoid over-engineering.


## What I'd redo

- **Use `--system-prompt-file` for prompt caching.** The cache rewards stable prefixes. My current code rebuilds the prompt inline every round and never gets a cache hit.
- **Tighten the auditor's budget below the builder's.** Same flag, smaller value. Read-only audits don't need the builder's headroom.
- **Consider a simple notification system** Burning rounds silently is wasteful when a human could've intervened cheaply.
- **Write at least one unit test of the merge path.** Two latent bugs survived multiple loops because the only test was live dogfooding. Five lines of test would've caught both at design time.
- **Steal Ralph's `progress.txt` + `AGENTS.md` pattern.** Ralph curates a `## Codebase Patterns` section at the top of `progress.txt` that the next iteration reads first.
- **Use a lighter coding agent.** Pi's minimal system prompt and extensibility lets you do actual context engineering to control what goes into the context window and how it's managed.

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
