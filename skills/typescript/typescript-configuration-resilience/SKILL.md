---
name: 'typescript-configuration-resilience'
description: 'Tune an APIMatic-generated TypeScript/Node.js PayPal Server SDK SDK client — retries and timeouts live under a nested `httpClientOptions`, not at the top level; retry defaults are baked in at generation time so you must read `DEFAULT_RETRY_CONFIG` rather than assume them; timeout is per-attempt, not total; plus AbortSignal cancellation, environment selection, and the built-in `logging` config this build emits. Use whenever adjusting retry policy, timeouts, the environment, or logging on the PayPal Server SDK TypeScript SDK — load it even after reading the options in the source, since the field list does not reveal that the retry defaults vary per SDK, that timeout is per-attempt, or that there is no free-form base URL.'
---

# Configuration & resilience for an APIMatic TypeScript SDK

All configuration is passed at construction time in the single `Configuration` object (see
**typescript-client-initialization**). Transport tuning is **nested under `httpClientOptions`** — it is
not top level.

```typescript
import { Client, Environment } from '@paypal/paypal-server-sdk';

const client = new Client({
  environment: Environment.{Name},
  timeout: 30_000,                 // ms, per attempt
  httpClientOptions: {
    timeout: 30_000,               // overrides the top-level timeout when present
    retryConfig: { /* see below */ },
    proxySettings: { /* ... */ },
    httpAgent: undefined,          // custom Node http agent (keep-alive, TLS)
    httpsAgent: undefined,
  },
});
```

## Base URL / environment

There is **no free-form `baseUrl` option**. The base URL is derived from the selected `Environment`
member (plus any server parameters such as `port`) by a private resolver in `src/client.ts`.

Read the `Environment` enum in **`src/configuration.ts`** for the real member names before naming one —
they vary per API, and a name like `Production` may not exist at all.

> **There is no supported way to send this client to a host no `Environment` member covers.** That is
> the honest answer, and it is worth stating plainly because the need is common — a gateway, a sandbox
> proxy, a recorded mock, a base URL supplied by an environment variable in production.
>
> **typescript-testing** describes a seam that *can* redirect traffic, but reaching it means supplying
> your own transport adapter, i.e. owning the HTTP call the SDK was going to make. That is a reasonable
> trade **in a test**, where you were replacing the transport anyway. In production code it means the
> SDK is no longer making the request, and the retry, timeout and auth behaviour documented here stop
> applying to it. Do not read the cross-reference as a supported production knob.
>
> If a deployment genuinely needs an arbitrary base URL, the options are: regenerate the SDK from a
> spec whose `ServerConfiguration` declares that environment, or route to it below the SDK — a proxy,
> or DNS — where the client's own behaviour is unaffected.

## Retries

Retry policy lives at `httpClientOptions.retryConfig`:

```typescript
const client = new Client({
  httpClientOptions: {
    retryConfig: {
      maxNumberOfRetries: 5,
      retryInterval: 1,            // seconds
      backoffFactor: 2,
      maximumRetryWaitTime: 60,    // seconds; total retry-wait budget — 0 disables retrying entirely
      retryOnTimeout: true,
      httpStatusCodesToRetry: [408, 429, 500, 502, 503, 504],
      httpMethodsToRetry: ['GET', 'PUT'],
    },
  },
});
```

The seven fields above are the complete set. **Their defaults are fixed when the SDK is generated, not
by the runtime** — `maxNumberOfRetries`, `httpStatusCodesToRetry` and `httpMethodsToRetry` in particular
differ from SDK to SDK because they come from the code-generation settings for that build.

> Do not assume a default. Read `DEFAULT_RETRY_CONFIG` in **`src/defaultConfiguration.ts`** — that is
> the generated, authoritative value for this SDK. Retries may well be off (`maxNumberOfRetries: 0`).

Anything you pass is merged over that default, so you can override one field and leave the rest.

> **Raising `maxNumberOfRetries` alone is not enough.** `maximumRetryWaitTime` is the total retry-wait
> budget, and a retry only happens when the computed backoff fits inside it — so when the generated
> `DEFAULT_RETRY_CONFIG.maximumRetryWaitTime` is `0` (a common generated default) **nothing is ever
> retried**, whatever `maxNumberOfRetries` says. Set both.

Notes:
- Only the methods listed in `httpMethodsToRetry` are retried. If `POST`/`PATCH`/`DELETE` are absent —
  the common case — those errors surface without any retry. Add one only if the operation is idempotent.
