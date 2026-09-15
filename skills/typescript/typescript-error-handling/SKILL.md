---
name: 'typescript-error-handling'
description: 'Handle errors from an APIMatic-generated TypeScript/Node.js SDK — calls always throw on non-2xx, either base `ApiError` or a typed subclass named after the **error response model** (not the operation), carrying `statusCode`, `headers` and the raw `body` — plus a parsed `result` that the runtime fills on the typed subclasses only. Use the moment you write a try/catch around a call, handle a non-2xx/error response, or read a status code or rate-limit headers on the PayPal Server SDK TypeScript SDK — load it even after reading the thrown type in the source, since the type alone won''t warn you that one operation maps several status codes to several different error classes, that the payload lives on `err.result` rather than on accessors of the subclass, or that a bare `ApiError` leaves `result` undefined so you must read `body`.'
---

# Error handling for an APIMatic TypeScript SDK

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{operation}`, `{apiGroup}`) — replace it with the concrete identifier from the source.

Endpoint methods **throw on non-success responses** — there is no non-throwing variant, and this is not a
generator option: the shared `@apimatic/core` request builder installs a response validator that throws for
any status outside `200`–`299`, on every operation of every build.

The thrown type is always `ApiError`, but it comes in **two shapes**, depending on the status code the
operation — or the client-wide `withErrorHandlers` block in `src/client.ts`, which registers the spec's
global errors for every operation at once — maps:

- **Typed subclass (Case A)** — a `{ErrorModel}Error` subclass of `ApiError` generated under
  `src/errors/`. It is named after the **error response model**, never after the operation, and a single
  operation commonly maps several status codes to several *different* classes.
- **Base `ApiError` (Case B)** — for a status the operation maps to no typed model, `err` is `ApiError`
  itself.

## Which error class does an endpoint throw?

Do **not** guess from the operation name. For the operation you are calling, either:

- open its `## Errors` table in `doc/controllers/{group}.md`, which lists the exception class per status
  code; or
- read the `req.throwOn(<status>, <ErrorClass>, ...)` and `req.defaultToError(<ErrorClass>, ...)` lines
  in the operation's own method body in `src/controllers/` — that folder is named by the
  `ControllerNamespace` generator setting, so confirm the name in your own `src/`.

Because one operation can raise several classes, a catch ladder needs **one `instanceof` branch per
class** you want to treat specially, with a final `instanceof ApiError` branch as the catch-all — or just
the single `ApiError` branch if you handle them uniformly.

## Catch the exception

`ApiError<T>` exposes `statusCode: number`, `headers: Record<string, string>`, `request: HttpRequest`,
`result: T | undefined` (the **parsed** JSON payload) and `body: string | Blob | NodeJS.ReadableStream`
(the **raw, unparsed** payload).

`result` is populated only for the typed `ApiError` *subclasses*: the runtime JSON-parses the body into
`result` only when the class registered by `req.throwOn(...)` is a subclass of `ApiError` —
`@apimatic/core`'s request builder guards the parse with
`if (errorConstructor.prototype instanceof ApiError)` under the comment
`// Load result only for the sub classes of ApiError`. A bare `ApiError` — the `rb.defaultToError(ApiError)`
path taken for any status the operation maps to no typed model — is thrown without that step, so its
`result` is always `undefined` and only `body` is filled in. Read `body` on Case B.

Typed subclasses add **no members of their own** — they are one-liners
(`export class {ErrorModel}Error extends ApiError<{Payload}> {}`), so the payload fields are on the
inherited `err.result`, not on `err` directly.

### Case A — the status maps to a typed `{ErrorModel}Error`

```typescript
import { Client, ApiError, {ErrorModel}Error } from '@paypal/paypal-server-sdk';

try {
  const response = await api.{operation}(/* ... */);
  // use response
} catch (err) {
  if (err instanceof {ErrorModel}Error) {
    // the typed payload is on err.result (possibly undefined) — open the error class
    // under src/errors/ for its payload interface
    console.error('Typed error:', err.statusCode, err.result?.code, err.result?.message);
  } else if (err instanceof ApiError) {
    // Fallback for any other API error
    console.error(`HTTP ${err.statusCode}:`, err.result ?? err.body);
  } else {
    throw err;  // re-throw non-API errors (network, timeout, etc.)
  }
}
```

`ApiError` and every generated error class are exported from the **package root** — there is no
`@paypal/paypal-server-sdk/errors` subpath (`.` and `./metadata` are the only exported subpaths).

### Case B — the status maps to no typed model

`err` is `ApiError` — read the status off it, and the payload off **`body`**: nothing parsed the body
into `result` on this path, so `err.result` is `undefined` here.

```typescript
import { ApiError } from '@paypal/paypal-server-sdk';

try {
  const response = await api.{operation}(/* ... */);
  // use response
} catch (err) {
  if (err instanceof ApiError) {
    console.error(`HTTP ${err.statusCode}`);
    console.error(err.body);        // err.result is undefined on a bare ApiError
  } else {
    throw err;
  }
}
```

`body` is `string | Blob | NodeJS.ReadableStream`, so parse it yourself (`JSON.parse(err.body as string)`
in Node) if you need the fields.

Response headers — rate-limit headers included — are on `err.headers` on both shapes; no special call is
needed to reach them.

## Notes

- Network/transport failures (connection refused, DNS failure, TLS error) surface as errors from the underlying axios transport, **not** as `ApiError`; cancellation via an `AbortSignal` surfaces as `AbortError`. Handle both separately from `ApiError` — the `else { throw err; }` branches above exist for exactly this.
- When none of an operation's accepted auth combinations has its credentials configured, the composite
  auth provider throws a **plain `Error`** (message: `Required authentication credentials for this API
  call are not provided or all provided auth combinations are disabled`) before the request is sent. It
  is not an `ApiError`, so it falls through to the `else { throw err; }` branch. There is no `AuthError`
  type. Where the SDK generates a fallback credentials object (a scheme with a deprecated bare-string
  field — check whether `src/client.ts` hands `createAuthProviderFromConfig` a cloned config), that
  pre-flight throw never fires: the call goes out with an empty credential and returns a `401` as an
  `ApiError` instead. See **typescript-authentication**.
- Automatic retries are gated by **several** generated defaults in `src/defaultConfiguration.ts` — read
  the real values in your own SDK rather than assuming. `DEFAULT_RETRY_CONFIG.maxNumberOfRetries` (the
  `Retries` setting) and `maximumRetryWaitTime` (the `BackoffMax` setting) both default to `0`, and
  **either one at `0` means no retry happens at all**: every transient status surfaces on the first
  attempt, `GET` and `PUT` included, until you raise both via `httpClientOptions.retryConfig`. Past
  those, only the methods in `httpMethodsToRetry` and the statuses in `httpStatusCodesToRetry` are
  retried, so methods outside that set surface without retry. See **typescript-configuration-resilience**.
