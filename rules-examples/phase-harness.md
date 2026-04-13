# phase-harness personal overrides (example)

This is a sample rules file. Copy it to `~/.claude/rules/phase-harness.md`
and edit the mappings to match your environment's available subagent types.

The phase-harness skill ships with self-contained defaults. This file adds
environment-specific routing: abstract `Domain` labels → concrete
`subagent_type`, cost_tier defaults, checklist injection triggers, and
anything else you want to tweak.

## Domain → subagent_type mapping

When a phase declares a `Domain`, the Dispatcher looks up the concrete
subagent in this table. If the Domain is not listed or the specialist
doesn't exist in your setup, it falls back to `general-purpose`.

Replace the right-hand values with the subagent names you actually have
installed.

| Domain              | Generator subagent_type  | Evaluator subagent_type  |
|---------------------|--------------------------|--------------------------|
| `frontend-design`   | `frontend-design`        | `code-reviewer`          |
| `frontend-web`      | `frontend-developer`     | `typescript-reviewer`    |
| `backend-python`    | `python-pro`             | `python-reviewer`        |
| `backend-typescript`| `typescript-pro`         | `typescript-reviewer`    |
| `backend-go`        | `backend-developer`      | `go-reviewer`            |
| `backend-java`      | `backend-developer`      | `java-reviewer`          |
| `backend-kotlin`    | `backend-developer`      | `kotlin-reviewer`        |
| `backend-rust`      | `backend-developer`      | `rust-reviewer`          |
| `mobile-flutter`    | `flutter-expert`         | `flutter-reviewer`       |
| `mobile-swift`      | `backend-developer`      | `code-reviewer`          |
| `db-migration`      | `sql-pro`                | `database-reviewer`      |
| `devops`            | `cloud-architect`        | `code-reviewer`          |
| `testing`           | `tdd-guide`              | `code-reviewer`          |
| `documentation`    | `doc-updater`            | `code-reviewer`          |
| `cli`               | `cli-developer`          | `code-reviewer`          |
| `security`          | `security-reviewer`      | `security-reviewer`      |
| `data-analytics`    | `data-scientist`         | `code-reviewer`          |

**Operator is never specialized** — always `general-purpose` + the
Operator model (opus by default). Its job is cross-domain planning; a
specialist would narrow judgment inappropriately.

## Secondary Evaluators for high-criticality phases

For phases marked `criticality: high`, the Dispatcher runs one additional
Evaluator in parallel (an AND gate — both must PASS). Common choice:
`security-reviewer`. Tune to match your risk profile.

Typical `high`-criticality triggers: authentication, payments, encryption,
production-data migrations, PII handling.

## Evaluator checklist injection

The Dispatcher scans the spec for keywords and injects a matching
checklist into the Evaluator prompt. Adjust keywords / checklists as
needed:

- `authentication|session|jwt|oauth|password|crypto|encrypt` → **security** checklist
- `query|database|sql|orm|migration|index` → **performance** checklist (N+1, indexes, pagination)
- `api|endpoint|route|handler|public` → **backward-compatibility** checklist
- `input|form|upload|request|body` → **input-validation** checklist (XSS, injection, size limits)

## cost_tier default overrides

Default is `standard`. You may auto-promote certain phase patterns:

- Payments / auth / security-sensitive → `maximum`
- Production-data migrations → `maximum`
- Trivial CRUD / boilerplate → `minimal` auto-suggest

## Browser-verification tool preference

For UI-changing phases, preferred verification tool in order of preference:

1. Tauri projects → your Tauri MCP/skill (if any)
2. Web projects → `chrome-devtools` MCP
3. Otherwise → `playwright` MCP or manual

Adjust to match your environment.

## Safety hooks interaction (optional)

If your environment has PreToolUse hooks that block certain commands
(e.g., DDL/DCL blocking), document the interaction:

- Example: DDL/DCL commands are blocked by `ddl-dcl-block.py` → DB schema
  phases should surface a batch approval prompt to the user before the
  phase starts, rather than failing mid-run.

## Slash command aliases (optional)

Add language-specific aliases that the Dispatcher should treat like the
standard commands. Example (Korean):

- "일시정지" → `pause phase-harness`
- "재개" → `resume phase-harness`
- "상태 확인" → `status phase-harness`
- "아카이브" → `archive phase-harness`
- "도움말" → `help phase-harness`

## Notes on prerequisites

phase-harness requires Claude Code's subagent (Task/Agent) support. In
environments without subagents, use a single-loop harness (e.g., a
Ralph-style loop) or direct implementation instead.
