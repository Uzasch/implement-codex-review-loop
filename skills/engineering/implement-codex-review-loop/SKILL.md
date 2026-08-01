---
name: implement-codex-review-loop
description: Implement a PRD or set of issues, then harden it in a Claude↔Codex review loop — Claude implements and self-reviews, Codex reviews independently in one persistent session, Claude triages every finding against the ADRs and the originating issue, fixes what it accepts, and answers back. Repeats until Codex approves or the round cap is hit. Big issues can fan the build out across parallel agents first. Ends with a drift report of everything built beyond the issue and beyond the ADRs.
argument-hint: "<issue #s / PRD path> [--rounds N] [--base <ref>] [--fanout]"
disable-model-invocation: true
---

Implement the work, then drive it through an independent Codex review loop until Codex signs
off. You stay the orchestrator for the whole run — same Claude session start to finish, and
**one** Codex session resumed every round so the reviewer remembers what it already asked for.

**Do not delegate the loop to a subagent.** The triage step needs your full context.

## Arguments

- The work: issue numbers (`#76 #77`), a PRD path, or a plain description. Required — ask if absent.
- `--rounds N` — max Codex review rounds. Default **5**. Minimum **2** (see *Exit conditions*).
- `--base <ref>` — the fixed point the review diffs against. See *Resolving BASE* below; you
  rarely need to pass this.
- `--fanout` — the user's standing consent to split the build across parallel agents when the
  issue clears the Phase 1b gate. Without it, size the issue, propose the split, and **stop for an
  answer** — never spend a fleet on your own initiative.

## Setup

Resolve these once and keep them for the whole run:

```bash
# The `*` is the marketplace name, and it is deliberate: codex-orchestrator installs under
# whichever marketplace pulled it in — its own when installed directly, or `uzasch` when
# auto-installed as this plugin's dependency.
PLUGIN_ROOT="$(ls -d ~/.claude/plugins/cache/*/codex-orchestrator/*/ 2>/dev/null | sort -V | tail -1)"
SELF_ROOT="${CLAUDE_PLUGIN_ROOT:-$(ls -d ~/.claude/plugins/cache/*/uzasch-skills/*/ 2>/dev/null | sort -V | tail -1)}"
REPO="$(git rev-parse --show-toplevel)"
```

`PLUGIN_ROOT` must be non-empty — if it is, the `codex-orchestrator` plugin is not installed and
this skill cannot run. Say so and stop; do not improvise the contract it owns.

This skill also needs the `implement`, `tdd` and `code-review` skills from `mattpocock-skills`,
which installs as a declared dependency. If `implement` is not available, stop and say so rather
than improvising a build phase — Phase 1 is deliberately thin because that skill owns it.

Publish this skill's own Codex-side reviewer so `codex` can find it by name. Idempotent, safe to
re-run every time:

```bash
mkdir -p ~/.codex/skills
ln -sfn "${SELF_ROOT%/}/skills/engineering/implement-codex-review-loop/codex-skill" ~/.codex/skills/code-review-codex-loop
test -r ~/.codex/skills/code-review-codex-loop/SKILL.md   # must pass before any review round
```

Then perform **Run Initialization** exactly as `$PLUGIN_ROOT/skills/workflow/SKILL.md` defines
it: add `/.codex-orchestrator/` to the local git exclude (never `.gitignore`), confirm both
exclude checks pass, and capture the baseline HEAD, branch, and `git status --short
--untracked-files=all`. Treat already-dirty paths as pre-existing user work — do not claim them.

This working tree is **normally dirty** (brand `log.md` files, `MEMORY.md`, data backups). Stage
by explicit path and never `git commit -a` — sweeping those into a commit puts them inside
`BASE...HEAD`, and Codex will review, and raise findings on, work that was never yours. If the
issue genuinely needs to touch a file that was already dirty, ask the user before claiming it.

Create the run and its journal:

```bash
RUN_ID="run-$(date -u +%Y%m%d)-01"          # bump the suffix if it already exists
RUN_DIR="$REPO/.codex-orchestrator/runs/$RUN_ID"
mkdir -p "$RUN_DIR/codex-review-01"
```

Append `run_started` to `$RUN_DIR/journal.jsonl`. Read
`$PLUGIN_ROOT/docs/orchestration-contract.md` once before writing any journal entry — it owns
every field and the closure semantics. Record `claude_version` and `codex --version`.

