# Loop Protocol (rounds 2+)

Every round is a **fresh session**. You did not conduct the earlier review and hold no memory of
it — the prompt's `## Prior findings` section is that review, quoted verbatim, and it is the only
record you have. Treat it as your own prior work, and read this before re-judging anything.

The prompt also tells you: the new commit SHAs, the new `base...head` range, and — per prior
finding — either `FIXED in <sha>` or `REJECTED: <reason> (<citation>)`.

## Re-judging prior findings

**Read the repository before you re-judge.** Claude's message is a claim; the diff is the fact.
Check the cited commit actually contains the fix it describes.

Restate **every** prior finding with exactly one status:

- `RESOLVED` — you verified the fix in the code. Not "Claude says it's fixed".
- `STILL OPEN` — with what specifically is still wrong, and fresh evidence.
- `WITHDRAWN` — you were wrong, or Claude showed it was out of scope.

A finding never silently disappears. If it isn't restated, the loop has lost track of it.

## Accepting a rejection

Claude rejects findings on four grounds. When the citation genuinely covers the finding, mark it
`WITHDRAWN` and **do not raise it again**:

- **A live ADR** under `docs/adr/`. The decision has already been argued and made; you have not
  read the argument behind it. Read the cited ADR before disputing it.
- **The issue's stated scope.** Behaviour the spec didn't ask for is not a defect.
- **A documented project rule** — a lint baseline, a forbidden command, an interpreter choice.
- **Not reproducible.** If Claude ran your cited command and observed something else, either
  produce a sharper repro or withdraw.

Dispute a rejection only with **new evidence that the citation does not apply** — for example,
the ADR covers a different code path, or the spec line does cover the behaviour after all. Say
which. Re-raising a rejected finding without that is a loop failure: it burns a round and cannot
converge.

## Reviewing the new work

Review the new commits with the same rigour as round 1 — fixes introduce defects, and a rushed
patch to close a `blocker` is exactly where the next one appears.

**Do not escalate scope.** Opening a fresh axis of criticism late in the loop, on code that was
already in front of you in round 1, never converges. Raise something new only when it is a
genuine `blocker` you can evidence, or when the new commits introduced it.

## Converging

Approve as soon as the bar is met: zero open `blocker` and zero open `major` on either axis.
`minor` findings and `suggestion`s do not block — list them as open and approve anyway, noting
them under `## Caveats / Blockers` so Claude can carry them forward as follow-ups.

Withholding approval over taste is how a review loop stops being worth its cost.
