# The Coordinator Workflow — Replication Guide

_A portable write-up of the KruxCode build process (operator + coordinator + tiered
subagents), written 2026-07-02 so it can be re-instantiated in future sessions and other
projects. Everything project-specific is tagged **⚙ ADAPT**. Everything else is the
process itself — it earned its shape by catching real bugs, and each rule notes why it
exists._

---

## 0. The shape in one paragraph

One human **operator** directs one **coordinator** agent (largest model, highest
reasoning effort). The coordinator writes almost no production code itself — it
orchestrates: a multi-agent **research loop (A)** before any plan is ratified, an
**implementer subagent** to build the ratified plan in an isolated worktree, an
adversarial **review loop (B)** after it's built, a **`.1` hardening commit** fixing every
confirmed finding, then a **gate report and a hard STOP** for operator approval before
merge. Every piece of state that must survive context loss lives in files, not in the
conversation. Model tiers are matched to the judgment each task requires, and model
boundaries fall only on agent boundaries (cache-safe).

```
scope → (A) research loop → ratify (OPERATOR) → build (implementer, worktree)
      → coordinator verify (diff scope + authoritative green)
      → (B) adversarial review → triage → .1 hardening → re-verify
      → GATE REPORT → STOP (OPERATOR) → --no-ff merge → green-confirm-after-merge → push
```

## 1. Roles

| Who | Does | Never does |
|---|---|---|
| **Operator** | Sets scope, answers open questions, ratifies plans, approves gates and merges | — |
| **Coordinator** (Fable 5 / xhigh) | Orchestrates; verifies diff scope; runs the **authoritative** green-confirm itself; merges; writes gate reports, decision log, review log; keeps the resume file current | Delegate green-confirm, merges, or gate reports; proceed past a gate without explicit approval; trust a subagent's reported "green" |
| **Subagents** (roster below) | All the reading, building, and reviewing | Touch frozen contracts; blanket `git add`; put session identifiers anywhere |

Two operator-relationship rules that keep the loop honest:
- **Propose-then-ask.** Surface the proposal + trade-offs and ASK before any write/push/
  destructive action. Description of an idea is not consent to execute it.
- **Approval is per-gate.** "Approved" at one gate does not extend to the next. But gate
  approval DOES imply the mechanical follow-through (merge, push, PR) — don't add an
  extra permission round-trip for those.

## 2. The subagent roster (`.claude/agents/*.md`)

Project-level agent definitions; each `.md` = frontmatter (name, description, `tools`,
`model`, `effort`, `color`) + a system-prompt body carrying the standing disciplines.

| Agent | Model / effort | Tools | Role |
|---|---|---|---|
| `kc-scout` | Haiku / **low** | Read, Grep, Glob, Bash (read-only) | Recon, inventories, grep fan-outs, parity baselines — the cheap-reads lever |
| `kc-mech` | Sonnet / **medium** | All except Agent/Workflow | Fully-specified mechanical work; **stops and escalates** on any design question |
| `kc-implementer` | Opus / **xhigh** | All | Executes the ratified plan in a worktree; returns a gate-style report |
| `kc-reviewer` | Opus / **xhigh** | Read, Grep, Glob, Bash (read-only) | (B) lenses + refute-by-default verifiers; findings only, never fixes |
| `kc-researcher` | Fable / **xhigh** | Read/Grep/Glob/Bash + Write/Edit (research dir only) + web | (A) lenses, adversarial plan critics, dossier synthesis |

Design principles behind the roster:
1. **Tier the model to the judgment required.** Most token burn is reads, not reasoning —
   route recon to Haiku, mechanics to Sonnet at medium (xhigh on boilerplate wastes
   thinking tokens), judgment to Opus/Fable at xhigh.
2. **Bake the standing disciplines (§5) into the definitions.** Saves ~1–2k tokens of
   boilerplate per brief, and kills self-propagating-convention bugs at the definition
   layer instead of relying on the coordinator to re-type rules correctly every time.
3. **Reviewer is structurally read-only.** Findings route to a `.1` fix commit by an
   implementer; accidental mutation during review is impossible, and review stays
   adversarial.
4. **Cross-model at the verify stage.** Implementer and reviewer default to the same
   model → shared blind spots. HIGH/CRITICAL findings get verified by refuters on a
   *different* model (per-call `model` override keeps the reviewer definition's tools +
   stance). Diversity catches what redundancy can't.
5. `.claude/` is typically gitignored → the roster is box-local. Keep a recreation spec in
   persistent memory or a committed doc.