### Resolving BASE

**Never infer the trunk from a branch name, and never fall back to `main`/`master`.** On
`video-compilation2.0` the trunk is `native-ubuntu-pc1`; a `main` branch still exists there and is
hundreds of commits stale, so diffing against it would hand Codex months of unrelated work as
though it were this change. Resolve it, don't assume it:

```bash
TRUNK="$(git rev-parse --abbrev-ref origin/HEAD 2>/dev/null | sed 's|^origin/||')"
git rev-parse --verify "$TRUNK"
```

If `TRUNK` comes back empty (no `origin/HEAD` on this clone), **ask the user which branch is the
trunk** — do not guess, and do not run `git remote set-head` to invent an answer. One wrong trunk
costs a whole review round.

Then, unless `--base` was passed:

- **On the trunk itself** (the usual case here — work happens directly on `native-ubuntu-pc1`):
  `BASE="$(git rev-parse HEAD)"`, captured **before you write a line of code**. The loop reviews
  exactly what this run builds.
- **On a feature or worktree branch** (`issue/NN`, `worktree/*`, `feat/*`):
  `BASE="$(git merge-base "$TRUNK" HEAD)"`.

Sanity-check before launching anything: `git rev-list --count BASE..HEAD` should match the work in
front of you. Hundreds of commits means you resolved the wrong trunk — stop and fix it rather than
paying Codex to read them.

### Working directly on the trunk — the live-services hazard

Seven systemd units run **from this checkout**, and `video-compilation-cadence-dev.timer` launches
a fresh one-shot every 30 minutes that imports Python straight off the working tree. There is no
deploy step: half-finished code on disk under `backend/` or `compilation-agent/scripts/` can be
picked up by a real build-and-upload within half an hour.

Before Phase 1 (and *always* before a Phase 1b fan-out, where several agents leave the tree
mid-edit at once), check what the change touches:

```bash
systemctl list-timers --all --no-pager | grep -i cadence
```

- **Touches nothing the services import** — frontend, docs, tests, a brand-new module nothing
  imports yet: carry on, no precaution needed.
- **Touches `backend/` or `compilation-agent/scripts/`**: tell the user the timer is live and offer
  the two options — pause it for the duration
  (`sudo systemctl stop video-compilation-cadence-dev.timer video-compilation-shorts-cadence-dev.timer`,
  restarting it at Phase 3), or build in an isolated worktree via the `worktree` skill and merge
  back. **Do not choose for them, and never leave a timer stopped without saying so.**
- **A Phase 1b fan-out over backend code**: use a worktree. Up to six agents writing into the live
  tree guarantees windows where imports are broken, and the cadence one-shot does not care that you
  were mid-build.

Rollback is also sharper here. With no worktree there is nothing to delete: abandoning the work
means `git revert`, or `git reset` to `BASE` — and **never `git reset --hard` in this tree**, which
would destroy the pre-existing uncommitted brand-log edits along with your own.

Record `BASE` (the resolved SHA) in the journal goal line. Everything the loop reviews is
`BASE...HEAD`.

**Pin the spec now.** Fetch the issues via `docs/agents/issue-tracker.md` (`gh issue view N`) or
read the PRD, and write the acceptance criteria into a `task` entry. Codex reviews against this
text; if it is vague, the loop cannot converge. If there is no spec at all, say so explicitly in
the Codex prompt rather than letting Codex invent one.

## Phase 1 — Implement

Invoke the **`implement`** skill from `mattpocock-skills` and follow it. It uses `/tdd` at
pre-agreed seams, typechecks and runs tests as it goes, ends with its own two-axis `/code-review`,
and commits. Let it do all of that — Codex must review *self-reviewed, committed* work, not a first
draft.

Two constraints it does not know about, which still bind here:

- Stage by explicit path. The dirty-tree rule from *Setup* holds: never `git commit -a`.
- **Its `/code-review` output is yours alone.** The *Withhold from round 1* rule below means Codex
  never sees it. A primed reviewer is not an independent one.

Record the resulting commits. Append `verification` entries for the checks you actually ran
(typecheck, test suite) — your observed output, not a claim.

For an issue too big for one context, run **Phase 1b** first and come back here.

## Phase 1b — Fan out the build (opt-in, off by default)

Phase 1 is one context. When the issue exceeds it, split the build across parallel Claude agents
via the Workflow tool — but only after you have made the split safe, and only when the user says
go.

