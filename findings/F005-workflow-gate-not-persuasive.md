# F005 — the workflow gate is skipped, and (on happy-path work) nothing is lost by it

**Severity: DOWNGRADED — observation, not a defect.** **Disposition: `defer`. Do NOT change the
workflow parentheticals on this evidence.**
Evidence: three clean runs, all `final_commits=2`, `debrief_touched=0`, every B2 row `[MEASURED]`.

> ## CORRECTED
>
> An earlier version of this finding called the gate a HIGH-severity failure of the plugin's central
> mechanism and proposed rewriting every step's parenthetical. **Section H says otherwise: across all
> three runs there were ZERO SDK-surface failures.** Not one build error, test failure or runtime fault
> traced to the provider's API surface. Every failure logged was the agent's own TypeScript, its own
> shell quoting, a port collision, or concurrency from a contaminated run.
>
> So the steps were skipped *and the code was still right*. Any fix premised on "skipping causes
> defects" would be fixing something the measurement does not show.

## The measurement — five clean runs

All five: `final_commits=2`, `debrief_touched=0`, every B2 row `[MEASURED]`.

| Step | A1 cs | A1 ts#1 | A1 ts#2 | A3 cs | A3 ts | loaded |
| --- | :--: | :--: | :--: | :--: | :--: | :--: |
| 1 client construction | Y | N | N | Y | N | 2/5 |
| 2 authentication | Y | Y | N | N | N | 2/5 |
| 3 calling endpoints | Y | N | N | N | N | 1/5 |
| 4 models | Y | N | N | N | N | 1/5 |
| **5 error handling** | Y | Y | Y | Y | Y | **5/5** |
| **6 configuration & resilience** | Y | Y | Y | Y | Y | **5/5** |
| 7 testing | N | Y | N | N | N | 1/5 |
| total | 6/7 | 4/7 | 2/7 | 3/7 | 2/7 | |

**Steps 5 and 6 load in every single run, across both languages and both scenarios. Nothing else
exceeds 2 of 5.**

The mechanism is consistent and the agents state it plainly: they skip a step when the source answers
it. A controller signature shows you how to call an operation; an interface shows you a model. Nothing
in the source shows you what throws or what retries — so those two are always loaded, and the rest are
not.

## The finding that makes it actionable

**Across all three runs, not one skip was vindicated.** Every skipped step's parenthetical trap turned
out to be true, or true in the part the build exercised — confirmed afterwards from the source by the
same agent that had judged the skill redundant:

- step 1 — *"`httpClientOptions` does nest `timeout`, and I built one client and reused it"* `[MEASURED]`
- step 3 — *"`new OrdersController(client)`; every operation takes a collapsed options object with
  `requestOptions`"* `[MEASURED]`
- step 4 — enum members must be referenced, not written as bare strings `[MEASURED]`; the union and
  date-codec traps `[UNKNOWN]`, never exercised

So the skills were right, and reading the source is what convinced the reader they were not needed.
The instruction already anticipates exactly this refusal — *"load the named companion skill — **even if
you've already read the relevant source**"* — and an agent quoted that line while declining to follow
it.

## Why the gate is skipped — and why that is probably fine

The parentheticals are **already** framed as the counter-argument: each reads
*"(The signature won't tell you: ...)"*. That is not the problem. The problem, if it is one, is that
each parenthetical then **states the answer**:

> *The signature won't tell you:* the client is configured with a single `Partial<Configuration>`
> options object — there is no separate options/builder; `httpClientOptions` nests
> timeout/retries/proxy/agents; create one client and reuse it.

A reader who reads that has already learned the thing. Loading the companion is then genuinely
redundant *for that fact*, and the reader correctly notices. The entry skill's identity table does the
same job — one ledger row records three companions skipped as *"judged covered: getting-started's
identity table already carried the warning"*.

**So the content is reaching the agent through a cheaper channel than the one the design intends.**
On this evidence that is a feature, not a bug: the facts landed, the code compiled, nothing broke.

### The fix I was about to make, and why I am not making it

The obvious move — make the parenthetical name the trap without resolving it, forcing the load — would
**remove a fact that is currently arriving for free**. An agent that skips the step today still gets
the fact from the parenthetical. Under that change, an agent that skips gets nothing. The measured
outcome is already zero SDK-surface failures, so the change has no upside to buy and a clear downside.

Revisit only if a scenario shows skipping actually costing correctness.

### What would actually test this

A1 is a happy-path task, and the source genuinely does suffice for client construction, calling an
endpoint and building a model. Steps 5 and 6 loaded 3/3 **precisely because** the source cannot answer
what throws and what retries.

**A3 (the resilience scenario) is the real test**, because it demands a bounded timeout, an explicit
retry decision, non-duplicating writes and idempotency — facts that live in the runtime's defaults
rather than in any signature. If the gate holds anywhere it should hold there, and if a skip costs
correctness anywhere it should cost it there. Judge this finding on A3, not on A1.

## The conclusion that actually matters for the hackathon

The two skills that load **5/5** are the two that carried the worst defects found in this entire
effort:

- `csharp-configuration-resilience` — told readers the verb whitelist stopped a `POST` being retried.
  It does not; one create call produced two `POST`s at the provider. **PR #5517.**
- `csharp-error-handling` — omitted `Polly.Timeout.TimeoutRejectedException` from its list of
  non-`ApiException` failures, so a stalled provider surfaced as a bare `500`. **PR #5517.**

So the leverage is not in making the gate stricter. **The skills that get read are already the right
ones; what mattered was whether they were correct.** A defect in `-error-handling` or
`-configuration-resilience` reaches essentially every reader. A defect in `-models` or `-testing`
reaches roughly one reader in five, and the source often covers for it.

**Prioritise correctness in the two always-loaded skills over coverage anywhere else.** That is the
single most useful thing this measurement says, and it inverts the effort's original assumption — which
was that the routing mechanism was the weak point.

## Two measurement caveats that must travel with this finding

1. **The harness masked the missing router.** Both runs report that all eight skills were surfaced with
   descriptions in the system prompt, so getting-started was never needed as an index — its
   *description* did the routing. csharp put it plainly: *"the no-router design cost me nothing in this
   harness — but only because the harness enumerated all eight for me."* Any claim about v3 lacking a
   router must be measured in a client that does **not** pre-list skills.
2. **Two runs is two runs.** csharp at 6/7 and typescript at 4/7 differ enough that the per-language
   number is not yet meaningful. What replicates is the *pattern* — skips land on 1/3/4, and the traps
   were true anyway.

## No rule proposed

An earlier draft proposed a `CLAUDE.md` rule requiring each workflow step to say what the reader would
get wrong. The packs **already do that**, and the measurement shows the current wording produces
correct code. Writing a rule here would codify a change the evidence does not support.

## Corroborated side-finding

Both runs independently reported the `doc/` routing defect (**F004**, fixed in PR #5513). The clean
typescript run also confirms the testing skill earns its place when loaded: step 7 was
*"True and load-bearing — `unstable_httpClientOptions.adapter` is the only seam, and it is what carried
`PAYPAL_BASE_URL` in the shipped service."* In the earlier contaminated run the agent skipped that
skill and reached the same seam unaided, but slower and via a cross-reference it called ambiguous.
