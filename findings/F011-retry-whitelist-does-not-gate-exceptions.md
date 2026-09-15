# F011 — `RequestMethodsToRetry` does not protect a POST (duplicate-write hazard)

**Severity: CRITICAL (safety, duplicate writes).** **Disposition: `accept-corpus`, csharp.**
**LANDED — PR #5517.** Source: round-01 A3 csharp §D1, `[MEASURED]`; verified by me in the runtime.

## What the corpus said

`csharp-configuration-resilience`, in the section whose stated purpose is preventing duplicate writes:

> "only the verbs in `RequestMethodsToRetry` are eligible, and that list defaults to **`GET, PUT`** on
> every build checked, so `POST`/`PATCH`/`DELETE` still surface immediately until you add them — and add
> one only when the operation is genuinely idempotent."

Read literally: a `POST` is never repeated while `POST` is absent from the list.

## What is true `[MEASURED]`

`apimatic/core-lib-csharp`, `APIMatic.Core/Http/HttpClientWrapper.cs:279-282`:

```csharp
Policy.HandleResult<HttpResponseMessage>(response => ShouldRetry(response, retryOption))
    .Or<TaskCanceledException>()
    .Or<HttpRequestException>()
```

`ShouldRetry` (`:260`) is the only place `_requestMethodsToRetry` is consulted. **The two `.Or<...>`
arms do not consult it.** A call that fails by *throwing* — a timeout (`TaskCanceledException`) or a
transport fault (`HttpRequestException`) — is retried regardless of verb.

The builder observed it directly: `NumberOfRetries(2)` with a `GET`-only whitelist produced **two
create `POST`s at the provider** from one caller request. It instrumented its fake provider to log each
attempt and saw `attempt #1` and `attempt #2`. Only the provider's own idempotency key stopped that
becoming two orders.

## Why this one matters more than the others

The false claim sits in the section a careful reader consults *specifically* to decide whether a write
is safe to retry, and it tells them the thing that makes it unsafe is already handled. A reader who
trusts it raises `NumberOfRetries` believing writes are excluded, and ships a double-charge path.

This is also the failure mode the benchmark's own corpus records for every prior SDK arm — all four
re-sent a `POST` two-to-four times on a dropped connection. That was previously read as builders not
writing resilience. **At least in C#, the skill was actively telling them they did not need to.**

## The fix as landed

States that the whitelist gates only the response-triggered arm; that a raised `NumberOfRetries` is a
decision about every verb; and that the whitelist cannot express "never re-send this one" — separate
the clients, or keep `NumberOfRetries(0)` on the write path.

## Not swept — deliberately

csharp only. The other five packs sit on different runtime libraries with their own retry wrappers.
**Read that language's wrapper before writing a line of this claim into its pack.** Whether the same
split exists in `core-lib-php`, `io.apimatic:core`, `apimatic_core` (python/ruby) and the TS adapters
is **unknown and must be checked individually**, not assumed from C#.
