# Results so far — PayPal v3 skills effort

## Shipped: 9 PRs into `apimatic-codegen`, all off `v3-staging`

| PR | Change | Scope |
| --- | --- | --- |
| #5512 | gate the pagination surface on `HAS_PAGINATION` | python |
| #5513 | say where the `doc/` tree is relative to the installed package | 5 packs (php already correct) |
| #5514 | warn a cloned branch can be a different release than the install | `{{SDK_SOURCE}}` renderer, all |
| #5515 | say what an unset environment selects | csharp, python, ruby |
| #5516 | say when a generated file is a thin wrapper over the runtime | csharp, python |
| #5517 | **`RequestMethodsToRetry` does not protect a POST** | csharp |
| #5518 | stop pointing a production base-URL need at the test seam | typescript |
| #5519 | **`httpMethodsToRetry` does not stop a POST being re-sent** | java |
| #5520 | which root exports are values and which are types | typescript |

Four new rules in `Corpus/CLAUDE.md`. Every corpus diff verified by rendering the pack and reading the
output, not by the suite alone.

## The duplicate-write hazard, and how far it reached

`#5517` found that C#'s verb whitelist gates only the response-triggered retry arm; exception-triggered
retries ignore it, and one create call produced two `POST`s at the provider.

**The obvious next move was to sweep that claim into the other five packs. That would have been wrong
in four of them.** Each runtime was read instead:

| Language | Verdict | Mechanism |
| --- | --- | --- |
| TypeScript | refuted | `shouldRetry` gates both branches |
| PHP | refuted | same single-gate shape |
| Ruby | refuted twice over | faraday-retry's one guard, and Net::HTTP's `IDEMPOTENT_METHODS_` |
| Python | refuted for the dangerous case | urllib3 applies `allowed_methods` to the read path; connect-phase retries cannot duplicate a write |
| **Java** | **confirmed** | a *different* mechanism — an unbounded recursive re-send on `SocketException`, plus OkHttp's always-on `retryOnConnectionFailure` |

Java is worse than C#: C# at least bounds re-sends by `NumberOfRetries`; the Java recursion has no
bound at all, and switches on the moment anyone enables retries.

## The scenario design was the thing that mattered

| scenario | runs | SDK-surface failures | plugin-caused defects |
| --- | --- | --- | --- |
| **A1** happy-path checkout | 3 (csharp, ts ×2) | **0** | 1 (`doc/` routing, cosmetic) |
| **A3** resilience | 1 so far (csharp) | **4** | **2, one safety-critical** |

Three happy-path runs found nothing that broke code. The first resilience run found a duplicate-write
hazard the corpus was actively promising did not exist.

A1 lets an agent answer everything from the source: a config type, a method signature, an interface.
A3 demands facts that live in **runtime defaults and runtime dependencies** — what retries, what a
timeout throws — which appear in **no generated file** and can only come from the pack.

**That is the transferable lesson for the hackathon: the skills are load-bearing exactly where the SDK
source is silent.** Scenarios that never leave the happy path cannot measure them.

## The single most important defect

`csharp-configuration-resilience` told readers the verb whitelist kept a `POST` from being retried. The
runtime retries on `.Or<TaskCanceledException>()` / `.Or<HttpRequestException>()` **without** consulting
that whitelist. One create call produced two `POST`s at the provider.

The benchmark's own archived corpus records every prior SDK arm re-sending a `POST` two-to-four times
on a dropped connection, and read that as builders failing to write resilience. **At least in C#, the
skill was telling them they did not need to.**

## Rejected findings — the discipline that earned its keep

Three confidently-reported `[MEASURED]` defects were rejected on verification. All three would have
degraded correct prose:

| Reported | Reality |
| --- | --- |
| `ApiHelper.JsonSerialize` missing → compile failure on every route | `ApiHelper : CoreHelper {}` — inherited; the call compiles |
| python's first enum member is falsy | true of `IntEnum`; the generator emits `Enum` |
| `tslib` filed as a devDependency | the generator puts it in `dependencies` deliberately; the **published SDK is older than the generator** |

That last one exposed a triage category the rubric lacked: **the artifact can pre-date the generator**,
in which case the pack is right and nothing should change. Two of the three were that.

## Corrections I made to my own conclusions

- **F005** (workflow gate) — first reported HIGH with a proposed rewrite of every step's parenthetical.
  Section H showed **zero** SDK-surface failures across three runs; the steps were skipped and the code
  was still right. Downgraded to `defer`, and the proposed fix would have *removed* a fact that
  currently arrives for free.
- **F001** — first claimed the SDK defaults to Production ("live money"). The `Builder` initialises to
  `Sandbox`; enum member order does not set the default. Later **raised** to HIGH for a different
  reason: the environments table omits Production entirely, so a builder said *"had I trusted the table
  I would have shipped a service with no route to production."*

## Open

- A3 running for typescript and python.
- Whether the C# retry/exception split exists in the other five runtimes — under investigation. If it
  does, five more packs may carry the same false promise. **Not assumed from C#.**
- F010 (TS value/type export split) — single-session, held for corroboration.
- F006 (OAuth provider rejection poisoning) — single-session, and must be checked against the two
  known deliberate divergences (#5445, #5498) before anything is written.
