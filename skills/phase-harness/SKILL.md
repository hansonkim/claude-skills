---
name: phase-harness
description: >
  Multi-phase development harness for complex features that span multiple
  phases (e.g., DB schema -> API -> UI -> integration). Every role (Operator,
  Generator, Evaluator) runs as an isolated subagent so the main session
  (Dispatcher) stays thin and the full feature can be built without exhausting
  context. Supports DAG-based parallel phase execution, domain-specialized
  subagents, cost tiers, multi-evaluator consensus for critical phases,
  progress streaming, and bounded retrofit. Use PROACTIVELY when the user
  describes a feature requiring 3+ distinct implementation steps, mentions
  "phases/stages/step-by-step development", or the task clearly spans multiple
  layers. Also use when a single-loop harness would likely exceed 10 iterations.
---

# Phase Harness

Multi-phase development orchestrator.
All roles (Operator, Generator, Evaluator) run as subagents so the main
session stays lean and each phase gets a fresh context window.

## Architecture

```
Main Session (Dispatcher) — thin loop, reads phase-status.json only
  │
  ├─ Phase 1
  │   ├─ Operator (planning model)    → writes phase-{N}/spec.md
  │   ├─ Generator (coding model)     → implements, streams progress
  │   ├─ Evaluator(s) (review model)  → independent verification
  │   └─ Dispatcher: PASS? → commit → Phase 2 (or parallel branches)
  │
  ├─ Phase 2 … (fresh context each phase; parallel where DAG allows)
  │
  └─ Final integration verification + archive
```

## Role Definitions

| Role | Location | Responsibility |
|------|----------|---------------|
| **Dispatcher** | Main session | Read phase-status.json, spawn subagents, commit, user communication, slash-command handling |
| **Operator** | Subagent | Plan phases, write per-phase specs, analyze prior results, distill patterns periodically |
| **Generator** | Subagent | Implement code, write tests, self-verify, stream progress |
| **Evaluator** | Subagent | Independent verification, code read-only; multiple Evaluators on critical phases |

**Dispatcher rules:**
- Never read source code or implementation files
- Read only phase-status.json, the Verdict line of eval-report.md, and
  progress.jsonl (tail only)
- Actions limited to: spawning subagents (serial or parallel), updating
  phase-status.json, git commit, user communication, slash commands

## Core Principles

1. **Full context isolation**: Every role is a subagent. The main session
   only reads a small status file and the tail of progress logs.
2. **Forward-preferred with bounded retrofit**: Phase order and verification
   gates are designed so rollback is unnecessary. When unavoidable, a
   bounded retrofit request may be logged and approved by the user.
3. **File-based handoff**: State passes between phases through files only.
4. **Verification gates**: A phase advances only after Evaluator(s) return
   PASS. Critical phases require consensus of multiple Evaluators.
5. **Learning accumulation**: Patterns discovered during a phase are
   persisted to `patterns.md` and to module-level `CLAUDE.md` files.
   Periodic distillation prevents pattern bloat.
6. **Adaptive cost**: Each phase declares a `cost_tier` so simple work runs
   on cheaper models while critical work runs on the strongest available.

---

## Model Selection

Each subagent is spawned with a specific Claude model. A phase's `cost_tier`
determines the concrete model assignments.

| cost_tier | Operator | Generator (simple) | Generator (complex) | Evaluator |
|-----------|----------|--------------------|---------------------|-----------|
| `minimal` | sonnet | haiku | sonnet | haiku |
| `standard` (default) | opus | sonnet | opus | sonnet |
| `maximum` | opus | opus | opus | opus |

Guidance:
- `minimal` — trivial CRUD, config files, rename-level refactors. Save cost.
- `standard` — default for most phases. Balanced.
- `maximum` — critical phases (auth, payments, data migrations at scale,
  high-stakes algorithms). Maximize judgment.

`cost_tier` is set per phase in `phase-plan.md` (see Step 1). Environments
may override via user CLAUDE.md / rules.

---

## Specialized Agent Routing

By default the Generator and Evaluator run as `general-purpose` subagents.
Quality and adherence improve when phases are routed to domain-specialized
subagents.

`phase-plan.md` declares a `Domain` label per phase (abstract, portable).
The Dispatcher maps the label to a concrete `subagent_type` via the user's
CLAUDE.md / `.claude/rules/`. If no mapping exists or the specialist is
unavailable, the Dispatcher falls back to `general-purpose` transparently.

