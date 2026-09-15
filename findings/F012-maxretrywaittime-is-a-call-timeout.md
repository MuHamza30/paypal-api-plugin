# F012 — `MaximumRetryWaitTime` is also a whole-call timeout, and its exception is not a cancellation

**Severity: HIGH (wrong status to the caller).** **Disposition: `accept-corpus`, csharp.**
**LANDED — PR #5517.** Source: round-01 A3 csharp §D2, `[MEASURED]`; verified by me in the runtime.

## What the corpus said

The `HttpClientConfiguration.Builder` surface table described it in four words:

> `MaximumRetryWaitTime(TimeSpan)` | `TimeSpan` | cap on retry waiting

And `csharp-error-handling`'s "Failures that are not `ApiException`" list named transport,
auth-validation, JSON and union-validation failures — **not this one.**

## What is true `[MEASURED]`

`HttpClientWrapper.cs:293-301`:

```csharp
private AsyncTimeoutPolicy GetTimeoutPolicy() =>
    ... Policy.TimeoutAsync(_maximumRetryWaitTime);

private AsyncPolicyWrap<HttpResponseMessage> GetCombinedPolicy(RetryOption retryOption) =>
    GetTimeoutPolicy().WrapAsync(GetRetryPolicy(retryOption));
```

The setting sizes a Polly timeout policy **wrapping the whole call**, retries included. Breaching it
throws `Polly.Timeout.TimeoutRejectedException`, which **does not derive from
`OperationCanceledException`**.

Verbatim from the build's service log:

> `Polly.Timeout.TimeoutRejectedException: The delegate executed asynchronously through TimeoutPolicy
> did not complete within the timeout.`

## The consequence

The builder's catch ladder handled cancellation, as the skill's list implied was sufficient. The Polly
exception escaped it, so **a stalled provider surfaced to the caller as a bare `500` instead of the
`504` the service intended**. It cost more debugging time than anything else in that build — first
misdiagnosed as test-ordering interference, then as a bug in its own fake.

## The detail that makes it a corpus problem rather than a reader problem

The type comes from the runtime's own dependency (`Polly`), so **it appears in no generated file**. A
reader grepping the SDK for what a timeout throws finds nothing and concludes cancellation is the whole
story. There is no route to this fact except the skill stating it.

That is the general shape worth remembering: *when a behaviour is produced by the runtime's
dependencies rather than by generated code, the pack is the only place it can come from.*

## Not swept — deliberately

csharp only. Polly is a .NET library; the other packs' timeouts throw whatever their own runtime uses.
