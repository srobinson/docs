# Projects

## The Problem

LLMs have limited context windows. This is the number one blocker for LLMs being truly useful. Full stop. Everything below is an attempt to work around or solve this constraint. Nothing is proven yet — these are experiments along a thesis.

**Current focus: All effort is directed at addressing LLM context/memory limitations.**

## Lineage

Each project emerged from hitting the ceiling of the one before it:

```
1. Context window is too small — the root constraint
   └─ Nancy: keep the LLM working despite the limit (orchestration)
      └─ mdcontext: LLMs produce markdown nobody reads — make it searchable (search)
         └─ fmm: what if code itself had metadata LLMs could navigate? (navigation)
```

1. **Nancy** — The Ralph Wiggum loop showed LLMs can iterate, but they hit context limits and go sideways. Nancy adds orchestration so a human can intervene.
2. **mdcontext** — LLMs produce a LOT of markdown. Humans rarely consume it. LLMs can't either — context window. Useful content never acted on. mdcontext makes it searchable.
3. **fmm** — Taking it to code. What if every source file had frontmatter and the LLM had instructions to navigate metadata instead of reading files? Can we help the LLM navigate code without burning its window?

---

## nancy

Evolved from the [Ralph Wiggum loop](https://github.com/anthropics/claude-code/tree/main/plugins/ralph-wiggum) — a `while true` bash loop that keeps Claude Code iterating on a task. Nancy adds orchestration: human-worker communication, directive-based control, task sequencing via Linear, and session management.

- **Tech:** Bash, Claude Code CLI, tmux, Linear API
- **Linear ID:** `e56257da-fe79-4a3e-94c3-36a84de72904`
- **Status:** Planned (High priority)
- **Dir:** `nancy-placeholder/`

### What it solves

The Ralph Wiggum loop is fire-and-forget. Nancy adds the human back in: pause/unpause, send directives mid-run, monitor token burn, sequence work through Linear sub-issues. The worker still hits context limits, but the orchestrator can intervene before things go sideways.

### How it works

Define work in Linear as parent + sub-issues. `nancy orchestrate ALP-XXX` spins up a tmux session (orchestrator, worker, inbox panes). Worker iterates through sub-issues sequentially. Git worktree isolation per task.

### Open work

- ALP-228: Discoverable nancy skill as gateway to commands (Backlog)
- Batch of `[FUTURE]` issues deferred for public tool extraction

---

## mdcontext

LLMs produce a LOT of markdown. Humans rarely consume it. LLMs can't consume it either — context window. Useful content gets generated and never acted on. mdcontext makes markdown corpora searchable and consumable despite the window limit.

- **Tech:** TypeScript, Effect, @effect/cli, OpenAI embeddings, Vitest
- **Linear ID:** `e27dfa64-7e53-4aaa-9058-43ef3de00278`
- **Status:** Planned
- **Dir:** `mdcontext/`

### What it solves

You have 1500+ markdown docs. You can't stuff them in a prompt. mdcontext indexes them structurally, then provides semantic search (BM25 + vector embeddings with RRF fusion) so an LLM can pull just the relevant sections into its context window.

### Core features

- Structural indexing — parse markdown with sections, headings, links
- Semantic search — BM25 keyword + vector embeddings with RRF fusion
- Tree output — visualize document structure for AI consumption
- Watch mode — real-time index updates on file changes
- Multi-provider embeddings — namespaced vector stores, switch without rebuild

### Recent completed work

- ALP-264: Multi-provider embedding switching
- ALP-265: Timeout config fixes
- ALP-266/267: Progress indicators for indexing + embedding
- ALP-239: Replaced heavy `natural` dep with lighter alternatives
- ALP-240: Fixed stemText/matchPath bugs
- ALP-270: PR #18 review findings

---

## fmm

Taking the mdcontext insight further: what if every *code* file had frontmatter metadata, and the LLM had instructions to navigate via those metadata blocks instead of reading entire files? fmm generates `.fmm` sidecar files with structured YAML (exports, imports, deps, LOC) so LLMs can understand a file's role without consuming it.

- **Tech:** Rust CLI + TypeScript tooling
- **Linear ID:** `a8f1e6b8-4c24-40fd-97e6-36e381018fac`
- **Status:** Backlog (actively developed)
- **Dir:** `fmm/`

### What it solves

An LLM exploring a 3000-file codebase burns its entire context window just reading files to understand the architecture. fmm gives it a map — read the sidecar, understand the file's role, only open source when you need to edit. Hypothesis: 90%+ token savings on navigation.

### Core features

- Generates `.fmm` sidecar YAML per source file
- Claude skill + MCP server for codebase navigation
- `fmm gh issue <url>` — GitHub issue-to-PR pipeline using sidecars
- `fmm gh batch` — batch benchmarking with `--validate` for corpus health

### Current focus: Proving the hypothesis

- ALP-440: `fmm gh issue` pipeline (Worker Done, PR #17)
- ALP-476: `fmm gh batch --validate` (Done, PR #21)
- ALP-479/480: A/B experiment — same GitHub issue, WITH vs WITHOUT fmm navigation

---

## Experiment Log

### Experiment 1: OpenClaw #5606 (ALP-479 vs ALP-480)

Same GitHub issue (OpenAI Realtime API voice mode), same 3071-file repo. One worker WITH fmm, one WITHOUT. Analyzed by `nancy src/analyze` log analyzer (ALP-487, [PR #1](https://github.com/srobinson/nancy/pull/1)).

```
                           Control (no fmm)    Treatment (fmm)
Reads before first edit:   25                  40
Navigation tokens:         ~30,292             ~15,884
Navigation % of total:     52%                 41%
Time to first edit:        iter1 @ 48%         iter1 @ 13%
Sidecar lookups:           n/a                 0/55
Total tokens:              ~58,310             ~39,111
```

**Findings:**
- Treatment 33% more token-efficient overall
- Navigation is the dominant cost center in both conditions (41-52%)
- Treatment oriented faster (first edit at 13% vs 48% of iter1)
- **Zero sidecar usage** — agent ignored fmm tools entirely despite availability
- The instruction compliance problem is confirmed and is the next thing to solve

Full analysis: `docs/experiments/ALP-479-vs-480.md`

### Next: Solve instruction compliance

The harness works. The data is clear. Navigation dominates the window and fmm makes it cheaper — but only if the agent actually uses it. Three angles: task-prompt injection, tool restriction, sidecar-as-gateway. See `MEMORY_PROPOSAL.md` for details.