## 3. The per-track loop

A **track** = one section of work (a feature, a subsystem, a hardening pass). Every track
gets the full loop — do not collapse it to "explore + DIY + one review" (it drifted that
way once; the operator pulled it back: the bracketing loops repeatedly catch critical bugs).

### 3.1 (A) Pre-research loop — before ratifying any plan
Run as a **Workflow** (deterministic fan-out), all calls `agentType: 'kc-researcher'`:
- **Lenses in parallel:** prior-art / harness study (how do mature systems solve this?)
  + integration analysis **grounded in the actual tree** (cite `file:line`; a loop once
  shipped a "no precedent exists" claim that was factually wrong) + any domain lens the
  track needs.
- **Synthesis** into a draft plan.
- **Adversarial critics** attack the draft's integration seams: name the file it will
  break, the test that will catch it, the migration it forgot. Verdict vocabulary:
  `ratifiable` / `ratifiable_with_fixes` (+fixes) / `not_ratifiable` (+why).
- **Reconcile** → commit the dossier to the durable research dir (never `/tmp`) →
  present the plan to the operator with **every open question carrying a
  recommendation**. Operator ratifies (often "recommended on all"); ratified answers
  become LAW for the build.

### 3.2 Build — spawn an implementer
- **Isolation:** each concurrent implementer gets its own in-tree git worktree
  (`.worktrees/<name>`, own branch). Parallelize only genuinely **disjoint** chunks;
  interdependent work goes to ONE sequential implementer.
- **The brief carries:** the ratified plan + LAW, the worktree/branch, a decision-number
  band, red-before-green for safety fixes, commit-per-step (never squash), and — for
  rewrite-class work — a **baseline inventory** so the report can tally
  ✓ rendered / → relocated / ⚠ missing (parity checklist).
- Standing disciplines come from the agent definition — the brief doesn't repeat them.

### 3.3 Coordinator verify (never delegated)
- **Diff scope first:** `git diff --stat` against the real merge-base *before* reading the
  report — catches moved-base and `git add -A` sweeps that a reported file list hides.