### Abstract Domain labels

```
frontend-design, frontend-web, backend-python, backend-typescript,
backend-go, backend-java, backend-kotlin, backend-rust,
mobile-flutter, mobile-swift, db-migration, devops, testing,
documentation, cli, security, data-analytics
```

The Operator is always `general-purpose` + the Operator model.

### Forced Evaluator mapping (quality guarantee)

When a `Domain` is declared, the Dispatcher MUST use a matching language/
domain reviewer as the Evaluator (not `general-purpose`). Rationale:
language-specialized reviewers catch subtle idiomatic issues that generic
reviewers miss. Mapping is defined in the user's rules file.

Example mapping (actual mapping lives in user CLAUDE.md / rules):

```
| Domain         | Generator subagent   | Evaluator subagent   |
|----------------|----------------------|----------------------|
| backend-python | python-pro           | python-reviewer      |
| frontend-web   | frontend-developer   | typescript-reviewer  |
| testing        | tdd-guide            | code-reviewer        |
```

### Domain self-check (mismatch detection)

Every Generator/Evaluator subagent starts by reading `spec.md` and
self-assessing: "Does this work fit my declared domain?" If not, the
subagent records `domain_mismatch: true` in its report. The Dispatcher
retries the phase with `general-purpose` as a fallback.

### Multi-domain phases

A phase may declare multiple domains:

```
Domain: [backend-python, db-migration]
```

The primary domain (first) selects the Generator subagent. Additional
domains provide cross-domain Evaluators — one per additional domain —
each of which must independently PASS.

### Multi-Evaluator consensus (criticality)

Phases may declare a `criticality` level. `high` triggers multiple
independent Evaluators (e.g., domain reviewer + `security-reviewer`).
All must PASS for the phase to advance.

```
criticality: normal | high
```

### Evaluator checklist injection

Based on phase characteristics, the Dispatcher injects relevant checklists
into the Evaluator prompt:
- Phase touches authentication/session/crypto → security checklist
- Phase touches database queries → performance/N+1 checklist
- Phase touches public API → backward-compatibility checklist
- Phase handles user input → input-validation/XSS/injection checklist

---

## Prerequisites

### Required
- Claude Code or compatible agent with subagent (Task/Agent) support
- A writable `.harness/` directory in the project
- Git (for per-phase commits)
- An executable build/test toolchain

### Optional (enhance but not required)
- **Browser verification** for UI phases: `chrome-devtools` MCP,
  `playwright` MCP/CLI, project-specific UI skill, or manual confirmation
- **Domain specialists**: language- or role-specific subagents mapped via
  user CLAUDE.md / rules. Fallback to `general-purpose` is automatic.
- **Security reviewer** subagent for `criticality: high` phases
- **Pre-existing agent definitions** (e.g., from scaffolding) — if present,
  their tech context is incorporated into prompts.

### Gitignore (recommended)
Add to `.gitignore`:
```
.harness/
!.harness/archive/
```
(Archive directory kept under VCS for historical reference if desired.)

### Personal environment overrides
User-level overrides live in `~/.claude/CLAUDE.md` or
`~/.claude/rules/phase-harness.md`:
- Domain → subagent_type mapping
- Preferred browser verification tool
- Additional safety hooks behavior

Defaults are self-contained — overrides are optional refinements.

---

## Slash Commands (Dispatcher-recognized phrases)

The Dispatcher recognizes these user messages during a harness run:

| Command / phrase | Effect |
|---|---|
| `pause phase-harness` / `/phase-harness-pause` | Set `paused: true` in phase-status.json. Dispatcher waits after the current subagent returns and does not advance to the next phase. |
| `resume phase-harness` / `/phase-harness-resume` | Clear `paused: true` and continue. |
| `status phase-harness` / `/phase-harness-status` | Dispatcher reads phase-status.json and any `phase-{N}/progress.jsonl` tails, returns a concise progress summary to the user. |
| `archive phase-harness` / `/phase-harness-archive` | Move current `.harness/` contents (except `archive/`) to `.harness/archive/{YYYY-MM-DD}-{feature-slug}/`. Used at hand completion or before starting a new feature. |
| `help phase-harness` / `/phase-harness-help` | Dispatcher renders state-aware guidance (current phase, next steps, common options). |

