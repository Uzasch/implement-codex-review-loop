---
name: implement-codex-review-loop
description: Implement a PRD or set of issues, then harden it in a Claude↔Codex review loop — Claude implements and self-reviews, Codex reviews independently in a fresh session each round, Claude triages every finding against the ADRs and the originating issue, fixes what it accepts, and answers back. Repeats until Codex approves or the round cap is hit. Big issues can fan the build out across parallel agents first. Ends with a drift report of everything built beyond the issue and beyond the ADRs.
argument-hint: "<issue #s / PRD path> [--rounds N] [--fanout]"
disable-model-invocation: true
---

Implement the work, then drive it through an independent Codex review loop until Codex signs off.
You stay the orchestrator for the whole run — same Claude session start to finish, and a **fresh**
Codex session every round, with continuity carried in the prompt rather than in a resumed thread.

**Do not delegate the loop to a subagent.** The triage step needs your full context.

## Arguments

- The work: issue numbers (`#76 #77`), a PRD path, or a plain description. Required — ask if absent.
- `--rounds N` — max Codex review rounds. Default **5**. Minimum **2** (see *Exit conditions*).
- `--fanout` — standing consent to split the build across parallel agents when the issue clears the
  Phase 1b gate. Without it, size the issue, propose the split, and **stop for an answer** — never
  spend a fleet on your own initiative.

## Setup

You are already where the work happens. This skill does **not** create branches or worktrees, and
does not resolve a trunk to diff against — the user has put you in the right tree.

```bash
SELF_ROOT="${CLAUDE_PLUGIN_ROOT:-$(ls -d ~/.claude/plugins/cache/*/uzasch-skills/*/ 2>/dev/null | sort -V | tail -1)}"
REPO="$(git rev-parse --show-toplevel)"
BASE="$(git rev-parse HEAD)"     # capture BEFORE writing a line of code
RUN="$REPO/.codex-review/<slug>" # <slug> = the issue numbers or PRD name
mkdir -p "$RUN"
```

`BASE` is simply where you started, unless the user names a commit — then use theirs. Everything the
loop reviews is `BASE...HEAD`. Add `/.codex-review/` to the local git exclude
(`.git/info/exclude`), never `.gitignore`.

This skill needs the `implement`, `tdd` and `code-review` skills from `mattpocock-skills`, which
installs as a declared dependency. If `implement` is not available, stop and say so rather than
improvising a build phase — Phase 1 is deliberately thin because that skill owns it.

Publish this skill's Codex-side reviewer so `codex` finds it by name. Idempotent:

```bash
mkdir -p ~/.codex/skills
ln -sfn "${SELF_ROOT%/}/skills/engineering/implement-codex-review-loop/codex-skill" ~/.codex/skills/code-review-codex-loop
test -r ~/.codex/skills/code-review-codex-loop/SKILL.md   # must pass before any review round
```

Some repos carry a permanently dirty tree — generated logs, agent memory, data backups. **Stage by
explicit path; never `git commit -a`.** Sweeping those in puts them inside `BASE...HEAD` and Codex
reviews work that was never yours. If the issue genuinely needs a file that was already dirty, ask
before claiming it.

**Pin the spec now.** Fetch the issues (`gh issue view N`) or read the PRD, and write the acceptance
criteria to `$RUN/spec.md`. Codex reviews against this text; if it is vague the loop cannot
converge. If there is no spec at all, say so in the Codex prompt rather than letting Codex invent one.

## Phase 1 — Implement

Invoke the **`implement`** skill from `mattpocock-skills` and follow it. It uses `/tdd` at
pre-agreed seams, typechecks and runs tests as it goes, ends with its own two-axis `/code-review`,
and commits. Let it do all of that — Codex must review *self-reviewed, committed* work, not a first
draft.

Two constraints it does not know about:

- Stage by explicit path, per *Setup*.
- **Its `/code-review` output is yours alone.** Codex never sees it. A primed reviewer is not an
  independent one.

For an issue too big for one context, run **Phase 1b** first and come back here.

## Phase 1b — Fan out the build (opt-in, off by default)

