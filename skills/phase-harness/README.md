# phase-harness

Multi-phase development harness for Claude Code.

All roles (Operator, Generator, Evaluator) run as isolated subagents so the
main session (Dispatcher) stays thin. This keeps context usage low and
enables large, multi-layer features (DB → API → UI → tests) to be built in
a single run without exhausting the context window.

See [SKILL.md](SKILL.md) for the full specification.

## Architecture

```mermaid
flowchart TD
    User([User]) -->|feature request| Dispatcher
    Dispatcher{{"Dispatcher<br/>(main session)"}}
    Dispatcher <-->|read/write| Status[(phase-status.json)]
    Dispatcher <-->|append/read| Patterns[(patterns.md)]

    subgraph Phase1["Phase 1 — sequential within phase"]
        direction LR
        Op1[["Operator<br/>(opus)"]] -->|spec.md| Gen1[["Generator<br/>(sonnet/opus)"]]
        Gen1 -->|report + progress.jsonl| Ev1[["Evaluator(s)<br/>(sonnet)"]]
        Ev1 -->|eval-report.md<br/>PASS/FAIL| Gate1{{Gate}}
    end

    subgraph Phase2["Phase 2 — parallel when Depends allow"]
        direction LR
        Op2[["Operator"]] --> Gen2[["Generator"]] --> Ev2[["Evaluator(s)"]] --> Gate2{{Gate}}
    end

    subgraph Phase3["Phase 3"]
        direction LR
        Op3[["Operator"]] --> Gen3[["Generator"]] --> Ev3[["Evaluator(s)"]] --> Gate3{{Gate}}
    end

    Dispatcher -->|spawn subagents| Phase1
    Gate1 -->|PASS: git commit| Dispatcher
    Dispatcher -.->|DAG: parallel if no deps| Phase2
    Dispatcher -.->|DAG: parallel if no deps| Phase3
    Gate2 -->|PASS| Dispatcher
    Gate3 -->|PASS| Dispatcher

    Dispatcher -->|all phases PASS| Final[Final integration<br/>build + test]
    Final --> Archive[(archive/)]

    classDef dispatcher fill:#1f6feb,color:#fff,stroke:#0b4fc7
    classDef subagent fill:#2da44e,color:#fff,stroke:#216e39
    classDef evaluator fill:#bf8700,color:#fff,stroke:#7d5900
    classDef file fill:#fafbfc,stroke:#57606a,color:#24292f
    classDef gate fill:#cf222e,color:#fff,stroke:#82071e

    class Dispatcher dispatcher
    class Op1,Op2,Op3,Gen1,Gen2,Gen3 subagent
    class Ev1,Ev2,Ev3 evaluator
    class Status,Patterns,Archive file
    class Gate1,Gate2,Gate3 gate
```

The **Dispatcher** (main session) never reads source code. It only
reads `phase-status.json` and the *verdict* line of eval reports.
Every role (Operator, Generator, Evaluator) runs as a fresh
**subagent** with isolated context. Phases whose `Depends` lists are
satisfied run in parallel; within a phase the three roles run
sequentially through a verification gate.

## When to use

- Feature spans 3+ distinct implementation steps
- Task touches multiple layers (backend + frontend + infra + tests)
- A single-loop harness would likely exceed 10 iterations
- Context exhaustion has been a problem with prior attempts

## When NOT to use

- 1–2 phase work — use a single-loop harness or direct implementation
- Requirements are unclear / exploratory — do a short spike first
- Urgent bug fix — direct implementation is faster
- Cost-sensitive environments — this spawns many subagents

## Key features

- **Full context isolation** — Operator, Generator, Evaluator all run as
  subagents. The main session only reads status JSON.
- **DAG-based parallel phases** — independent phases run in parallel via
  simultaneous subagent invocations.
- **Domain-specialized subagents** — abstract `Domain` labels route to
  language/role specialists (Python-pro, frontend-developer, etc.). Falls
  back to `general-purpose` transparently when specialists aren't available.
- **Cost tiers per phase** — `minimal` / `standard` / `maximum` chooses
  the model mix so simple phases are cheap and critical phases use the
  strongest model.
- **Multi-evaluator consensus** — `criticality: high` phases require
  multiple Evaluators (e.g., domain reviewer + security reviewer) to
  all PASS.
- **Evaluator checklist injection** — security / performance / compatibility
  checklists auto-injected based on phase content.
- **Forward-preferred with bounded retrofit** — verification gates prevent
  rollback. When unavoidable, a `retrofit_request` is logged for user
  approval instead of silent rewrites.
- **Learning accumulation** — `patterns.md` and module-level `CLAUDE.md`
  files are appended by Evaluator on PASS. Periodic distillation prevents
  bloat.
- **Progress streaming** — Generators append JSONL milestones so the user
  can ask for status without waiting for the phase to complete.

## Install

```bash
# User-level (all projects)
cp -r skills/phase-harness ~/.claude/skills/

# Or project-level
mkdir -p .claude/skills
cp -r /path/to/claude-skills/skills/phase-harness .claude/skills/
```

## Optional: personal overrides

Map the skill's abstract `Domain` labels to your environment's concrete
subagent types by adding a rules file:

```bash
cp /path/to/claude-skills/rules-examples/phase-harness.md \
   ~/.claude/rules/phase-harness.md
# edit to reflect your actual subagent names
```

Without this, all phases fall back to `general-purpose` subagents —
still functional but without domain-specialized quality gains.

## Example

See [examples/phase-harness/](../../examples/phase-harness/) for a
filled-in `phase-plan.md` demonstrating DB schema → auth service →
frontend → E2E tests phases.

## Slash commands

During a harness run, the Dispatcher recognizes:

- `status phase-harness` — progress summary from status.json + progress.jsonl
- `pause phase-harness` — stop before the next phase
- `resume phase-harness` — continue from a pause
- `archive phase-harness` — move completed run to `.harness/archive/`
- `help phase-harness` — state-aware guidance

## License

MIT — see the repo root LICENSE.