- `timeout` is **per attempt**, not total. To bound a whole call including retries, use an `AbortSignal`.
- **Two generated fields gate retrying, and either one at `0` suppresses every retry** — the first
  backoff wait is at least `retryInterval` and never fits a zero budget. `maxNumberOfRetries` is the
  `Retries` code-generation setting and `maximumRetryWaitTime` is the `BackoffMax` setting; both default
  to `0` and a spec can raise either independently, so a build with `Retries: 3` but `BackoffMax` unset
  still retries nothing. **Read both values in `DEFAULT_RETRY_CONFIG` in `src/defaultConfiguration.ts`
  before you reason about resilience** — they are what decides whether this SDK retries at all.
  Raise both only where nothing above the SDK already retries — a queue consumer, a job runner, a
  failover wrapper, or your own orchestration loop. Retry layers multiply rather than add:
  `maxNumberOfRetries: 3` is **four** requests per attempt, so inside a 3-attempt job it is twelve
  requests against an API whose rate limit counts every one.

## Per-request timeout / cancellation

Pass an `AbortSignal` as the `abortSignal` member of `requestOptions` to bound an individual call.
`requestOptions` is the operation's **last** parameter, but how you reach it depends on the parameter form
the operation was generated in — positional, or one collapsed options object when the operation has more
than one parameter and the build sets `CollapseParamsToArray` (see **typescript-calling-endpoints**). Read
the signature in `src/controllers/` first — that folder is named by the `ControllerNamespace` generator
setting, so confirm the name in your own `src/`:

```typescript
const controller = new AbortController();
setTimeout(() => controller.abort(), 10_000);

// Positional: {operation}({id}: string, {optionalParam}?: string, requestOptions?: RequestOptions)
// pad the optionals you skip:
const a = await api.{operation}({id}, undefined, { abortSignal: controller.signal });

// Collapsed: {operation}({ {id}, {optionalParam} }: {...}, requestOptions?: RequestOptions)
// nothing to pad — requestOptions follows the options object:
const b = await api.{operation}({ {id} }, { abortSignal: controller.signal });
```

The key is `abortSignal`, not `signal`, and it is the only member `RequestOptions` has.

## Pagination

**No operation in this API is paginated.** `package.json` does not depend on `@apimatic/pagination`, no
method returns a `PagedAsyncIterable`, and there is no `doc/paged-async-iterable.md` — every operation is
`async`, is awaited, and hands back `ApiResponse<T>`. A list endpoint here is a plain list call: drive its
own `page`/`perPage` (or cursor) parameters yourself and stop when a page returns fewer items than you
asked for.

## Logging

This SDK **was generated with logging enabled**, so logging is built in: the `Configuration` interface in
`src/configuration.ts` carries a `logging` property, and `LogLevel`, `LoggerInterface` and `ConsoleLogger`
are exported from the package root. Configure it directly — there is no need to wrap the transport:

```typescript
import { Client, LogLevel } from '@paypal/paypal-server-sdk';

const client = new Client({
  logging: {
    logLevel: LogLevel.Info,
    logRequest:  { logBody: true, logHeaders: true, headersToExclude: ['authorization'] },
    logResponse: { logBody: true, logHeaders: false },
  },
});
```

With **no** `logging` config the client uses `DEFAULT_LOGGING_OPTIONS` from
`src/defaultConfiguration.ts`, which ships a `NullLogger` — logging is silent. Supplying **any**
`logging` object switches the fallback to the runtime defaults (`ConsoleLogger` at `LogLevel.Info`), so
logging starts immediately even if you set neither `logger` nor `logLevel`; pass your own `logger` to
redirect it.

`logRequest` and `logResponse` each also accept `headersToInclude` and `headersToWhitelist`;
`includeQueryInPath` is request-only. Header masking is `maskSensitiveHeaders`, set on `logging` itself
rather than on `logResponse`.

## Verify on the wire (first run of any new integration)

Log the first execution of any new call — by whichever mechanism the section above leaves you — and
inspect the output. A wrong environment, a leftover `{placeholder}`, or a mis-serialized path segment
**compiles cleanly** and produces no in-band signal; the only symptom is a runtime `404`/`422`.

Checklist for the first logged request:
1. the **verb** matches the operation;
2. the **path** has no literal `{placeholder}` left unsubstituted;
3. each **path-param segment** is the value the API expects;
4. the query params you set actually appear in the query string.

Turn it back down once verified.
