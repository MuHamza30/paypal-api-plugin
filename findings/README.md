# Findings — testing the v3 context-plugin skills with PayPal

What happened when coding agents built real PayPal integrations against the skills in this plugin, in
isolated sessions, and then answered a structured debrief about how they had used them.

**PayPal was the probe, not the subject.** Every accepted fix is API-agnostic and landed in the
generator's shared corpus, so it reaches every SDK that generator builds — not just this one.

## Accepted, with a PR against `apimatic-codegen`

| ID | Finding | Severity | PR |
| --- | --- | --- | --- |
| [F011](F011-retry-whitelist-does-not-gate-exceptions.md) | **`RequestMethodsToRetry` does not protect a `POST` (C#).** The verb whitelist gates only the response-triggered retry arm; exception-triggered retries ignore it. One create call produced **two `POST`s at the provider**. | critical | #5517 |
| [F012](F012-maxretrywaittime-is-a-call-timeout.md) | `MaximumRetryWaitTime` also sizes a whole-call timeout, and its exception is not an `OperationCanceledException` — a stalled provider surfaced as a bare `500`. | high | #5517 |
| — | **`httpMethodsToRetry` does not stop a `POST` (Java)** — a *different* mechanism: an unbounded recursive re-send on `SocketException`, plus OkHttp's always-on `retryOnConnectionFailure`. | critical | #5519 |
| [F004](F004-doc-tree-not-in-published-package.md) | Five of six packs route the reader to a `doc/` tree their published package does not ship. Composer's does — measured per registry. | high | #5513 |
| [F002](F002-default-environment-may-be-live.md) | No pack said what an unset environment selects, and five languages trap differently. | med-high | #5515 |
| [F003](F003-python-pagination-ungated.md) | Python documented `PagedIterable` for an SDK with no pagination package. | high | #5512 |
| [F006](F006-oauth-provider-rejection.md) | A rejecting `oAuthTokenProvider` poisons the cached promise — one transient failure kills the client for the process. | high | #5521 |
| [F008](F008-clone-branch-vs-pinned-version.md) | The clone lands on a branch while the install pins a version; the caveat was on the wrong arm. | med-high | #5514 |
| [F010](F010-ts-value-vs-type-exports.md) | The TS barrel exports enums and union bases as values, plain models as types — unstated, and a hard error under `isolatedModules`. | medium | #5520 |
| — | A generated file that is a thin wrapper over the runtime reads as empty; a reader concluded a member was missing and hand-rolled a replacement. | medium | #5516 |
| — | A read timeout and a refused connection are the **same class** in Python — the distinction that decides whether a write is safe to re-send. | high | #5522 |
| — | A TS production base-URL need was pointed at a test seam. | medium | #5518 |

## Rejected — and this is the more useful half

Five reported defects, several tagged `[MEASURED]` by the reporting agent, did **not** survive
verification. Each would have degraded correct prose into a workaround.

| ID | Reported | What was actually true |
| --- | --- | --- |
| [F007](F007-apihelper-false-defect.md) | `ApiHelper.JsonSerialize` missing — "a compile failure on every route" | The file is `public class ApiHelper : CoreHelper { }`. The member is **inherited**; the call compiles. The pack was right in all six places it says so. |
| [F009](F009-tslib-rejected-version-skew.md) | `tslib` mis-filed; no `exports` map; the `/errors` subpath reason is wrong | The generator emits all three as the corpus describes. **The published SDK simply pre-dates the generator.** |
| [F013](F013-base-controller-example.md) | The `BaseApi` example is wrong for this SDK | It is a conditioned illustration of a correct rule, and the next clause tells the reader to check the source. |
| — | Python's first enum member is falsy | True of `IntEnum`; the generator emits `Enum`. |

**Four of those five were not "the corpus is wrong" or "the generator is wrong" but a third thing: the
installed artifact was older than the generator, or the reader had read the wrong file.** Correcting the
corpus toward a stale package would make it wrong for every SDK built from then on. Any defect count
taken from agent reports without this filter overstates the fault rate badly.

## Not a corpus problem

| ID | Finding | Owner |
| --- | --- | --- |
| [F001](F001-environment-spec-defect.md) | The spec bundle declares one environment; the published SDK has two. A builder said *"had I trusted the table I would have shipped a service with no route to production."* | spec |
| — | `core-lib-python`'s `force_retries` mutates a shared `Retry` object and never restores it. | runtime |
| — | `apimatic-js-runtime` caches a rejected token promise with no reset path. | runtime |

## Method, and what it cost to get right

| ID | What went wrong in the measurement | Fix |
| --- | --- | --- |
| [R001](R001-round-01-invalidated.md) | Two runners shared a workspace — a "killed" background job was still alive, and `git init` over a surviving `.git` preserves history, so a workspace looked fresh and was not. | atomic lock; assert the baseline is the only commit |
| [R002](R002-round-02-first-attempt-confounded.md) | The improved pack was rendered with the wrong package identity, so every import line was wrong. Two rounds differed in two ways instead of one. | supply the real SDK coordinates before rendering |

Three further harness bugs were fixed along the way: the debrief ran in a *fresh* session (so every
recall-dependent answer came back `[UNKNOWN]` until it used `--continue`), contamination was detected by
`git status` rather than content (reading a file moves its mtime on Windows), and a completion detector
polled a file that a case-insensitive filesystem had merged with its own prompt.

## The result that generalises

| scenario | runs | SDK-surface failures | plugin-caused defects |
| --- | --- | --- | --- |
| happy-path checkout | 3 | **0** | 1 (cosmetic) |
| resilience | 4 | several | **4, two safety-critical** |

Three happy-path runs found nothing that broke code. The resilience scenario found a duplicate-write
hazard the corpus was actively promising did not exist.

The reason is structural: a happy path can be answered from the SDK source — a config type, a method
signature, an interface. Resilience cannot. What retries, what a timeout throws, and what a whitelist
actually gates live in **runtime defaults and runtime dependencies**, which appear in no generated file.

Measured across five clean runs, the two skills agents load **every single time** are
`-error-handling` and `-configuration-resilience`; nothing else exceeded 2 of 5. **Those two are also
where the worst defects were.** So the leverage is not in routing agents to more skills — it is in the
accuracy of the two they always read.

See [RESULTS.md](RESULTS.md) for the full write-up.

---

*Findings are dated to the runs that produced them. Each states how its claim was established, and
`[MEASURED]` / `[ASSERTED]` / `[UNKNOWN]` tags are the reporting agent's own grading, preserved
verbatim where quoted.*