These are documented behaviors — the Dispatcher interprets them without
needing separate command files. Skip creating dedicated slash-command
definitions unless your environment prefers that.

---

## Step 0: Suitability Check (quick triage)

Before launching the Operator, the Dispatcher does a brief self-check:

```
Is phase-harness actually the right tool?
- Genuinely 3+ distinct phases? yes → proceed
- 1-2 phases only? → propose single-loop harness OR direct implementation
- Exploratory, requirements unclear? → propose a short spike first, not a
  full harness
- Emergency bug fix? → propose direct implementation
```

If unsure, the Dispatcher surfaces the question to the user with concrete
alternatives (e.g., "This seems like a 1-file change; running phase-harness
adds ~13 subagent calls. Proceed with direct implementation instead?").

---

## Step 1: Initial Planning (Operator subagent)

After suitability check passes, the Dispatcher launches the Operator.

```
Agent({
  description: "Phase Harness: Initial Planning",
  model: "opus",
  prompt: <template below>
})
```

**Operator Planning Prompt:**

```
You are the Operator of a multi-phase development harness. Produce a
phase plan for the following feature request.

## Feature Request
{user's feature description}

## Project Context
- Working directory: {CWD}
- Read the project's CLAUDE.md (if present) for tech stack, build/test
  commands, and conventions.
- Analyze existing code structure with ls/Glob/Grep as needed.
- If `.claude/agents/*-planner.md` or similar scaffolding exists, use its
  tech context.

## Consider Phase 0: Discovery (optional)
If the feature is novel to this codebase, or your confidence in the plan
is low, include an initial "Phase 0: Discovery" whose Generator performs
a short spike (read code, run tools, gather facts) and produces
phase-plan-v2.md with a revised plan based on what it finds. Skip Phase 0
when the feature is well-understood.

## Output
Write `.harness/phase-plan.md` with this structure:

# Phase Plan: {FEATURE_NAME}

## Overview
{summary, tech stack, overall architecture decision}

## Tech Context
- Build: {build command}
- Test: {test command}
- Lint: {lint command}
- Browser verification: {MCP/skill/manual}

## Execution Settings
- plan_review_interval: 2       # re-review remaining phases every N phases
                                 # (0 disables; default 2)
- patterns_distill_interval: 3  # distill patterns.md every N phases
                                 # (0 disables; default 3)

## Phases

### Phase {N}: {PHASE_NAME}
- **Objective**: {what this phase achieves}
- **Inputs**: {outputs of earlier phases or existing code — concrete paths}
- **Outputs**: {list of files this phase produces}
- **Domain**: {abstract label OR [primary, secondary...]}
- **Depends**: [<prior phase ids>]   # omit or [] for no dependencies
- **Cost tier**: minimal | standard | maximum
- **Criticality**: normal | high
- **Generator model**: simple | complex
- **Acceptance Criteria**:
  - [ ] {statement: "what a user or downstream phase observes"}
        Verify: `{executable command or MCP/skill invocation}`
  - [ ] {statement 2}
        Verify: `{command}`
  UI-changing phases MUST include a browser-verification criterion.
- **Commit message**: {commit message for this phase}

## Planning Principles (strict)
1. Upstream first: schema → model → service → API → UI
2. Each phase consumes only the outputs of prior phases it declares as
   Depends
3. Define interfaces/contracts in early phases
4. Every acceptance criterion MUST be a "statement + executable verify"
   pair
5. Forward-preferred — designs requiring earlier-phase edits are rare;
   if unavoidable, log a retrofit_request instead of silently rewriting
6. Generator model per phase aligns with cost_tier and complexity
7. UI phases MUST have browser-verification criteria
8. Declare Depends explicitly so the Dispatcher can parallelize
   independent phases
```

After the Operator returns, the Dispatcher shows the plan to the user
and waits for approval. If Phase 0 was included, Phase 0 may execute
first and produce a revised plan; the revised plan is also shown for
approval before proceeding.

---

## Step 2: State Initialization

The Dispatcher creates `.harness/phase-status.json`:

```json
{
  "feature": "{FEATURE_NAME}",
  "total_phases": 4,
  "paused": false,
  "retrofit_request": null,
  "phases": [
    {
      "id": 1,
      "name": "{PHASE_NAME}",
      "status": "pending",
      "domain": ["backend-python"],
      "depends": [],
      "cost_tier": "standard",
      "criticality": "normal",
      "generator_model": "complex",
      "attempts": 0,
      "max_attempts": 3,
      "verdict": null,
      "domain_mismatch": false
    }
  ]
}
```

---

## Step 3: Phase Execution Loop

The Dispatcher iterates until every phase is `complete`.

### 3-0. DAG resolution (parallel scheduling)

Before each wave, the Dispatcher computes the set of ready phases:
a phase is ready when all its `depends` are `complete`. Multiple ready
phases run in parallel by issuing their 3-1/3-2/3-3 subagent calls in the
same message (multiple `Agent({...})` blocks).

Sequential semantics within a phase (Operator → Generator → Evaluator)
are preserved. Only inter-phase parallelism is enabled.

### 3-1. Operator — Phase specification

Before each phase, an Operator subagent reads prior results and writes the
current phase's concrete spec.

```
Agent({
  description: "Phase {N} Operator: {PHASE_NAME}",
  model: "<per cost_tier>",
  prompt: <template below>
})
```

**Operator Phase-Spec Prompt:**

```
You are the Operator for Phase {N}.

## Task
Analyze prior phase results and write a concrete execution specification
for Phase {N}.

## Files to read (in order)
1. `.harness/patterns.md` — accumulated Codebase Patterns. Read
   FIRST. Extract only items RELEVANT to Phase {N}; do not copy the
   whole file into the spec — summarize/cite the relevant subset.
2. `.harness/phase-plan.md` — Phase {N} section
3. `.harness/phase-{dep}/eval-report.md` for each dep in Depends
4. `.harness/phase-{dep}/generator-report.md` for each dep
5. Source files produced/modified by depended phases (from Outputs) and
   any CLAUDE.md in those directories

## Self-sufficiency
Do NOT ask the user questions. If information is missing, re-examine
files and make your best judgment. You do not have access to the
main-session conversation.

## Output
Write `.harness/phase-{N}/spec.md`:

# Phase {N} Specification: {PHASE_NAME}

## Objective
{refined goal — reflecting depended-phase outcomes}

## Context from Depended Phases
{files, interfaces, types the Generator needs}

## Applicable Patterns (excerpt)
{only patterns from patterns.md relevant to this phase}

## Module-level CLAUDE.md References
{list of module CLAUDE.md paths the Generator must read before editing}

## Implementation Requirements
{concrete instructions — file paths, function signatures, data structures}

## Acceptance Criteria
{copied verbatim from phase-plan.md — statement + verify pairs}

## Constraints
{what not to do; files from earlier phases that must not be modified}
```

### 3-2. Generator — Implementation

```
Agent({
  description: "Phase {N} Generator: {PHASE_NAME}",
  subagent_type: <per Domain mapping, or general-purpose>,
  model: "<simple or complex per cost_tier>",
  prompt: <template below>
})
```

**Generator Prompt:**

```
You are the Phase {N} Generator.

## Specification
Read `.harness/phase-{N}/spec.md` and implement accordingly.

## Domain self-check (first step)
Phase {N}'s declared Domain is: {Domain}
Read spec.md and confirm this work fits your domain expertise. If it
does not (e.g., you are python-pro but the spec asks for Rust):
  1. Stop implementation
  2. Write `.harness/phase-{N}/generator-report.md` with a single
     line: `domain_mismatch: true` and a brief reason
  3. Exit

## Self-sufficiency
Do NOT ask the user questions. Note ambiguities in the generator report
for the Evaluator.

## Progress streaming
Append one line to `.harness/phase-{N}/progress.jsonl` at each
major milestone using the format:
{"ts": "<ISO-8601>", "step": "<short label>", "status": "started|done|blocked"}
Examples of milestones: "read spec", "implement module X", "run tests",
"fix failing verify Y", "write report". This lets the Dispatcher brief
the user on demand.

## Work Rules
1. Read Applicable Patterns in the spec first
2. Read every CLAUDE.md in Module-level CLAUDE.md References
3. Implement every Implementation Requirement
4. Run each Acceptance Criterion's verify command and confirm PASS
5. Fix and re-run until all PASS
6. Do NOT violate Constraints
7. Write `.harness/phase-{N}/generator-report.md`:
   - Summary (files created/modified)
   - Acceptance Criteria: PASS/FAIL per item, with verify output
   - Known limitations
   - Discovered Patterns: candidate reusable conventions
{REWORK_FEEDBACK}
```

`{REWORK_FEEDBACK}` appears only on retries:
```
## Feedback from Previous Attempt
{Issues section of prior eval-report}
Address every item above in this attempt.
```

### 3-3. Evaluator(s) — Independent verification

For `criticality: normal` phases, one Evaluator runs. For `criticality:
high` phases, multiple Evaluators run in parallel; all must PASS.

For multi-domain phases (`Domain: [A, B, ...]`), one Evaluator per
domain runs in parallel (cross-domain verification). All must PASS.

```
Agent({
  description: "Phase {N} Evaluator ({which}): {PHASE_NAME}",
  subagent_type: <per Domain mapping; forced to language/domain reviewer
                  if Domain declared>,
  model: "<per cost_tier>",
  prompt: <template below — with checklist injection>
})
```

**Evaluator Prompt:**

```
You are the Phase {N} Evaluator. Independently verify the Generator's
output. Do NOT modify code. Use Read and Bash (execute/inspect only).

## Self-sufficiency
Do NOT ask the user questions.

## Specification
Read `.harness/phase-{N}/spec.md`.

## Generator Report
Read `.harness/phase-{N}/generator-report.md`. If it contains
`domain_mismatch: true`, record verdict FAIL with that reason and exit.

## Verification
1. Run every Acceptance Criterion's verify command directly
2. Confirm each statement's user/feature intent is actually met — do not
   trust the Generator's self-report
3. Additional checks (your judgment):
   - Code quality: function size, naming, error handling
   - Compatibility with prior phases
   - Constraints violations
   - Security, performance, compatibility per injected checklist (if any)
{CHECKLIST_INJECTION}

## Learning Accumulation (required when verdict is PASS)
- Append reusable general patterns to `.harness/patterns.md`
  (create with `# Codebase Patterns` header if missing)
- Append directory-specific patterns to the CLAUDE.md in that directory.
  Do NOT create a new CLAUDE.md; record "CLAUDE.md creation needed:
  {path}" in Learnings Consolidated for user review.
- Pattern quality threshold — add only if ALL hold:
  - Applies beyond this single phase
  - Has a concrete rationale you can state in one sentence
  - Not duplicated or contradicted by existing patterns.md entries

## Output
Write `.harness/phase-{N}/eval-report-{evaluator_name}.md`
(or `eval-report.md` if only one Evaluator). Structure:

# Phase {N} Evaluation Report ({evaluator_name})

## Verdict: PASS | FAIL | PARTIAL

## Verification Results
| Criterion | Result | Notes |
|-----------|--------|-------|
| {criterion} | PASS/FAIL | {details} |

## Issues (only when FAIL/PARTIAL)
- [ ] {concrete, actionable instruction}

## Quality Notes
{observations on code quality, security, compatibility}

## Learnings Consolidated
- Added to patterns.md: {list or "none"}
- Updated module CLAUDE.md files: {paths or "none"}
- CLAUDE.md creation needed: {paths or "none"}
```

`{CHECKLIST_INJECTION}` example snippets injected by the Dispatcher
when the spec touches the corresponding concerns:

```
### Security checklist (injected)
- Input validation on every boundary
- No secrets in code; env/config only
- Authentication/authorization on all protected paths
- Output encoding to prevent injection/XSS
- Rate limiting / abuse protection where applicable
```

```
### Performance checklist (injected)
- No N+1 queries
- Indexes present for filter/sort columns
- Pagination on potentially large result sets
- Caching on expensive repeated operations
```

### 3-4. Dispatcher Decision

The Dispatcher consolidates Evaluator verdicts (all must PASS for
multi-Evaluator phases).

**PASS (all Evaluators):**
1. Update phase-status.json: `status: "complete"`, `verdict: "PASS"`
2. `git add` + `git commit` with the phase-plan commit message
3. Move previous attempts' reports to numbered names:
   `eval-report.md` → `eval-report-attempt-{K}.md` for failed tries
4. Advance: mark this phase complete; Step 3-0 re-computes ready set

**FAIL/PARTIAL (any Evaluator):**
1. Increment `attempts`. Rename the failing attempt's reports to
   `generator-report-attempt-{K}.md` and `eval-report-attempt-{K}.md`.
2. If `attempts < max_attempts`: re-run Generator (not Operator) with
   combined Issues from all failing Evaluators as `{REWORK_FEEDBACK}`
3. If `attempts >= max_attempts`: report to the user with options:
   - Manual intervention
   - Accept with documented limitations
   - Revise the phase plan
   - Log `retrofit_request` if the real fix is in a prior phase

**domain_mismatch:**
1. Dispatcher re-runs the phase with `subagent_type: "general-purpose"`
   (bypassing the Domain mapping for this attempt)
2. Does not count against `max_attempts` (it's a routing miss, not a
   code failure)

### 3-5. Pattern Distillation (every `patterns_distill_interval` phases)

After every N completed phases (default 3, configurable), the
Dispatcher launches a short Operator subagent:

```
Agent({
  description: "Patterns Distillation",
  model: "<per project cost_tier default>",
  prompt: "Read .harness/patterns.md. Merge duplicates, remove
  phase-specific noise, consolidate related items, keep only general
  reusable conventions. Preserve the `# Codebase Patterns` header.
  Write the refined version back to patterns.md."
})
```

This keeps the patterns file from bloating into noise.

### 3-6. Plan Review Checkpoint (every `plan_review_interval` phases)

After every N completed phases (default 2, configurable), the
Dispatcher launches a brief Operator review:

```
Agent({
  description: "Plan Review Checkpoint",
  model: "opus",
  prompt: "Read .harness/phase-plan.md and all existing
  eval-reports. Are the remaining phases still appropriate given what
  we've learned? If no changes needed, write a single line:
  `plan_still_valid: true` to .harness/plan-review-{N}.md and
  exit. If changes needed, propose a revised plan and list the diffs."
})
```

If the review proposes changes, the Dispatcher shows them to the user
for approval before applying.

---

## Step 4: Final Integration Verification

After every phase is `complete`:
1. Run the full build and test commands (phase-plan Tech Context)
2. Report results to the user
3. On full pass, finalize phase-status.json to `"completed"`

---

## Step 5: Archive (on completion or new feature)

When the user invokes `archive phase-harness` (or the harness completes
successfully and the user starts a new feature), the Dispatcher moves:

```
.harness/*  (except archive/)
  → .harness/archive/{YYYY-MM-DD}-{feature-slug}/
```

The archive preserves the full history (phase-plan, patterns, all
phase-{N}/ dirs) for future reference.

---

## File Structure

```
.harness/
  phase-plan.md                  # Full plan (Operator, Step 1)
  phase-status.json              # Execution state (Dispatcher)
  patterns.md                    # Accumulated Codebase Patterns
                                 # (Evaluator append, periodic distill)
  plan-review-{N}.md             # Plan review checkpoints (optional)
  phase-1/
    spec.md                      # Operator output
    generator-report.md          # Generator output (successful)
    eval-report.md               # Evaluator output (single) OR
    eval-report-{evaluator}.md   # Per-Evaluator (multi)
    generator-report-attempt-{K}.md  # Failed attempt history
    eval-report-attempt-{K}.md
    progress.jsonl               # Streamed milestones
  phase-2/
    ...
  archive/
    2026-04-13-user-auth/        # Prior completed runs
      phase-plan.md
      patterns.md
      phase-1/...

Project source tree (example):
  src/api/CLAUDE.md              # Module conventions (Evaluator append)
  src/ui/CLAUDE.md
  ...
```

---

## Comparison with Single-Loop Harnesses

Single-loop harnesses run planner/generator/evaluator by switching roles
inside one session that reruns via a stop-hook. They work well for a
single feature or 1-2 phases but exhaust context on 3+ phases.

| Aspect | Single-loop | Phase-harness |
|--------|-------------|---------------|
| Suitable for | 1-2 phases | 3+ phases |
| Execution model | Stop-hook retry | Direct subagent invocations |
| Context | One session cycles | Fresh subagent per role |
| Main session | Runs the loop | Dispatcher only |
| Parallelism | None | DAG-based inter-phase |
| Specialization | None | Domain-specialist routing |
| Cost control | Fixed | Per-phase cost_tier |
| Critical-path review | Single evaluator | Multi-evaluator consensus |
| Rollback | Rework counter | Verification gates + bounded retrofit |

If the project already has scaffolding that generated agent files such as
`.claude/agents/{PROJECT}-planner.md`, incorporate their project-specific
rules into the Operator/Generator/Evaluator prompts for consistency.

---

## Quick Start

1. **Invoke the skill** with a multi-phase feature request:
   ```
   "Implement user authentication: DB schema, auth service, frontend,
    and E2E tests."
   ```
2. Dispatcher runs Step 0 suitability check. If it's genuinely multi-phase,
   it proceeds; otherwise it proposes a lighter alternative.
3. Operator produces `phase-plan.md`. Review and approve (or request edits).
4. Execution begins. Use `status phase-harness` any time for a progress
   summary. Use `pause phase-harness` to stop before the next phase.
5. On completion, review the final build/test report. Run
   `archive phase-harness` to stash the run under `.harness/archive/`.

## Example phase-plan.md

```markdown
# Phase Plan: User Authentication

## Overview
JWT-based user auth with email/password, added to existing Express + React app.

## Tech Context
- Build: `pnpm build`
- Test: `pnpm test`
- Lint: `pnpm lint`
- Browser verification: chrome-devtools MCP

## Execution Settings
- plan_review_interval: 2
- patterns_distill_interval: 3

## Phases

### Phase 1: DB schema
- Objective: users table with email/password_hash/created_at
- Inputs: existing migration infra at migrations/
- Outputs: migrations/0042_users.sql
- Domain: db-migration
- Depends: []
- Cost tier: minimal
- Criticality: normal
- Generator model: simple
- Acceptance Criteria:
  - [ ] users table exists with email UNIQUE and password_hash NOT NULL
        Verify: `psql -c "\d users" | grep -E 'email.*UNIQUE|password_hash.*NOT NULL'`
  - [ ] migration up/down/up round-trip works
        Verify: `pnpm migrate up && pnpm migrate down && pnpm migrate up`
- Commit message: "feat(db): add users table"

### Phase 2: Auth service
- Objective: email/password signup + login endpoints issuing JWT
- Inputs: Phase 1 schema
- Outputs: src/auth/*, tests/auth/*
- Domain: [backend-typescript, security]
- Depends: [1]
- Cost tier: standard
- Criticality: high
- Generator model: complex
- Acceptance Criteria:
  - [ ] POST /auth/signup creates a user and returns JWT
        Verify: `pnpm test tests/auth/signup.test.ts`
  - [ ] POST /auth/login returns JWT for valid credentials, 401 otherwise
        Verify: `pnpm test tests/auth/login.test.ts`
  - [ ] Passwords are bcrypt-hashed, never stored/logged plaintext
        Verify: `pnpm test tests/auth/hashing.test.ts`
- Commit message: "feat(auth): signup and login endpoints"

### Phase 3: Frontend forms
- Objective: signup and login forms with error states
- Inputs: Phase 2 endpoints
- Outputs: src/ui/auth/*
- Domain: frontend-web
- Depends: [2]
- Cost tier: standard
- Criticality: normal
- Generator model: simple
- Acceptance Criteria:
  - [ ] Signup form submits and redirects on success
        Verify: chrome-devtools MCP: navigate, fill, submit, assert URL
  - [ ] Login form shows error on wrong password
        Verify: chrome-devtools MCP: navigate, fill wrong creds, assert error
- Commit message: "feat(ui): signup and login forms"

### Phase 4: E2E tests
- Objective: full signup → login → protected route flow
- Inputs: Phases 1-3
- Outputs: e2e/auth-flow.spec.ts
- Domain: testing
- Depends: [3]
- Cost tier: minimal
- Criticality: normal
- Generator model: simple
- Acceptance Criteria:
  - [ ] Full flow passes end-to-end
        Verify: `pnpm test:e2e e2e/auth-flow.spec.ts`
- Commit message: "test(auth): add E2E signup→login flow"
```

## State-aware help

When asked for `help phase-harness`, the Dispatcher responds based on the
current state:
- Before Step 1 → explains suitability check and planning
- During planning → explains how to review/edit phase-plan.md
- During execution → explains pause/status/archive commands and how to
  interpret reports
- On failure → explains options (manual intervention, accept limitations,
  revise plan, retrofit_request)
