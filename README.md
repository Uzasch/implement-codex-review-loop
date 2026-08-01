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

**`/implement-codex-review-loop <issue #s | PRD path> [--rounds N] [--base ref] [--fanout]`**

Matt Pocock's `/implement` with two things added: a **Codex review loop**, and an **orchestrator
for specs too big for one context**.

- **Implement** — build against the pinned spec, TDD at pre-agreed seams, then self-review on the
  two axes (Standards, Spec) and commit. This is the `/implement` + `/code-review` pass, inlined.
- **Codex loop** — Codex reviews the committed diff independently, in one persistent session that
  remembers what it already asked for. Claude triages every finding against the ADRs and the
  originating issue, fixes what it accepts, and answers back. Repeats until Codex approves or the
  round cap is hit. Claude's own self-review is deliberately withheld from Codex — a primed
  reviewer is not an independent one.
- **Orchestrator (ultracode)** — a spec too large for one context is fanned out across parallel
  agents via the Workflow tool, behind a size gate, with a slice map and a seams list so
  cross-slice inconsistency is reviewable as a Standards finding. Opt-in via `--fanout`; without
  it the skill sizes the issue, proposes the split, and stops for an answer rather than spending a
  fleet on its own initiative.

Ends with a drift report: everything built beyond the issue and beyond the ADRs.

Self-contained — it calls neither `/implement` nor `/code-review`, so no other skills plugin needs
to be installed.

`codex-skill/` inside it is the **Codex-side** reviewer, not a Claude skill. Codex discovers it by
name from `~/.codex/skills/`, so the loop symlinks it there during setup. Nothing to install by
hand, and it never appears in Claude's skill picker.

## Requirements

`codex-orchestrator` is declared as a dependency and installs automatically. You also need the
`codex` CLI (authenticated), `gh`, and `python3` on the machine.

## Adding a skill

1. Create `skills/<category>/<name>/SKILL.md`.
2. Add `"./skills/<category>/<name>"` to the `skills` array in `.claude-plugin/plugin.json`.
3. Bump `version` in `plugin.json` so installed copies pick it up.
4. `claude plugin validate . --strict`

The array is required rather than optional: the default scan only finds `skills/<name>/SKILL.md`
one level deep, so nothing under a category directory is auto-discovered. The upside is that
`skills/in-progress/` is a real staging area — a skill can sit there version-controlled but
unshipped until you list it.

## Repo assumptions

`implement-codex-review-loop` carries rules specific to `Uzasch/video-compilation2.0` stated as
facts about that repo, not universals — the live-systemd-units hazard, the normally-dirty working
tree, `docs/agents/issue-tracker.md`, the `CLAUDE.md` lint baselines, `docs/adr/`. Elsewhere they
read as harmless context; edit them if they get in the way.

The trunk is resolved from `origin/HEAD` and the skill stops to ask if that is unset. It never
falls back to `main`.

## Credits

This is [Matt Pocock's](https://github.com/mattpocock/skills) `/implement` and `/code-review` (MIT)
reworked into one skill — the build discipline, the two-axis Standards/Spec split, and the Fowler
smell baseline are all his. No files from that repo ship here; the content is inlined and rewritten
so the skill stands alone.

`codex-orchestrator` is [alexzh3's](https://github.com/alexzh3/codex-orchestrator), MIT, fetched
from upstream as a dependency rather than vendored.