- **Authoritative green-confirm:** the coordinator re-runs the full test bar itself and
  reads the **real exit codes** (`PIPESTATUS[0]`, not a pipe tail's). A subagent's green
  is a claim, not evidence.
- Code-read the highest-stakes changes at source.

### 3.4 (B) Post-implementation review loop
Run as a **Workflow**, all calls `agentType: 'kc-reviewer'`:
- **Find:** parallel lenses (security, correctness, integration, UI-dead-paths, docs/
  public-hygiene, …), each returning schema-structured findings with `file:line` +
  a concrete failure scenario (inputs/state → wrong outcome). No scenario, no finding.
- **Verify:** refute-by-default — verifiers actively try to break each finding;
  uncertain → refuted. **3 refuters for HIGH/CRITICAL, at least one on a different
  model** (`model` override). Majority-refute kills the finding.
- **Triage:** coordinator ranks confirmed findings; spurious-source-grep findings die here.

### 3.5 `.1` hardening
One commit (or few) fixing **all** confirmed findings — red-before-green for the safety-
relevant ones. Then the coordinator **re-reviews each fix at source and re-runs green
itself** (same rule: never trust the subagent's green).

### 3.6 Gate → STOP → merge
- **Gate report** to the operator: (1) what was built, file-by-file; (2) how it works;
  (3) key decisions + rationale; (4) what was NOT done and why; (5) test summary;
  (6) real test output; (7) risks; (8) dependencies introduced; (9) verification
  commands; (10) changelog bullets + docs touched (or "none — internal" with why).
- **Hard STOP.** No merge without explicit approval.
- On approval: **`--no-ff` merge** (preserve per-step history; never squash/FF) →
  **green-confirm-after-merge** (auto-merge misses semantic conflicts — e.g. a new enum
  variant making another branch's exhaustive match non-exhaustive) → push.

### 3.7 Bookkeeping (continuous, not at the end)
- **Docs-sweep** ends every gate: grep for the feature across all doc surfaces, read doc
  vs code side-by-side, update or explicitly state "none — internal".
- **Deferrals become tracked git issues** (with cross-links), never TODO comments.

## 4. Workflow-tool script shape

Every `agent()` call in an (A)/(B) script names a roster `agentType` — a bare `agent()`
is a smell. `pipeline()` by default (no barrier); barrier only when a stage truly needs
ALL prior results (dedup, early-exit).

```js
// (A): lenses → critics → synthesis
agent(lensPrompt,   {agentType: 'kc-researcher', phase: 'Research', schema: LENS})
agent(critique,     {agentType: 'kc-researcher', phase: 'Critique', schema: VERDICT})

// (B): find → cross-model refute-by-default verify
pipeline(LENSES,
  l => agent(l.prompt, {agentType: 'kc-reviewer', phase: 'Review', schema: FINDINGS}),
  r => parallel(r.findings.map(f => () =>
    agent(refute(f), {agentType: 'kc-reviewer',
                      model: isHighSev(f) ? 'fable' : undefined,   // cross-model on HIGH/CRIT
                      phase: 'Verify', schema: VERDICT}))))
```

## 5. Standing disciplines (bake these into the agent definitions)

Each of these exists because its absence shipped a real bug:
- **No session identifiers anywhere** — no session URLs/trailers in commits, PRs, briefs,
  or compaction summaries.
- **Never blanket `git add`** — explicit paths only (repo roots accumulate untracked
  private notes).
- **Tests assert OBSERVABLES** (the projection / the wire), **mocks match REAL shapes**
  (read the real shape before mocking it).
- **Red-before-green** for safety-relevant fixes.
- **All-ASCII corpora hide UTF-8 char-boundary panics** — include non-ASCII in string-
  handling test corpora.
- Language/framework floor (⚙ ADAPT): here — no `unwrap()` in prod, `tracing` not
  `println!`, structured errors, no unjustified `#[allow]`, paths via constants,
  YAML-data capabilities, deterministic policy engine.
- Full production build for type-strict frontends (dev-mode jest/e2e miss what the
  production pipeline catches); visual baselines verify-only on dev boxes, refresh in CI.
- Env hygiene before test runs (unset the known leaking vars); known-flake list is
  explicit and everything else is yours.
- **Durable thinking is committed** (research/dossiers → a committed research dir, never
  `/tmp`); private operational notes → a gitignored notes dir. Test: "would losing it
  cost rework?" → commit it.

## 6. Token & cache efficiency

1. **Coordinator model is fixed for the session.** Model swaps only at agent boundaries —
   each subagent is its own context, so a Haiku scout never busts the coordinator's cache.
2. **agentType per call, definitions frozen mid-track.** Same-type+same-model cohorts
   share a cached system-prompt prefix; editing definitions mid-track breaks that.
3. **Never continue an existing agent with a changed model** (re-reads its transcript
   uncached).
4. **Recon in cheap contexts:** scouts read, expensive models receive conclusions —
   file dumps never land in xhigh contexts.
5. Briefs don't repeat what definitions carry.

## 7. State & continuity (surviving compaction and box death)

- **Resume file** (gitignored, repo root): a `[CURRENT STATE]` header that is law —
  current position, blockers, the single NEXT ACTION, open threads. Updated after
  **every meaningful step** (auto-compaction is unpredictable; stale resume = lost
  position). Never contains derivable facts (HEADs, counts) — those rot.
- **Live-state hook**: a SessionStart hook injects real git/gh state each session; it
  **always wins** over the resume file or a compaction summary; discrepancies get fixed
  in the resume file, not argued with.
- **CLAUDE.md carries the self-alignment section** ("on any resume: read the resume
  file, reconcile against live state, re-read the master plan + contracts") — CLAUDE.md
  is re-injected into every context window, so the process self-heals even after an
  unbriefed auto-compaction.
- **Persistent memory** holds operator feedback + process rules (including the roster
  recreation spec, since `.claude/` is gitignored).
- **Verify before destructive ops, always:** history rewrites, worktree removals, force
  pushes → inventory + backup + verify + present + wait for explicit go, and check
  cross-branch blast radius (a "safe" rewrite once turned out to fork ~46 live sibling
  branches sharing the base).

## 8. ⚙ ADAPT checklist for a new project

1. Write the project's CLAUDE.md: resume/self-alignment section, test bar, language
   floor rules, doc surfaces, port/table facts agents keep getting wrong.
2. Create the resume file + (optionally) the SessionStart live-state hook.
3. Create `.claude/agents/` roster: copy the 5 definitions, swap the project-specific
   disciplines in §5, keep the models/efforts/tool restrictions. Must restart claude-code before new agents are usable.
4. Establish: decision log + review log + issue-for-every-deferral.
5. Set the coordinator model/effort; state the never-delegated list (green-confirm,
   merges, gates).
6. Agree the gate format with the operator (the 10 points in §3.6).
7. First track: run the FULL loop even if it feels heavy — the loop is the product.