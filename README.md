# codex-collab

**English** · [简体中文](./README.zh-CN.md) · [繁體中文](./README.zh-TW.md) · [日本語](./README.ja.md)

A playbook for pairing Claude and [Codex](https://developers.openai.com/codex)
as a two-model team inside [Claude Code](https://claude.com/claude-code): Claude
plans, reviews, and gates; Codex executes and first-pass reviews. It encodes a
division of labor — and the operational mechanics of making it reliable — as a
single file you drop into Claude Code.

> Claude tokens are expensive, Codex tokens are cheap. Spend Claude on judgment,
> not breadth.

---

## The problem it solves

Running one model as both author and reviewer has two failure modes:

1. **Cost.** Using a premium model for bulk reading, mechanical edits, and wide
   fan-out burns expensive tokens on work a cheaper executor could do.
2. **Self-consistent hallucination.** A single model reviewing its own output
   shares its own blind spots — a fresh session removes *conversational*
   self-justification but not *model-level* ones (same weights → two sessions
   can "independently" agree on the same wrong belief).

`codex-collab` addresses both by splitting the roles across **two different
models**:

- **Claude** owns direction (plan, risks, acceptance criteria), model dispatch,
  and the final quality gate — it reads real diffs and re-runs the oracle itself.
- **Codex** owns execution and a fresh-session first-pass review.
- A **cross-model gate** (Claude reviewing Codex's work against pre-written
  criteria, never against Codex's own summary) catches the blind spots a
  same-model review rubber-stamps.

On top of that it standardizes the *operational* details that make multi-round,
multi-agent delegation actually work — handoff docs, output contracts against
truncation, worktree isolation for parallel fan-out, test-interpreter
verification, and parallel-session safety on shared repos. These rules are distilled from real-world delegation experience.

## Recommended usage — the dual-workflow model

**The recommended way to run codex-collab is the dual-workflow model.** Multi-
round coding/research runs as **two workflows in parallel**:

- a **Workflow** layer (multi-agent orchestration) that fans out, decomposes,
  and verifies; and
- the **codex-collab** layer that delegates implementation to Codex while Claude
  holds planning, review, and gating.

Every fan-out lane is a *Codex executor* — the orchestration raises Codex
throughput; it does not move execution back onto Claude. See
[`dual-workflow.md`](./dual-workflow.md) for the full operating model
(division of labor, externalizing state to GitHub/Linear, anti-bloat).

Single-round trivia skips all of this — just do it directly, whichever model is
faster.

## What's in this repo

| File | What it is |
| --- | --- |
| [`SKILL.md`](./SKILL.md) | The full playbook: fast path, roles & dispatch, cost rules, review discipline, mechanics, output contract, and ultracode fan-out mode. |
| [`dual-workflow.md`](./dual-workflow.md) | The orchestration model that wraps the playbook. |

## Requirements

- **[Claude Code](https://claude.com/claude-code)** (the host this runs in).
- **Node.js 18.18+** (required by the Codex plugin).
- **Codex CLI** — `@openai/codex`.
- A **ChatGPT subscription (incl. Free) or an OpenAI API key** for Codex.
  Codex usage counts toward your Codex limits.

## Installation

### 1. Install and log in to the Codex CLI

```bash
npm install -g @openai/codex
codex login          # or configure an OpenAI API key
```

### 2. Install the Codex plugin for Claude Code

codex-collab drives Codex through the official OpenAI Codex plugin
([`openai/codex-plugin-cc`](https://github.com/openai/codex-plugin-cc)), which
provides the `/codex:*` commands and the `codex:codex-rescue` subagent it
delegates to. Inside Claude Code:

```text
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

`/codex:setup` reports whether Codex is ready; it can offer to install the Codex
CLI for you if npm is available. After install you should see the `/codex:*`
slash commands and the `codex:codex-rescue` subagent in `/agents`.

The plugin gives you:

- `/codex:review` — normal read-only Codex review
- `/codex:adversarial-review` — steerable challenge review
- `/codex:rescue` / `/codex:status` / `/codex:result` / `/codex:cancel` —
  delegate work and manage background jobs

### 3. Install codex-collab

It installs as a Claude Code skill — a single file under
`~/.claude/skills/<name>/`. Copy this repo's `SKILL.md` into place:

```bash
mkdir -p ~/.claude/skills/codex-collab
cp SKILL.md ~/.claude/skills/codex-collab/SKILL.md
```

Claude Code auto-discovers it from the YAML frontmatter `name`/`description`.
Restart Claude Code (or reload) and it becomes available — Claude invokes it
automatically for multi-round coding/research, or you can call it by name.

### 4. Configure Codex

Codex reads `~/.codex/config.toml`. codex-collab assumes a couple of settings;
a minimal example:

```toml
# Pick whatever Codex model your plan exposes (e.g. the latest gpt-5.x).
model = "gpt-5"
model_reasoning_effort = "xhigh"   # Claude overrides per-lane: medium / high / xhigh
model_verbosity = "low"

# codex-collab writes complete lists to files and keeps chat replies short, but
# a generous tool-output limit reduces mid-stream truncation; it calls out 32000
# specifically.
tool_output_token_limit = 32000

# For unattended delegation in a trusted local repo. Loosen/tighten to taste —
# more restrictive sandboxes are fine but some lanes (e.g. in-worktree commits)
# need workspace write access.
approval_policy = "never"
sandbox_mode = "workspace-write"
```

Notes:

- `model_reasoning_effort` is the default; Claude picks `--effort` **per lane**
  by task shape (mechanical → `medium`, multi-branch → `high`, novel/security/
  no-oracle → `xhigh`).
- Codex's sandbox blocks `/dev/nvidia*`: GPU-bound lanes must run on the Claude
  side. Plan that split up front.

## Verifying the setup

```text
/codex:setup            # confirms Codex CLI present + authenticated
/codex:review --background
/codex:status
/codex:result
```

If `codex:codex-rescue` shows up in `/agents` and a review round-trips, you're
ready. Then just start a multi-round task — Claude will reach for it.

## Usage patterns

The recommended default is multi-round feature work via the dual-workflow model,
but the same Claude-judges / Codex-executes split fits many shapes:

- **Multi-round feature work (recommended)** → the [dual-workflow model](#recommended-usage--the-dual-workflow-model):
  Claude writes acceptance criteria, fans out Codex executor lanes (worktree-
  isolated for parallel work), runs fresh Codex first-pass reviews, then re-runs
  the oracle itself at the gate before merge.
- **Cross-model code review** → run `/codex:review` or `/codex:adversarial-review`
  on a diff or PR for an independent second-model pass. A different model catches
  the blind spots a same-model self-review rubber-stamps.
- **Large refactor / migration** → fan out one Codex lane per file (worktree
  isolation), each with an explicit do-not-touch list, then cherry-pick the
  disjoint commits. Scale beyond what a single context can hold.
- **Bug hunt / audit** → loop-until-dry: each round fans out Codex finders and
  adversarially verifies every finding (a majority "refute" kills it); stop after
  K empty rounds.
- **Test generation** → parallel Codex lanes add tests for untested modules;
  Claude re-runs the suite itself at the gate before trusting them.
- **Cost-optimized bulk reading** → delegate wide reading to a read-only Codex
  recon lane that returns a report; Claude reads the report and spot-checks key
  files, spending premium tokens only on judgment.
- **Research / exploratory** → an inverted rhythm: pre-register metrics and
  thresholds before implementation; only a fixed, deterministic evaluator
  declares success; stop-loss at 2–3 rounds per hypothesis.
- **Single-round trivia** → skip the ceremony; Claude does it or delegates one
  lane directly.

Read [`SKILL.md`](./SKILL.md) for the full discipline and
[`dual-workflow.md`](./dual-workflow.md) for the recommended operating model —
both are written to be followed verbatim by the agent.

## License

[MIT](./LICENSE).