Phase 1 is one context. When the issue exceeds it, split the build across parallel Claude agents via
the Workflow tool — but only after you have made the split safe, and only when the user says go.

**Eligible** when two of: ≥12 files touched; ≥3 distinct subsystems; ≥5 acceptance criteria of which
≥4 are testable without the others existing. Or alone: ≥8 near-identical units. Estimate with
`git grep`, not vibes.

Then the **shared-seam test, which overrides eligibility.** Two slices are really one if both must
change the same function signature, table schema, JSON contract, or shared resolver. Refuse to fan
out when the criteria form a chain (B's test cannot be written until A's schema exists), or when
more than half the estimated diff sits in files two slices would touch. One deep behaviour change
with many call sites is one piece of thinking; build it serially.

**Write the seam yourself and commit it before the fan-out** — schema, contract, resolver,
migration, interface types. Only the leaves fan out. That commit is what makes the slices
independent; without it there is no fan-out, only a merge.

**Never auto-escalate.** Print the estimate — slice count, agent count, what happens afterwards —
and stop until the user says go.

One workflow, **one stage, `pipeline`, max 6 slices**. Not `parallel()`: the seam commit already
landed, so no slice needs another's result and a barrier buys only latency. No second stage: a
subagent re-running its own tests is not verification. Six is the ceiling because *you* must still
read `BASE...HEAD` end to end.

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

Write each brief to disk first and point the agent at it — the script has no filesystem API. Every
brief carries: the slice's acceptance criteria verbatim; its **exclusive file glob** ("touch anything
else and return `blocked` with the path"); the named test per criterion, written first, with its
failing output pasted verbatim; the exact interpreter, test and lint commands from the repo's
`CLAUDE.md`, including any lint baseline it records and any build command it forbids; and **run no
git write command — Claude commits** (the index lock races).

Use `isolation:'worktree'` **only** when a slice runs a repo-wide mutating tool or two slices rewrite
call sites in the same file; then merge `--no-ff` in slice order, never `-X ours/theirs`, and after
three hand-resolved conflicts finish serially. A conflict means the partition was wrong. Worktrees
do not fix a shared database, port, or dataset — serialise those slices.

### Before Codex sees anything — you verify, alone

Slice test runs are void the moment slices integrate.

- **Leak check** — `git diff --name-only` against the seam commit, minus the union of every slice's
  glob. An unclaimed path is a partition leak: revert it or adopt it deliberately. A subagent's own
  out-of-glob self-report is not evidence.
- **Red-proof replay** — re-run every test a slice added, at the seam commit, and confirm it fails
  there. One that passes against pre-slice code is dead; rewrite it.
- Full suite, typecheck, and each linter the repo documents — as **deltas** against any baseline it
  records. **The workflow must never run the full suite**; parallel agents share databases and ports.
- Read every hunk of `BASE...HEAD`.

A slice returning `files_written: []` is failed, not "nothing to do". So is a null agent — never
fabricate a handoff. Build that slice inline; **never re-fan a failed slice.** A slice returning
`blocked` stops Phase 2 until you resolve it. A slice whose diff exceeds roughly twice its estimate
gets read line by line.

Then commit per slice in slice order, run `/code-review` over the merged result, and enter Phase 2
with one linear `BASE...HEAD`.

## Phase 2 — The Codex loop

Each round is a directory and five steps:

```bash
N=01                       # round number, zero-padded
DIR="$RUN/round-$N"
mkdir -p "$DIR"
```

### 2.1 Write the prompt

Write `$DIR/prompt.md`. Every round opens with:

> Use your `code-review-codex-loop` skill for this review.

That skill was symlinked into `~/.codex/skills/` during *Setup*, so Codex discovers it by name. Do
not paste its contents into the prompt.

**Round 1** gives Codex this and only this:

- The goal and the full acceptance criteria (`$RUN/spec.md`).
- The exact SHAs: `git diff <BASE>...<HEAD>`, and the commit list.
- Binding constraints: whatever `CLAUDE.md`/`AGENTS.md` records — lint baselines, the interpreter,
  forbidden commands — and the relevant ADR numbers if the repo keeps them.

