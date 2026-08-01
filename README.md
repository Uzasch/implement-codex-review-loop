# skills

Uzair's agent skills, as an installable Claude Code plugin.

```
/plugin marketplace add https://github.com/Uzasch/skills.git
/plugin install uzasch-skills@uzasch
```

Use the full `https://` URL, not the `Uzasch/skills` shorthand — the shorthand resolves to
`git@github.com:` and fails with `Permission denied (publickey)` on any machine without a GitHub
SSH key, even though this repo is public.

## Skills

### engineering

| Skill | What it does |
| :--- | :--- |
| `implement-codex-review-loop` | Implement a PRD or set of issues, then harden it in a Claude↔Codex review loop until Codex approves. |
| `implement` | Build the work described by a PRD or set of issues, TDD where there are seams, ending in a self-review and a commit. |

`skills/engineering/code-review-codex-loop/` ships too but is deliberately **not** in the plugin's
`skills` array: it is a *Codex-side* skill, discovered by the `codex` CLI from `~/.codex/skills/`.
`implement-codex-review-loop` symlinks it there during setup. Listing it would put a useless
`/code-review-codex-loop` in Claude's own skill picker.

## Requirements

`implement-codex-review-loop` additionally needs the `codex` CLI (authenticated), `gh`, and
`python3`. `codex-orchestrator` is declared as a dependency and installs automatically.

## Adding a skill

1. Create `skills/<category>/<name>/SKILL.md`.
2. Add `"./skills/<category>/<name>"` to the `skills` array in `.claude-plugin/plugin.json`.
3. Bump `version` in `plugin.json` so installed copies pick it up.
4. `claude plugin validate . --strict`

The array is required rather than optional here: the default scan only finds `skills/<name>/SKILL.md`
one level deep, so nothing under a category directory is auto-discovered. The upside is that
`skills/in-progress/` is a real staging area — a skill sitting there is version-controlled but not
shipped until you list it.

## Credits

- `skills/engineering/implement` — from [Matt Pocock's skills](https://github.com/mattpocock/skills)
  (MIT), which this repo's layout and the `/implement` + `/code-review` conventions follow.
- `codex-orchestrator` — [alexzh3](https://github.com/alexzh3/codex-orchestrator) (MIT). Fetched
  from upstream as a dependency, not vendored.
