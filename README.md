# implement-codex-review-loop

A Claude Code plugin. Implement a PRD or a set of issues, then harden the result in a
Claude↔Codex review loop: Claude implements and self-reviews, Codex reviews independently in one
persistent session, Claude triages every finding against the ADRs and the originating issue, fixes
what it accepts, and answers back. Repeats until Codex approves or the round cap is hit.

## Install

```
/plugin marketplace add Uzasch/implement-codex-review-loop
/plugin install implement-codex-review-loop@implement-codex-review-loop
```

That is the whole install. `codex-orchestrator` is declared as a dependency and is pulled in
automatically — you do not add its marketplace or install it yourself.

Then run it in any repo:

```
/implement-codex-review-loop #76 #77
/implement-codex-review-loop docs/prd/thing.md --rounds 3 --fanout
```

## Requirements

Install brings in `codex-orchestrator` for you. You still need these on the machine:

| Requirement | Why |
| :--- | :--- |
| `codex` CLI, authenticated | the review rounds are `codex exec` |
| `gh` CLI, authenticated | Phase 1 pins the spec from the issue tracker |
| `python3` | `codex_orch_tools.py state` / `validate` |

### Why codex-orchestrator is listed in this marketplace

Dependencies resolve within the declaring plugin's own marketplace, so this marketplace carries a
second entry for `codex-orchestrator` that points at
[`alexzh3/codex-orchestrator`](https://github.com/alexzh3/codex-orchestrator) as its source. It is
fetched unmodified from upstream — nothing is vendored or forked here, and it keeps receiving its
own updates. It owns the journal contract, run initialization, and `codex_orch_tools.py`, which
this skill defers to rather than reimplementing.

## What ships here

- `skills/implement-codex-review-loop/` — the orchestrator skill you invoke.
- `skills/implement/` — the Phase 1 build skill it delegates to.
- `skills/code-review-codex-loop/` — the **Codex-side** reviewer skill. Codex discovers it by name
  from `~/.codex/skills/`, so the loop's *Setup* step symlinks it there on every run. Nothing to
  install by hand.

## Repo assumptions

The skill carries hard-won rules from `Uzasch/video-compilation2.0` that are stated as specific
facts about that repo, not as universals — the live-systemd-units hazard, the normally-dirty
working tree (brand `log.md`, `MEMORY.md`, data backups), `docs/agents/issue-tracker.md`, the
`CLAUDE.md` lint baselines, and `docs/adr/`. On another codebase they read as harmless context;
edit them if they get in the way.

The trunk is resolved from `origin/HEAD` and the skill stops to ask if that is unset. It never
falls back to `main`.

## Credits

- `codex-orchestrator` — [alexzh3](https://github.com/alexzh3/codex-orchestrator), MIT. Fetched
  from upstream as a dependency, not vendored.
- `skills/implement/` — derived from [Matt Pocock's](https://github.com/mattpocock) engineering
  skills, which this workflow's `/implement` and `/code-review` conventions come from.