**After a fan-out**, round 1 also carries two *structural facts* — not claims, and not recoverable
from the diff: the **slice map** (`<agent> → <files>`), and a **seams list** of every contract
crossing a slice boundary, with the instruction *"treat inconsistency across slices as a Standards
finding even where each slice is internally correct."* Two slices agreeing on a name and disagreeing
on its meaning is the defect class fan-out manufactures.

Withhold from round 1: your Phase 1 `/code-review` output, every implementation handoff, all test
claims, and any tentative conclusion. Independence is preserved by withholding the reports, not the
map.

**Rounds 2+** are a fresh reviewer with no memory, so the prompt carries the history:

- `## Prior findings` — the previous round's `handoff.md`, **verbatim**.
- `## What changed in response` — per finding, one line: `F<n> — FIXED in <sha> (<what changed>)` or
  `F<n> — REJECTED: <reason> (<ADR / issue / project-rule citation>)`.
- The new commit SHAs and the new `BASE...HEAD` range.
- The instruction to restate every prior finding as `RESOLVED` / `STILL OPEN` / `WITHDRAWN`, then
  re-review the new commits.

Write *"a previous review produced the findings below"*, never *"you reviewed this"*. Telling a
fresh model it did something it cannot recall makes it hedge or invent the missing context. And
state rejections with their reason — omit one and the next reviewer raises it again.

Never edit or re-send a prompt already used. Each round's prompt is immutable.

### 2.2 Launch

Every round is a fresh session — round 1 and round 5 use the identical command:

```bash
codex exec --json --output-last-message "$DIR/handoff.md" \
  -s workspace-write -c approval_policy=never -C "$REPO" \
  - < "$DIR/prompt.md" > "$DIR/events.jsonl"
```

Never `codex exec resume`. A resumed session carries the reviewer's prior reasoning as hidden state
you cannot inspect or correct; a fresh session with the history in `prompt.md` costs a re-read of the
diff and gives you inputs you can see. `workspace-write` is deliberate even though the reviewer must
not edit — repo checks need to write their normal temporary output. Confirm the tree is clean
afterward.

`handoff.md` is the verdict and normally the only file you read. Open `events.jsonl` only when the
handoff is empty or truncated. An empty handoff on a non-zero exit is a failed round — never
fabricate one.

### 2.3 Triage every finding — this is the step that matters

Codex's handoff is a set of **claims**. Nothing in it changes the code until you have checked it.
For each finding, in order:

0. **If you did not personally write that code** (a fan-out ran), read the cited hunk **and the
   function around it** cold from the repository before forming any verdict. You have no memory of
   this code to judge from.
1. **Reproduce it.** Read the cited `file:line`. Run the cited command yourself. A finding you
   cannot reproduce is `REJECTED: not reproducible` — say what you observed instead.
2. **Check it against the ADRs.** Grep `docs/adr/`. A finding asking you to undo a live ADR is
   `REJECTED` citing that ADR number — Codex has not read the argument behind it. If the finding
   reveals the ADR itself is wrong, note it as a follow-up; that is not this loop's job.
3. **Check it against the issue.** A finding demanding behaviour the issue did not ask for is
   `REJECTED: out of scope` — or `DEFERRED` and filed as a new issue when genuinely worth doing.
4. **Check it against the project rules.** Where the repo records a pre-existing lint baseline, a
   finding demanding a clean run of that linter — or a fix in a file this ticket does not touch — is
   `REJECTED`, citing the baseline.
5. Otherwise: **ACCEPT**.

Append every verdict to `$RUN/findings.md`, one line each:

```text
F3 | r1 | REJECTED | asks to undo ADR 0041 (bucket stock never feeds cadence)
F4 | r1 | ACCEPTED | fixed in a1b2c3d
```

That file is the loop's only durable state, and it is load-bearing: the rejection reason you write
goes back to Codex verbatim next round, so write it to settle the point. Do not grow a schema around
it.

A `suggestion` never blocks; take it only if cheap and clearly right.

