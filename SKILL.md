---
name: codex-collab
description: "Claude×Codex division of labor for multi-round coding or research work — Claude plans/reviews/gates, Codex executes. Use when: delegating implementation to Codex, starting a multi-round work stream (defaults to ultracode + Workflow fan-out), cross-model review, or cost-aware model dispatch. Not needed for one-round trivial fixes — just do those."
---

# Claude × Codex collaboration

Codex tokens are cheap, Claude tokens are expensive: spend Claude on
judgment, not breadth. All rules are defaults with room for judgment.

## Part I — Orientation (read first)

### Quick reference

| Section | Use when |
| --- | --- |
| [§2 When to use](#2-when-to-use) | deciding fast-path vs multi-round; sizing the harness tax |
| [§3 Roles & dispatch](#3-roles--dispatch) | who owns what, picking the orchestration shape, choosing Codex `--effort`, fallback when Codex fails |
| [§4 Cost rules](#4-cost-rules) | spending Claude attention deliberately; what you may never economize on |
| [§5 Engineering rhythm](#5-engineering-rhythm) | running a normal multi-round engineering stream + handoff doc |
| [§6 Research / exploratory rhythm](#6-research--exploratory-rhythm) | exploratory / hypothesis-driven rounds (inverts §5) |
| [§7 Review: breaking self-consistent hallucination](#7-review-breaking-self-consistent-hallucination) | designing the review/gate so two same-weight sessions don't share a blind spot |
| [§8 Delegation & process mechanics](#8-delegation--process-mechanics) | dispatch channels, interpreter/path probes, freeze gates, background handoff, GPU sandbox, flag ordering |
| [§9 Output contract (anti-truncation)](#9-output-contract-anti-truncation) | preventing truncated lists/results; persisting complete data |
| [§10 Linear & GitHub + parallel-session safety](#10-linear--github--parallel-session-safety) | state layering across Linear/doc/PR; avoiding sibling-session collisions |
| [§11 Ultracode mode](#11-ultracode-mode) | the default fan-out orchestration for any multi-round stream |

Legend: every rule is numbered `R<section>.<n>` for cross-reference.

### 1. Core principle

(Stated in the lede above.) Codex tokens are cheap, Claude tokens are
expensive: spend Claude on judgment, not breadth. All rules are defaults
with room for judgment.

### 2. When to use

| Work shape | Action |
| --- | --- |
| One-round trivial (small fix, question, obvious edit) | do or delegate directly, whichever is faster — no handoff doc, no opening pass |
| Multi-round stream (everything below) | DEFAULT to ultracode + a dynamic, multi-agent Workflow — see [§3b Orchestration default](#3b-orchestration-default) and [§11 Ultracode mode](#11-ultracode-mode) |

For multi-round streams the model picks the concrete shape per scenario
(loop-based vs fixed fan-out, width, depth); drop to serial single-Codex
only for an indivisible lane.

> **Measured harness tax (2026-06-12):** every Codex invocation carries a
> fixed ~8s harness tax (process + ~15k-token agent bootstrap), regardless
> of effort, warm runtime, or MCP — noise for minutes-long batched tasks,
> dominant for trivia. Never delegate trivia; the rescue subagent adds its
> own spin-up on top.

## Part II — How work is split

### 3. Roles & dispatch

#### 3a. Role split

| Actor | Owns |
| --- | --- |
| Claude | direction (plan, risks, acceptance criteria), model dispatch, final quality gate |
| Codex | execution + first-pass review (fixed lane) |
| Explore/search subagents | haiku/sonnet |

#### 3b. Orchestration default

- **R3.1** Default orchestration for multi-round streams: ultracode +
  Workflow fan-out (see [§11 Ultracode mode](#11-ultracode-mode)). The
  Workflow is a harness that runs MORE Codex in parallel — every fan-out
  lane is a Codex executor — NOT a way to shift execution onto Claude. It
  raises Codex throughput; Claude stays judgment-only (direction, gates,
  synthesis).
- **R3.2** Drop to serial single-Codex delegation only for one indivisible
  lane or when fan-out isn't worth the worktree cost.

#### 3c. Effort selection

Claude picks Codex `--effort` per lane by task shape (overriding the xhigh
default is EXPECTED now, not exceptional):

| Tier | Use for | Examples |
| --- | --- | --- |
| **medium** | mechanical / low-ambiguity / one clear correct answer | pure fns, type guards, format conversion |
| **high** | multi-branch logic, several edge cases, domain nuance | parsers, precedence, state transitions |
| **xhigh** | novel algorithm, subtle correctness/security/concurrency, wide design space, or the review/gate lane | — |

- **R3.3** Bias DOWN when a deterministic oracle (tests/CI) will catch a
  miss; UP when no oracle or high blast radius.
- **R3.4** Keep `--model` at config default; pass effort via the executor
  prompt ("invoke codex-companion task with --effort <tier>").

#### 3d. Fallback

- **R3.5** Codex unavailable or fails twice → Claude implements directly,
  note it in the doc. Health check: `/codex:setup`.

### 4. Cost rules

- **R4.1** Bulk reading → read-only Codex recon task produces a report;
  Claude reads the report + spot-checks key files, then writes criteria.
- **R4.2** Per-round attention is tiered: round summary + `diff --stat` +
  Codex fresh-review verdict (cheap); full diff only at escalation triggers.
- **R4.3** Engineering: batch delegations large (`--background`), few
  round-trips.
- **R4.4** Keep Claude sessions short; state lives in the handoff doc.

> **NON-NEGOTIABLE — R4.5 — NEVER economize on:**
> - pre-implementation acceptance criteria;
> - the cross-model final gate (Claude reads real diffs at gates, never only
>   Codex's account);
> - raw-code spot checks.

## Part III — Execution rhythm & review

### 5. Engineering rhythm

- **R5.1** Opening pass → handoff doc `docs/handoff/<topic>.md`, git-tracked:
  plan, criteria, decisions, progress, problems. Delete in the final commit
  before merge (distilled into the PR description first).
- **R5.2** Update the doc on meaningful state change, not every round; Claude
  may jot updates itself. `--resume-last` is fine — the doc is the recovery
  point, not a session replacement.
- **R5.3** Each Codex round ends with: tests run (results pasted), round
  summary, fresh-session self-review.
- **R5.4** Scope change → amend the doc, no restart; full re-plan only if the
  approach is invalidated.

**Full-review triggers** (cross-referenced from [§7](#7-review-breaking-self-consistent-hallucination)):

1. criterion claimed done;
2. before commit/merge;
3. diff outside planned scope;
4. two stalled rounds.

### 6. Research / exploratory rhythm

This rhythm INVERTS the engineering rhythm in [§5](#5-engineering-rhythm).
Direction errors compound — Codex tunnels. Invert the default engineering
rhythm:

1. Pre-register metrics/thresholds/protocol in the doc BEFORE
   implementation; only the fixed evaluator's output declares success —
   self-report never counts. Oracles derive from the design spec or an
   external reference (statsmodels, numpy, paper formula), never from the
   implementation under test — an implementation-derived oracle is
   circular and passes wrong code.
2. Evaluator code is outside Codex's delegation scope; any diff touching it
   escalates to Claude. Prefer a deterministic evaluator (project CLI,
   pytest, fixed script) over an LLM-workflow evaluator when one exists —
   LLM evaluators die to quota/session limits mid-stream and their reports
   are advisory, not authority.
3. Stop-loss: max 2–3 rounds per hypothesis; miss the gate → return with
   findings. Pivoting metrics/hypotheses is Claude's decision, never
   Codex's. Revision rounds resume the executor's SAME session with an
   exact diagnosis of the miss (fresh sessions are for review, not
   revision — context of the attempt is an asset when fixing it).
4. One hypothesis per delegation; Claude reads EVERY round summary
   (trajectory watch — tunnel vision shows in trajectory, not diffs).

If your domain has its own fixed gates/ledgers, they take precedence.

### 7. Review: breaking self-consistent hallucination

Fresh sessions remove conversational self-justification, NOT model-level
blind spots (same weights → two sessions can "independently" share a wrong
belief). So:

- **R7.1** Codex first-pass review in a NEW session (`/codex:review` /
  `/codex:adversarial-review`), judged against Claude's pre-written
  criteria, not Codex's own summary.
- **R7.2** Claude's cross-model review is the final gate; Codex verdicts
  never substitute.
- **R7.3** Mechanical evidence (tests/CI/repro) outranks any model's "looks
  good". But before blaming YOUR diff for a gate failure, reproduce the exact
  same test invocation on the untouched baseline — order-dependent /
  state-polluting suites throw false regressions. Control one variable at a
  time (isolate the test AND keep the diff) before attributing.
- **R7.4** The evaluator gates Claude's own fixes too, not just Codex's:
  after any remediation, re-run the same evaluator on the remediated item
  until clean — a fix is done at re-verify PASS, not at commit. (Observed
  twice: first-pass verify evidence can overstate, and Claude's own
  adjudications carry the same blind-spot risk.)
- **R7.5** A GENERIC adversarial-verify wave is near-worthless against the
  auditor's *framing* errors — same weights, same blind spot, so the skeptic
  rubber-stamps and "confirms" most findings. When the audit has a known
  framing trap (e.g. spec-TEXT vs implementation), the re-verify prompt must
  ENUMERATE the concrete false-positive modes as a checklist, not just say
  "refute this" — a mode-enumerated skeptic separates real defects from
  plausible-but-wrong ones; a generic one cannot.
- **R7.6** NUMERIC / equivalence checks need PRODUCTION-SCALE inputs, not a
  tiny gate panel. A short probe can pass VACUOUSLY when the defect only
  manifests at scale — e.g. floating-point accumulation error that is
  negligible on a few rows but grows with length. For any
  accumulation-sensitive code, test at realistic size before trusting a
  pass, and let the mechanical result, not "looks safe", decide.

## Part IV — Operational mechanics

### 8. Delegation & process mechanics

#### 8a. Dispatch channels

- **R8.1** Delegate via `codex:codex-rescue` subagent (write-capable by
  default), `codex-companion.mjs task --write`, or plain `codex exec` in
  background with a Monitor tail when the companion isn't set up. Pass the
  handoff doc path; say what to implement and what to record. Monitor grep
  filters should match only verdict/test/error lines — broad patterns spam
  duplicate notifications.

#### 8b. Interpreter & path verification

- **R8.2** VERIFY THE TEST INTERPRETER before any fan-out: a wrong
  `.venv`-vs-conda guess makes EVERY lane's gate fail with
  ModuleNotFoundError (torch/pytest). Probe once — `<py> -c "import
  torch,pytest"` AND `PYTHONPATH=. <py> -c "import <pkg>"` from a worktree
  (PYTHONPATH=. must win over the editable install) — then hard-code that
  interpreter path in the executor prompt. (Cost this session: a full
  workflow kill+relaunch after lanes hit `.venv` with no torch.) This extends
  to CONSOLE SCRIPTS and CONFIG PATHS: a bare entry-point (e.g. a project
  CLI) can PATH-resolve to a different env's older copy than `<py> -m
  pkg.cli` loads (observed: base-env py3.13 console script lacked a flag the
  pinned-env editable build had), and a same-named config can live in two
  repos with different schemas. Invoke via the pinned `<py> -m`, and confirm
  the config/candidate paths resolve to the repo the runtime actually reads.

#### 8c. Freeze/contract gates & full-suite re-run

- **R8.3** Executor prompts NAME the repo's freeze/contract tests
  (export-surface, count pins, text-frozen docs) to run before commit —
  scope creep shows up there, not in the executor's own new tests. Re-run the
  gate yourself on the executor's worktree: a lane can pass its own gates
  while smuggling a HARMFUL drive-by (observed: an F lane gutted
  `torch.cuda.is_available()` from a cache module + hardcoded
  `torch.device('cuda')` in a budget module, unrelated to its issues — caught
  only by reading the full `diff --stat`, reverted before PR). A lane that
  runs ONLY its own issue's test also misses cross-test blast radius: name
  the FULL shared/governance gate suite (the whole CI gate's pytest list, not
  just the issue's file) in the executor prompt, and re-run that full suite
  yourself before merge — observed a contract-touching lane pass its own
  contract test yet break a sibling contract test (a sibling test in the same
  gate doing a self-compare with the new check); only Claude's full-gate
  re-run caught it. Worktree commits: under Workflow `isolation:'worktree'`
  the codex lane commits in-worktree itself (verified — writes the shared
  .git/objects); only in a restricted/companion sandbox that can't write the
  main repo's .git does Claude commit after the gate.

#### 8d. Background & checkout-handoff invariant

- **R8.4** Long runs: `--background` (direct task flag) or background
  subagent; follow up with `/codex:status` / `/codex:result`.
- **R8.5** Checkout-handoff invariant: an async/`--background` delegation can
  hand control back while codex is STILL alive and writing. Before the main
  agent reuses that checkout, either block until codex truly exits (a
  FOREGROUND companion `task` blocks on child exit — don't `--background`),
  OR confirm the process is dead: `kill -0 <pid>` on the tracked pid is the
  authoritative OS-level check (companion records `child.pid` per job; raw
  `codex exec` gives the shell pid); `/codex:status` `running[]` is
  convenient but only bookkeeping for companion jobs. NEVER mutate the same
  files on the assumption it stopped; concurrent same-checkout writes corrupt
  the tree. Worktree isolation sidesteps this; same-checkout reuse does not.

#### 8e. GPU sandbox constraint

- **R8.6** The Codex sandbox blocks /dev/nvidia* — `nvidia-smi` works but
  CUDA init fails. Any lane needing GPU (live probes, GPU compute) executes
  Claude-side; plan the split up front, don't discover it mid-round. A CPU
  `import pkg` SUCCEEDING does NOT prove CPU-runnable — heavy ML wheels import
  fine then fail-closed GPU-only at INSTANTIATION (e.g. a GPU ML library a
  batch-env builder builds a GPU-only component that requires CUDA). Probe a
  minimal CPU instantiate/forward Claude-side BEFORE routing the lane to the
  sandbox, not just the import.

#### 8f. codex exec flag ordering

- **R8.7** `codex exec` global flags go BEFORE the subcommand: `codex exec
  --sandbox workspace-write resume <session-id> "..."`.

### 9. Output contract (anti-truncation)

Chat output truncates at several layers (Codex `tool_output_token_limit`,
terse `model_verbosity=low`, Claude's Bash output cap). Rules:

- **R9.1** Complete data (failure lists, results, inventories) goes to a
  FILE — e.g. `docs/handoff/<topic>-results.md` or the artifact root; every
  Codex delegation prompt that produces a list ends with "write the complete
  list to <path>; chat reply = count + path + 3-line summary".
- **R9.2** Counts must reconcile: if the summary says 11 failures, the file
  must list 11. A mismatch is an incomplete round, not a display glitch.
- **R9.3** A READ-ONLY review lane cannot write its artifact — tell it to
  return the complete list in its chat reply and have Claude persist the
  file, or grant `--sandbox workspace-write`. Don't prompt a read-only lane
  to "write to <path>".
- **R9.4** NEVER rerun a task to recover lost output — fetch it with
  `codex-companion.mjs result [job-id]` (stored verbatim) or read the
  artifact file. Reruns burn tokens and can land on different results.

### 10. Linear & GitHub + parallel-session safety

#### 10a. Three-layer state model

Three layers, linked by reference, never duplicated:

| Layer | Holds | Lifetime |
| --- | --- | --- |
| Linear | why/what | permanent |
| Handoff doc | in-flight state | until merge |
| PR | change + review record | with the change |

Doc references the issue URL. Claude is the single Linear writer (status +
milestone comments only). Full review on the PR; `Fixes <issue-id>`
auto-links; CI is a mechanical gate alongside Claude's.

#### 10b. Parallel-session check

- **R10.1** On a shared repo where the user may run sibling sessions/worktrees
  on the same task, `git fetch origin <main>` and scan for an in-flight
  equivalent branch/PR BEFORE starting substantial work, and base on CURRENT
  origin/main (not a stale local HEAD) — rebase early, not at push time.
  Skipping this cost a full redundant component redesign: a parallel
  `worktree-*` branch landed the identical fix (and a better nan-guard) via
  its own PR while this session rebuilt it from scratch; the collision only
  surfaced at the conflicting-PR stage. Cheap insurance: a fetch + `gh pr
  list` up front.
- **R10.2** The up-front `git status` snapshot goes STALE within minutes if a
  sibling is committing to the SAME checkout's local `main` — re-run `git log
  origin/main..main` + `git reflog -5` RIGHT BEFORE you act, not just at
  session start (observed: a sibling agent committed a batch of tracked
  issues to local main 2 min before I'd have clobbered them; only the reflog
  timestamps revealed a live sibling — the opening status said "clean, behind
  by 2"). When found, cherry-pick its commits onto current origin/main as your
  own PR; leave its diverged local main alone.
- **R10.3** The branch/PR scan is NOT enough when you RESUME work a prior
  session set up: also detect a LIVE sibling still driving a background codex
  job on the SAME worktree — `pgrep -af "codex exec"` for the worktree path,
  a pending RESULTS.md, recent file mtimes — BEFORE you edit. A live sibling's
  git ops (commit/reset) can wipe your uncommitted changes and its concurrent
  same-worktree writes corrupt the tree (observed: a "handed-off" mint had its
  executor still alive + a parent still committing; my uncommitted admission
  fix got reset away and two pytest runs polluted each other). If a sibling is
  live, do NOT race its checkout — work in a SEPARATE worktree branched off
  its pushed commit, verify there, and reconcile via a stacked PR rather than
  editing the shared tree.

### 11. Ultracode mode

Default for multi-round streams — this is the orchestration default
referenced by [§2 When to use](#2-when-to-use) and
[§3b Orchestration default](#3b-orchestration-default).

#### 11a. Default shape & loop-vs-pipeline + HARD GUARD on stop condition

- **R11.1** DEFAULT shape = a dynamic, multi-agent Workflow; the model
  chooses the concrete form per scenario. Open-ended / unknown-size streams
  (discovery, audit, accumulate-to-target, research rounds) run
  loop-until-signal: each round fans out Codex lanes, STOP on K empty rounds
  OR a Codex-side pass signal. Known fixed-size work (a known file list, a
  fixed dimension set) uses pipeline/parallel fan-out instead — a loop there
  only adds a pointless convergence check. HARD GUARD (non-negotiable): stop
  a dynamic loop on round-cap + Codex-side signal, NEVER solely
  budget.remaining() — Codex GPT spend is unmetered (see the
  dynamic-workflow caveat in [11d](#11d-dynamic-loop-budget-caveat)), so a
  budget-guarded loop over-spins Codex and undercounts cost.

#### 11b. Recon fan-out defaults

- **R11.2** Recon fan-out defaults to Codex agents; Claude only for
  cross-model judgment; searches on haiku/sonnet. Judge panel only for
  genuinely competing approaches.

#### 11c. Execute contract

- **R11.3** Execute: `agent(prompt, {agentType: 'codex:codex-rescue',
  isolation: 'worktree', schema: {output: string}})`. Non-negotiable:
  - ANY parallel Codex fan-out (exec or review) uses worktree isolation —
    companion job state is per-checkout, concurrent same-checkout runs are
    unsafe. Codex scales to ~100 concurrent, so it is NOT the fan-out
    bottleneck — size lanes to the work, not to a fixed 3–5. The two real
    ceilings: (1) the Workflow per-run concurrency cap min(16, cores−2)
    (16 on a 32-core box) — pass more lanes and they queue, all still
    complete; (2) per-lane worktree cost (disk + a Codex runtime each).
    Lean wide (10–16) for genuinely parallel work; to exceed 16 simultaneous,
    split across multiple top-level Workflow runs (each gets its own cap).
  - Executor prompts must say: "run foreground, do not use --background;
    use --fresh" (backgrounded tasks return instantly and the worktree is
    cleaned up under Codex), and end with "git add -A && git commit".
  - Executors never edit the handoff doc — return did/decided/remaining in
    the structured result; Claude consolidates once per phase.
  - Empty/missing executor result = failure: retry once, then Claude
    implements that chunk.

#### 11d. Dynamic-loop budget caveat

- **R11.4** Dynamic (loop-based) workflows — loop-until-dry /
  loop-until-budget / while-accumulate — dispatch to Codex the same way: each
  iteration spawns `agent(..., {agentType: 'codex:codex-rescue', isolation:
  'worktree'})`; worktrees accumulate across rounds but still run ≤16 at
  once. CAVEAT: `budget.spent()` meters Claude orchestration tokens, NOT
  Codex's external GPT spend — the Codex lane is a thin Claude driver, so a
  budget-guarded loop massively undercounts real cost and over-spins Codex.
  Guard dynamic loops on round count or a Codex-side signal (tests pass,
  no-new-findings), not solely `budget.remaining()`.

#### 11e. Native state from the Workflow wrapper

- **R11.5** Native state comes from the Workflow WRAPPER, not the plugin. The
  `agent({agentType:'codex:codex-rescue'})` lane shows in `/workflows`,
  writes `subagents/workflows/*.jsonl`, returns a validated `schema` object,
  gets worktree isolation + a completion notification — all from the Workflow
  layer (verified this session). The plugin underneath is a DUMB FOREGROUND
  executor; its own `CLAUDE_PLUGIN_DATA/state` job store still writes but is
  harmless duplicate — the Workflow journal is authoritative. So DON'T fork
  the vendored plugin cache to "make Codex native" — it's already native at
  the layer that matters; Task-tool/status-line sync are cosmetic and belong
  upstream or in a thin separate wrapper, never a `plugins/cache/...` edit
  (gets wiped on update).

#### 11f. Verify lane / read-by-path rules

- **R11.6** Verify: fresh Codex agents first-pass against criteria; Claude
  synthesizes, reads diffs at the gate, adjudicates. Conflicting merges →
  serial Codex loop, not re-fan. The review lane reads the executor's
  worktree BY PATH (pass the executor's returned worktreePath) — do NOT give
  the reviewer its own `isolation:'worktree'`, a fresh worktree is empty of
  the executor's commit. Claude's gate still re-runs the oracle itself
  (mechanical > both models).
- **R11.7** Read-only AUDIT lanes (no executor to read after) that must see
  UNMERGED working state: their own `isolation:'worktree'` branches fresh
  from origin/default and will NOT contain your uncommitted/unpushed work —
  tell them to read source by ABSOLUTE path into the live worktree, not their
  own checkout. (Distinct from the review-lane rule above, which reads an
  executor's returned worktreePath.) Static-analysis flags carry a high
  false-positive rate (a multi-lane audit can flag N items while mechanical
  confirmation validates only ~half), so Claude's mechanical confirmation of
  every flag before acting on it is non-negotiable, per
  [R7.5](#7-review-breaking-self-consistent-hallucination).
