# claude-skills

A collection of reusable skills for [Claude Code](https://claude.com/claude-code).

Each skill is self-contained and environment-portable. Personal or
environment-specific settings are kept out of the skill itself and live in
your own `~/.claude/CLAUDE.md` or `~/.claude/rules/*.md` as overrides.

## Skills

| Skill | Description |
|-------|-------------|
| [phase-harness](skills/phase-harness/) | Multi-phase development harness. Runs every role (Operator/Generator/Evaluator) as an isolated subagent so the main session stays thin. Supports DAG-based parallel phases, domain-specialized subagents, cost tiers, and multi-evaluator consensus for critical phases. |

More skills will be added over time.

## Installation

Skills are plain markdown files with YAML frontmatter. Install by copying
the skill directory into Claude Code's skills directory.

### User-level install (all projects)

```bash
# From this repo root:
cp -r skills/phase-harness ~/.claude/skills/
```

### Project-level install

```bash
# From your project root:
mkdir -p .claude/skills
cp -r /path/to/claude-skills/skills/phase-harness .claude/skills/
```

After installing, restart your Claude Code session (or reload skills) so
the new skill is discovered.

## Personal overrides

Each skill is written to work with self-contained defaults. Optional
personal overrides (e.g., mapping abstract Domain labels to your
environment's subagent types) live in:

- `~/.claude/rules/<skill-name>.md` — user-level rule file (loaded every session)
- `~/.claude/CLAUDE.md` — user-level global instructions

See `rules-examples/` for sanitized starting templates.

## Examples

`examples/<skill-name>/` contains concrete usage examples (e.g., filled-in
`phase-plan.md` for phase-harness).

## Contributing

Pull requests welcome. Guidelines:

- **Self-contained**: a skill should work without requiring specific
  subagents, MCPs, or plugins beyond what Claude Code provides by default.
  Use abstract labels + user-level rules for overrides.
- **Language**: skill bodies in English for portability. Comments/examples
  in additional languages are fine in separate files.
- **Size**: aim for under ~900 lines per `SKILL.md`. Split supporting
  material into separate referenced files when needed.
- **Description**: the frontmatter `description` drives skill triggering.
  Make it specific about when to use and what it does.
- **Tests**: if a skill is verifiable, include example scenarios in
  `examples/<skill-name>/`.

## License

MIT — see [LICENSE](LICENSE).
