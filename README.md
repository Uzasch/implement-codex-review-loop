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

- **Implement** — calls his `/implement` unchanged, which runs `/tdd` at the seams and ends in his
  two-axis `/code-review` and a commit. Nothing is reimplemented here; that skill owns the build.
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

Install `uzasch-skills` and both dependencies come with it: `mattpocock-skills` for the build and
review skills, `codex-orchestrator` for the journal contract and Codex process control. You do not
add either marketplace yourself.

Phase 1 is 14 lines because his `/implement` does the work. The other ~480 are the Codex loop, the
fan-out, and the orchestration contract — the parts he does not have.

`codex-skill/` inside it is the **Codex-side** reviewer, not a Claude skill. Codex discovers it by
name from `~/.codex/skills/`, so the loop symlinks it there during setup. Nothing to install by
hand, and it never appears in Claude's skill picker.

## Requirements

`codex-orchestrator` and `mattpocock-skills` are declared dependencies and install automatically.
You also need the `codex` CLI (authenticated), `gh`, and `python3` on the machine.

## Adding a skill

1. Create `skills/<category>/<name>/SKILL.md`.
2. Add `"./skills/<category>/<name>"` to the `skills` array in `.claude-plugin/plugin.json`.
3. Bump `version` in `plugin.json` so installed copies pick it up.
4. `claude plugin validate . --strict`

The array is required rather than optional: the default scan only finds `skills/<name>/SKILL.md`
one level deep, so nothing under a category directory is auto-discovered. The upside is that
`skills/in-progress/` is a real staging area — a skill can sit there version-controlled but
unshipped until you list it.

## What it expects from a repo

Nothing hardcoded — the skill names no repo, branch, service, or interpreter. It reads them:

- **Trunk** from `origin/HEAD`, and it stops to ask when that is unset. It never infers the trunk
  from a branch name and never falls back to `main`/`master` — a repo can carry an abandoned `main`
  next to a differently-named default branch, and diffing against the wrong one hands the reviewer
  months of unrelated commits.
- **Interpreter, test, lint and forbidden build commands** from the repo's `CLAUDE.md`, including
  any pre-existing lint baseline it records.
- **Issues** from `docs/agents/issue-tracker.md` if present, otherwise `gh` / the PRD path you pass.
- **Live services** — if a timer, worker, or dev server runs off the working tree, the skill offers
  to stop it or build in a worktree rather than deciding for you. Name those units in your
  `CLAUDE.md` so it does not have to guess.

## Credits

The build and review are [Matt Pocock's](https://github.com/mattpocock/skills) `/implement`,
`/tdd` and `/code-review` (MIT), installed from upstream and called unchanged. This repo adds the
Codex loop and the fan-out around them. `codex-skill/references/smell-baseline.md` is the one
derived file, carried because Codex cannot read Claude's skills.

`codex-orchestrator` is [alexzh3's](https://github.com/alexzh3/codex-orchestrator), MIT, fetched
from upstream as a dependency rather than vendored.
