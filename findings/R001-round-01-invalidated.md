# R001 — round-01 is invalid as a measurement (harness fault, not a skills finding)

**Status: both A1 runs discarded.** The findings derived from them survive; the runs do not.

## What happened `[MEASURED]`

Two runners executed **concurrently in the same workspace**. The v1 runner was backgrounded with
`nohup ... &` and did not die when its tool call ended, as I assumed. I reset the workspaces and
launched v2 while v1 was still building.

Commit history proves it:

| workspace | commits |
| --- | --- |
| csharp | `fbc863f` empty baseline 17:02:42 · `09044ae` delivered **17:09:23** · `ad7dc12` delivered **17:28:24** |
| typescript | `8b0db0c` empty baseline 17:02:44 · `157f934` delivered 17:16:33 · `69ba049` 17:18:41 |

Two `delivered` commits per tree, from two different sessions.

The C# v2 agent **saw the other run's work** and said so in its own output:

> "All of it is intact in commit `09044ae` … Their `BaseAddressRewriteHandler` was genuinely better
> than my first draft … and I adopted that technique. If that other run was meant to be the
> deliverable, say so and I'll restore it instead."

So the delivered C# tree is a blend of two independent attempts. That breaks the independence rule the
whole loop rests on.

## Two harness faults behind it

1. **`nohup ... &` from a tool call is not reliably terminated.** Treat a launched runner as alive
   until its own sentinel says otherwise — never assume a tool-call boundary killed it.
2. **The reset did not actually reset.** `rm -rf "$BASE"` failed with *"Device or resource busy"*
   (the still-live v1 runner held the directory), and the follow-up `git init` on a surviving `.git`
   is a **no-op that preserves history**. The workspace looked fresh and was not.

## Fixes

- Reset must **verify** deletion and fail loudly if the directory survives, rather than proceeding.
- Reset must remove `.git` explicitly; a re-`init` over an existing repo keeps every prior commit.
- Before launching a round, assert **zero** live runner processes.
- Assert the baseline commit is the **only** commit before the build starts.

## What survives

The skill findings from the TypeScript debrief are **not** invalidated, because they are claims about
what the skills say versus what is true — independent of which tree was delivered:

- **F004** (`doc/` not in the published package) — independently verified by me against two registries.
- **F005** (workflow gate) — a self-report about that session's own skill loading, internally
  consistent and quoted from its own transcript.
- **F006** (OAuth provider rejection) — already flagged single-session and pending corroboration.

What is lost is the **run as a benchmark artifact**: the delivered trees cannot be scored, and section
B/B2 verdicts cannot be compared across the two languages for this round.