### 2.4 Fix, verify, commit

Implement every ACCEPTED finding. Nothing else — no unrelated cleanups mid-loop.

**Fix rounds are serial, always — never fan out a fix round.** Accepted findings land wherever the
defect is, so disjoint ownership no longer holds; and the `FIXED in <sha>` line you send Codex has
to be one you wrote, with nothing unread between it and the diff Codex re-verifies.

Typecheck and run the touched tests as you go; run the full suite before committing. Read each
linter as a **delta**: where `CLAUDE.md` records a baseline, your work is clean when it adds no new
rule and no new count — not when the run is green. Honour any command the repo forbids; some build
steps overwrite an artefact that is actually being served.

Commit with the round in the message (`fix(review r2): …`).

### 2.5 Exit conditions

Continue to the next round unless one fires:

- **Approved.** Codex's last line is `VERDICT: APPROVED`, you have no accepted finding still
  unfixed, and at least **2** rounds have run. A round-1 approval is not enough: run one more telling
  Codex what changed, so it approves the final state rather than the state it first saw.
- **Round cap.** `--rounds N` reached. Stop and report the still-open findings.
- **Deadlock.** Codex re-raises a finding you already rejected on a cited ADR or scope ground, a
  second time, with no new evidence. Stop and put the disagreement to the user with both positions
  and the citation.
- **Blocked.** Codex reports `Status: blocked`, or a finding needs a decision only the user can make.

Between rounds, do not reset, rebase, or squash — Codex reviews a moving HEAD off a fixed `BASE`,
and rewriting history under it invalidates every SHA you have sent.

## Phase 3 — The drift report (always, even when nothing drifted)

Walk the **whole** final diff one file at a time and sort every change into three buckets. The diff
is the source of truth, not your memory of what you meant to do — over five rounds, work accretes.

```bash
git diff BASE...HEAD --stat
git diff BASE...HEAD
```

1. **Asked for** — quote the acceptance criterion it satisfies.
2. **Outside the issue** — nobody asked for it. Mostly accepted Codex findings, the rest things you
   decided to do while in there: an extracted helper, an adjacent bug, a rename, a config touched.
3. **Outside the ADRs** — it settles something `docs/adr/` does not cover, or departs from one that
   does. Grep per subsystem; don't assume from memory. A change can be in both 2 and 3.

Bucket 3 bites later. A decision that never became an ADR is one the next agent cannot find, and a
departure from a live ADR silently makes the ADR wrong. For each, say which: **new ground** or
**departs from ADR NNNN**. Then say whether it still stands or you would take it back, and offer to
file the bucket-3 items as ADR follow-ups.

**After a fan-out**, add one line: the gap between what slices self-reported as `unresolved` and what
bucket 2 actually turned up. Anything in bucket 2 no slice declared is *"unattributed — surfaced by
the diff, not self-reported."* That rates the fan-out itself.

## The final message to the user

Plain language, for someone who has not read the diff. No jargon, no SHAs in the prose, no finding
numbers as though they mean something to the reader. Short sentences. Cover, in order:

**What you set out to do** — one sentence, in the issue's own terms.

**What you built** — the behaviour that now exists, not the files that changed.

**How many rounds it took and where it landed** — sign-off, round cap, deadlock, or blocked. Straight.

**What Codex asked for and what you did** — grouped, not enumerated. "It found three real problems, I
fixed all three. It asked for two more things I didn't do, because…" Reasons in ordinary words: *the
issue didn't ask for it*, *we already decided this the other way in decision record 0041*, *I ran the
command it cited and got a different result*.

**What changed that nobody asked for** — bucket 2. One line each: what it is, why it's there. This is
the section people skip, and the one that explains why the change is bigger than the ticket.

**What this decided that wasn't already decided** — bucket 3. What got settled, new ground or a
departure. Say plainly these are worth recording as ADRs, and offer to write them.

**What's still open** — unfixed findings, anything deferred (with its issue number), anything needing
a user decision.

If buckets 2 and 3 are empty, say so in one line. An explicitly empty drift report is a useful
result; a missing one reads as an oversight.