### Decide, then ask

Estimate with `git grep`, not from vibes. **Eligible** when two of: ≥12 files touched; ≥3 distinct
subsystems (`backend/api`, `backend/services`, `compilation-agent/scripts`, `frontend/src`, a BQ
migration, `docs/adr`); ≥5 acceptance criteria of which ≥4 are testable without the others
existing. Or, on its own: ≥8 near-identical units — a per-brand sweep, N call sites of one rename.

Then run the **shared-seam test, which overrides eligibility.** Two candidate slices are really one
slice if both must change the same function signature, table schema, JSON contract, or shared
resolver (`scripts/_pathresolve.py` — ADR 0040 made it the *only* path translator, so every caller
hangs off one seam). Refuse to fan out when the criteria form a chain (B's test cannot be written
until A's schema exists), or when more than half the estimated diff sits in files two slices would
touch. One deep behaviour change with many call sites is one piece of thinking; build it serially.

**Write the seam yourself and commit it before the fan-out** — schema, contract, resolver,
migration, interface types. Only the leaves fan out. That commit is what makes the slices
independent; without it there is no fan-out, only a merge.

**Never auto-escalate.** The Workflow tool requires explicit user opt-in. Print the estimate —
slice count, agent count, what you will do afterwards — and stop until the user says go.

### The workflow

One workflow, **one stage, `pipeline`, max 6 slices**. Not `parallel()`: the seam commit already
landed, so no slice needs another's result and a barrier buys only latency. No second stage: a
subagent re-running its own tests is not verification, and you re-run everything yourself anyway.
Six is the ceiling because *you* must still read `BASE...HEAD` end to end — past that, your read is
the bottleneck the fan-out existed to remove.

Write each slice brief to disk first and point the agent at it: the script has no filesystem API,
and this keeps the prompt-of-record immutable and journal-linkable.

```js
export const meta = { name: "impl-fanout", description: "Build disjoint slices of one issue", phases: ["build"] };
const SLICE = { type:"object", required:["slice","status","files_written","tests_added","commands","unresolved"],
  properties:{ slice:{type:"string"}, status:{enum:["done","blocked"]},
    files_written:{type:"array",items:{type:"string"}},
    tests_added:{type:"array",items:{type:"object",required:["path","name","red_output"],
      properties:{path:{type:"string"},name:{type:"string"},red_output:{type:"string"}}}},
    commands:{type:"array",items:{type:"object",required:["cmd","exit","tail"],
      properties:{cmd:{type:"string"},exit:{type:"integer"},tail:{type:"string"}}}},
    unresolved:{type:"array",items:{type:"string"}} } };
const out = await pipeline(args.slices, s =>
  agent(`Read your brief at ${s.brief} and follow it exactly. Report nothing you did not observe.`,
        { label: s.id, phase: "build", schema: SLICE, effort: "high" }));
log(`returned ${out.filter(Boolean).length}/${args.slices.length}`);
return out.filter(Boolean);
```

Every brief carries: the slice's acceptance criteria verbatim; its **exclusive file glob** ("touch
anything else and return `blocked` with the path"); the named test per criterion, written first,
with its failing output pasted verbatim; `backend/venv/bin/python`; the `CLAUDE.md` lint baselines;
never `npm run build`; and **run no git write command — Claude commits** (the index lock is a
shared resource and concurrent commits race it).

Everything happens in the one repo worktree. Use `isolation:'worktree'` **only** when a slice runs
a repo-wide mutating tool (formatter, codemod, migration) or two slices rewrite call sites in the
same file; then merge `--no-ff` in slice order, never `-X ours/theirs`, and after three
hand-resolved conflicts abandon the rest and finish serially. A conflict means the partition was
wrong. Worktrees do not fix a shared DB, port, or BQ dataset — serialise those slices instead.

### Journal

One `task` per slice with its glob as `files`, disjoint by construction. One `execution` per slice
under `claude-impl-01…NN` / `execution-01`, `provider: claude`, `role: implementation`,
`event_source: "claude"`, `events` omitted, `worktree` = `$REPO`, `head` = the seam commit — all
appended **before** the Workflow call. Afterwards write each returned object to its `handoff.md`
and append one `execution_result`. Subagents never touch `journal.jsonl`; one orchestrator owns it.
`codex-review-01` is untouched.

### Before Codex sees anything — you verify, alone

Slice test runs are void the moment slices integrate.

- **Leak check** — `git diff --name-only` against the seam commit, minus the union of every
  `task.files`. An unclaimed path is a partition leak: revert it, or adopt it with a `decision`
  entry. A subagent's own out-of-glob self-report is not evidence.
- **Red-proof replay** — re-run every test a slice added, at the seam commit, and confirm it fails
  there. One that passes against pre-slice code is dead; rewrite it. This is the seam where you
  stay the verifier.
- Full suite, typecheck, and `ruff check --statistics` / `cd frontend && npx eslint src/` as deltas
  against the `CLAUDE.md` baseline tables. **The workflow must never run the full suite** —
  parallel agents share the DB, ports, and BQ credentials.
- Read every hunk of `BASE...HEAD`.

A slice returning `files_written: []` is `failed`, not "nothing to do". So is a null agent (no
handoff — never fabricate one). Build that slice inline; **never re-fan a failed slice.** A slice
returning `blocked` keeps its task blocked, and Phase 2 does not start until you resolve it. A
slice whose diff exceeds roughly twice its estimate gets read line by line.

Only your own observations become `verification` entries; a subagent's `commands` array goes in
the `execution_result` and nowhere else. Then commit per slice in slice order, run `/code-review`
over the merged result, and enter Phase 2 with one linear `BASE...HEAD`.

## Phase 2 — The Codex loop

Rounds are `execution-01`, `execution-02`, … under the single agent `codex-review-01`, in one
Codex session. Each round is five steps.

### 2.1 Write the prompt

```bash
N=01                                              # round number, zero-padded
EXEC_DIR="$RUN_DIR/codex-review-01/execution-$N"
mkdir -p "$EXEC_DIR"
```

Write `$EXEC_DIR/prompt.md`. Every round opens with:

> Use your `code-review-codex-loop` skill for this review.

That skill ships inside this skill at `$SELF_ROOT/skills/engineering/implement-codex-review-loop/codex-skill/` and was
symlinked into `~/.codex/skills/` during *Setup*, so Codex discovers it by name. Do not paste its
contents into the prompt.

**Round 1** then gives Codex, and only this:

- The goal and the full acceptance criteria (the pinned spec text or issue body).
- The exact SHAs: `git diff <BASE>...<HEAD>`, and the commit list.
- Binding constraints it must respect: the repo's `CLAUDE.md` lint baselines, relevant ADR
  numbers under `docs/adr/`, the Python interpreter, the forbidden commands.

**After a Phase 1b fan-out**, round 1 also carries two *structural facts* — not claims, and not
recoverable from the diff:

- The **slice map**: `<agent> → <files>`.
- A **seams list**: every contract that crosses a slice boundary, with its owning slice — plus the
  instruction *"treat inconsistency across slices as a Standards finding even where each slice is
  internally correct."* Two slices agreeing on a name and disagreeing on its meaning is the defect
  class fan-out manufactures, and Codex cannot see the boundary in a merged diff.

Withhold from round 1, per `$PLUGIN_ROOT/skills/orchestrate/references/review.md`: your Phase 1
`/code-review` output, every implementation handoff (yours and every subagent's), all test claims,
and any tentative conclusion. A primed reviewer is not an independent one. Independence is
preserved by withholding the reports, not the map.

**Rounds 2+** additionally give Codex:

- The new commit SHAs since its last review, and the new `BASE...HEAD` range.
- Per prior finding, one line: `F<n> — FIXED in <sha> (<what changed>)` or
  `F<n> — REJECTED: <reason> (<ADR / issue / project-rule citation>)`.
- The instruction to restate every prior finding as `RESOLVED` / `STILL OPEN` / `WITHDRAWN` and
  to re-review the new commits.

Never edit or re-send a prompt already used — each execution's prompt is immutable.

### 2.2 Launch

Append the `execution` entry to the journal **before** launching (absolute `worktree`, full
`head`, `branch`, `provider: codex`, `role: review`, `event_source: exec`, prompt/events/handoff
paths). In-flight work has to survive context loss.

Round 1 — fresh session:

```bash
codex exec --json --output-last-message "$EXEC_DIR/handoff.md" \
  -s workspace-write -c approval_policy=never -C "$REPO" \
  - < "$EXEC_DIR/prompt.md" > "$EXEC_DIR/events.jsonl"
```

Capture the session id from the first event and keep it for the whole run:

```bash
TID="$(python3 -c "import json,sys;[print(json.loads(l)['thread_id']) for l in open(sys.argv[1]) if json.loads(l).get('type')=='thread.started']" "$EXEC_DIR/events.jsonl")"
```

Rounds 2+ — **resume that same session** (note the different flag order: `resume` after the
global flags, and `--json` after `resume`):

```bash
codex exec -C "$REPO" -s workspace-write -c approval_policy=never \
  resume --json --output-last-message "$EXEC_DIR/handoff.md" \
  "$TID" - < "$EXEC_DIR/prompt.md" > "$EXEC_DIR/events.jsonl"
```

Never use `--ephemeral`, and never start a second reviewer agent — the persistent session is the
point of this loop.

If a run looks stalled or the handoff is empty, use the bundled monitor rather than reading raw
events into context:

```bash
python3 "$PLUGIN_ROOT/scripts/codex_orch_tools.py" state "$TID" --file "$EXEC_DIR/events.jsonl" --json
```

Append `execution_result` (with `session_id`) once the handoff is on disk. An empty handoff on a
non-zero exit is `failed` — never fabricate one.

### 2.3 Triage every finding — this is the step that matters

Codex's handoff is a set of **claims**. Nothing in it changes the code until you have checked it.
For each finding, in order:

0. **If you did not personally write that code** (Phase 1b ran), read the cited hunk **and the
   function around it** cold from the repository before you form any verdict. You have no memory
   of this code to judge from, and a rejection recorded on remembered evidence is the worst
   failure mode this loop has after fan-out itself.
1. **Reproduce it against the repository.** Read the cited `file:line`. Run the cited command
   yourself. A finding you cannot reproduce is `REJECTED: not reproducible` — say what you
   observed instead. After a fan-out this always means a fresh run, never a recollection.
2. **Check it against the ADRs.** Grep `docs/adr/` for the subsystem. A finding that asks you to
   undo a live ADR is `REJECTED` citing that ADR number — the ADR is a decision already made,
   and Codex has not read the argument behind it. If the finding reveals the ADR itself is
   wrong, that is not this loop's job: note it as a follow-up.
3. **Check it against the issue.** Re-read the pinned acceptance criteria. A finding demanding
   behaviour the issue did not ask for is `REJECTED: out of scope` — or `DEFERRED`, filed as a
   new issue per `docs/agents/issue-tracker.md`, when it is genuinely worth doing later.
4. **Check it against the project rules.** `CLAUDE.md` binds here: a finding that demands a clean
   `ruff`/`eslint` run, or a fix in a file this ticket does not touch, is `REJECTED` citing the
   documented baseline.
5. Otherwise: **ACCEPT**.

Record consequential accepts and every rejection as a `decision` entry (`outcome:
claude_decision` or `consensus`, with the citation in `basis`). The rejection reason you write
here is what goes back to Codex verbatim next round — write it so it actually settles the point.

A `suggestion` never blocks; take it only if it is cheap and clearly right.

### 2.4 Fix, verify, commit

Implement every ACCEPTED finding. Nothing else — do not fold in unrelated cleanups mid-loop.

**Fix rounds are serial, always — never fan out a fix round.** Accepted findings land wherever the
defect is, so the disjoint-ownership precondition no longer holds; and the `F<n> — FIXED in <sha>`
line you send Codex has to be a line you wrote, with nothing unread between it and the diff Codex
re-verifies. The round cap does not rise to absorb fan-out damage either.

Typecheck and run the touched tests as you go; run the full suite before committing. Check
`ruff check --statistics` and `cd frontend && npx eslint src/` against the `CLAUDE.md` baseline
tables — your work is clean when it adds no new rule code and no new count, not when the run is
green. Never run `npm run build`.

Commit with the round in the message (`fix(review r2): …`). Append a `verification` entry per
criterion you checked, with the exact command and what you observed.

### 2.5 Exit conditions

Continue to the next round unless one of these fires:

- **Approved.** Codex's last line is `VERDICT: APPROVED` **and** you have no accepted finding
  still unfixed **and** at least **2** rounds have run. A round-1 approval is not enough on its
  own: run one more round telling Codex what you changed, so it approves the final state rather
  than the state it first saw.
- **Round cap.** `--rounds N` (default 5) reached. Stop and report the still-open findings — do
  not silently continue.
- **Deadlock.** Codex re-raises a finding you already rejected on a cited ADR / scope ground, a
  second time, with no new evidence. Stop the loop and put the disagreement to the user with both
  positions and the citation. Do not keep paying for rounds that cannot converge.
- **Blocked.** Codex reports `Status: blocked`, or a finding needs a decision only the user can
  make. Record it and stop.

Between rounds, do not reset, rebase, or squash — Codex is reviewing a moving HEAD off a fixed
`BASE`, and rewriting history under it invalidates every SHA in the journal.

## Phase 3 — Close

When the loop exits, re-read the whole journal and inspect the final diff (`git diff
BASE...HEAD`) yourself. Then run the close sequence — `validate` → `run_closed` → report — as
`$PLUGIN_ROOT/skills/workflow/SKILL.md` defines it:

```bash
python3 "$PLUGIN_ROOT/scripts/codex_orch_tools.py" validate "$RUN_DIR"
```

Fix omissions by appending only; never rewrite journal history. Copy the validation result
verbatim into a final `run_closed` entry with your own `judgment` (`passed` or `blocked`) —
validation detects omissions, you decide acceptance. Then invoke
`$PLUGIN_ROOT/skills/report/SKILL.md` once to write `report.md`.

## Phase 4 — The drift report (always, even when nothing drifted)

Before you write the final message, walk the **whole** final diff one file at a time and sort
every change into one of three buckets. The diff is the source of truth here, not your memory of
what you meant to do — over a five-round loop, work accretes.

```bash
git diff BASE...HEAD --stat
git diff BASE...HEAD
```

1. **Asked for** — the issue or PRD called for it. Quote the acceptance criterion it satisfies.
2. **Outside the issue** — nobody asked for it. Most of this arrives as accepted Codex findings,
   the rest as things you decided to do while in there: a helper you extracted, an adjacent bug
   you fixed, a rename, a test you added, a config or dependency you touched.
3. **Outside the ADRs** — it settles something the ADRs under `docs/adr/` do not cover, or it
   departs from one that does. Grep `docs/adr/` per subsystem you touched; don't assume from
   memory. A change can land in both bucket 2 and bucket 3.

Bucket 3 is the one that bites later. A decision that never became an ADR is a decision the next
agent can't find, and a departure from a live ADR silently makes the ADR wrong. For each, say
which: **new ground** (no ADR covers this) or **departs from ADR NNNN** (name it).

Then say, per item, whether it still stands or you'd take it back — and offer to file the
bucket-3 items as ADR follow-ups.

**After a Phase 1b fan-out**, add one more line: the gap between what the slices self-reported as
`unresolved` and what bucket 2 actually turned up from the diff. Anything in bucket 2 that no slice
declared is marked *"unattributed — surfaced by the diff, not self-reported."* That gap rates the
fan-out itself and tells the next run whether the partition held. The anchor does not change: the
diff is the source of truth, never a collected subagent report.

## The final message to the user

Write this in **plain language, for someone who has not read the diff**. No jargon, no SHAs in
the prose, no findings numbers as if they mean something to the reader. Short sentences. If a
term only makes sense to an engineer, spend a clause explaining it.

Cover, in this order:

**What you set out to do** — one sentence, in the issue's own terms.

**What you built** — plain description of the behaviour that now exists, not the files that
changed.

**How many review rounds it took, and where it landed** — did Codex sign off, hit the round cap,
deadlock, or block? Say it straight.

**What Codex asked for and what you did about it** — grouped, not enumerated. "It found three
real problems, I fixed all three. It asked for two more things I didn't do, because…" Give the
reason for each rejection in ordinary words: *the issue didn't ask for it*, *we already decided
this the other way in decision record 0041*, *I ran the command it cited and got a different
result*.

**What changed that nobody asked for** — bucket 2, in plain words. One line each: what it is, and
why it's there. This is the section people skip writing, and it's the one that explains why the
change is bigger than the ticket.

**What this decided that wasn't already decided** — bucket 3, in plain words. For each: what got
settled, and whether it's new ground or a departure from an existing decision record. Say plainly
that these are worth recording as ADRs, and offer to write them.

**What's still open** — unfixed findings, anything deferred to a new issue (with its number), and
anything that needs the user to decide.

If buckets 2 and 3 are empty, say so in one line — "nothing changed beyond what the issue asked
for, and nothing here decides anything the decision records don't already cover." An explicitly
empty drift report is a useful result; a missing one reads as an oversight.
