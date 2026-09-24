# Code That Holds

Claude Code skill: `code-that-holds`

## What

Defect families that survive a green test suite: validated-not-enforced, optional controls, rollback counters, null vs false, literal drift, AST-vs-behaviour, multi-door rules, reporting vs asking, production config. Distilled from real product defects. Use while writing, not only when reviewing.

## When to use

While writing or reviewing code; before committing; when deciding whether a change is finished.

## Install

Copy this folder into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/code-that-holds
cp SKILL.md ~/.claude/skills/code-that-holds/
```

Claude Code loads `SKILL.md` from `~/.claude/skills/<name>/`.

## Sibling skills

`guards-that-scan`, `fail-closed-review`, `reachability-audit`, `verified-delivery`, `adversarial-qa`, `contract-and-compat`, `migration-and-data-safety`, `handoff-faber-rigor`, `karpathy-method`

Proposed repos: see [Vorxeo](https://github.com/Vorxeo) `skill-*` packs.

## License

MIT — Copyright (c) 2026 Vorxeo. See [LICENSE](./LICENSE).
