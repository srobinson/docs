# Experiment Harness

## Goal

Prove or disprove that fmm sidecars improve LLM task accuracy and navigation efficiency. Not theory — measured from Nancy logs.

## What We're NOT Doing

- Not optimizing for cost. Cost savings are a side effect, not the goal.
- Not relying on CLAUDE.md or system prompts alone. Instructions must be in the task prompt, reinforced at decision points.
- Not building mdcontext integration yet. That's premature until we prove sidecars help at all.

## The Instruction Problem

CLAUDE.md instructions get diluted. The agent sees "use fmm for navigation" in system context, then the immediate task pressure takes over and it greps anyway. The instruction to use sidecars must be:

1. In the task prompt itself (not just system context)
2. Reinforced in the fmm skill (loaded when the agent decides to explore)
3. Part of the tooling — if the fmm MCP server is the agent's primary search tool, the agent uses it by default because it's the most convenient path

This is an open problem. The experiment should measure whether the agent actually uses sidecars when instructed, and what instruction placement works.

## The Harness: Nancy

Nancy already provides:
- Task orchestration via Linear issues
- Worker isolation via git worktrees
- Full logging of every tool call (formatted + raw)
- Token usage tracking
- Iteration management

What's needed on top:
1. **Experiment runner** — script that sets up A vs B conditions for the same issue
2. **Log analyzer** — script that extracts metrics from Nancy formatted logs

## Experiment Structure

### Setup

For each experiment pair:

```
1. Pick a GitHub issue on a real open-source repo
2. Clone the repo twice (or use worktrees):
   - control/  (no fmm)
   - treatment/ (fmm generated + instructions in task prompt)
3. Create Linear parent + sub-issues for each condition
4. Run nancy orchestrate on both
5. Analyze logs
```

### What We Measure (extracted from logs)

**Navigation metrics:**
- Total `Read` tool calls before first `Edit`/`Write`
- Total `Grep`/`Glob` tool calls before first `Edit`/`Write`
- Unique files read during navigation phase
- Tokens consumed during navigation phase (pre-first-edit)

**Accuracy metrics:**
- Did the agent find the right files to modify?
- Did the agent understand the existing patterns?
- Did tests pass?
- Does the change match the issue requirements?

**Instruction compliance:**
- Did the agent use fmm tools/sidecars when available?
- How many times did the agent fall back to raw grep despite having sidecars?
- At what point did the agent start ignoring sidecar instructions (if ever)?

**Overall:**
- Total iterations to completion
- Total tokens consumed
- Token budget alerts triggered
- Task outcome (complete / partial / failed)

## Log Analyzer

Nancy formatted logs have a consistent format:

```
▶ Session started (model)
💬 Agent reasoning text
🔧 ToolName: arguments
   ← tool output
```

The analyzer needs to:
1. Parse formatted logs into structured events
2. Classify each event (navigation / task-work / rediscovery / boilerplate)
3. Identify the "first edit" boundary (everything before = navigation cost)
4. Count tool calls by type
5. Flag sidecar usage vs grep fallback in treatment condition
6. Output a comparison table

### Example Output

```
Experiment: openclaw#7027 — Telegram channel_post support
Repo: openclaw/openclaw (3071 files)

                          Control (no fmm)    Treatment (fmm)
                          ────────────────    ───────────────
Reads before first edit:  18                  6
Greps before first edit:  12                  2
Files discovered:         23                  8
Navigation tokens:        ~8400               ~2100
Time to first edit:       iter1 @ 47%         iter1 @ 18%
Sidecar usage:            n/a                 14/16 lookups
Task complete:            yes                 yes
Tests pass:               yes                 yes
Total iterations:         2                   2
Total tokens:             ~42000              ~28000
```

## Existing Data

We already have one experiment: ALP-479 (with fmm) vs ALP-480 (without fmm) on OpenClaw #5606.

Logs at:
- `fmm-gh-issues/openclaw-with-fmm/.nancy/tasks/ALP-479/logs/`
- `fmm-gh-issues/openclaw/.nancy/tasks/ALP-480/logs/`

This data hasn't been analyzed with the structured metrics above. First step: build the log analyzer and run it against ALP-479/480 to get a proper baseline.

## Sequence

1. **Build the log analyzer** — parse Nancy formatted logs, extract metrics, output comparison table
2. **Analyze ALP-479/480** — get real numbers from the experiment we already ran
3. **Run experiment 2** — openclaw#7027, using the harness properly this time
4. **Iterate on sidecar instructions** — if the agent ignores sidecars, vary instruction placement and measure compliance
5. **Decide on mdcontext integration** — only if sidecars prove valuable and search quality is the bottleneck
