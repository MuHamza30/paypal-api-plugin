---
name: 'typescript-testing'
description: 'Unit-test code that calls the APIMatic-generated TypeScript/Node.js PayPal Server SDK SDK — the transport is an axios adapter, so the in-process seam is a stub adapter passed via `unstable_httpClientOptions`, with `nock`/`msw` as the HTTP-level alternative; covers stubbing success and error responses, asserting the outgoing request, asserting the right error class per operation, and registering a stub client in a DI container. Use when writing, mocking, or stubbing tests for calls made through the PayPal Server SDK TypeScript SDK — load it even after reading the constructor in the source, since the config alone won''t tell you which property is the seam, or that retries must be disabled so a stubbed 5xx fails fast.'
---

# Testing code that uses an APIMatic TypeScript SDK

The SDK has **no mocking helpers**. Its transport is an axios client
(`@apimatic/axios-client-adapter`), and the only injection point exposed on `Configuration` is
**`unstable_httpClientOptions`** — passed straight through to that adapter as its axios config. Supplying
a stub **axios adapter** there is the in-process seam: no network, no global patching.

> `unstable_` means exactly that: it is an escape hatch, not a stable API, and may change between SDK
> versions. If you would rather not depend on it, stub at the HTTP layer with `nock` (Node) or `msw`
> instead — the assertions below apply either way.

**Match the project's existing test stack — don't impose one.** Check the test project's `package.json`
and existing tests, then mirror both its **test framework** (Jest / Vitest / Mocha) and its **assertion
style**. The samples below use Jest `test` + `expect` **purely for reference** — they show the seam and
*what* to assert, not a mandated framework.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{apiGroup}`,
> `{operation}`) — replace it with the concrete identifier from the source.

## A reusable stub helper

```typescript
import { Client } from '@paypal/paypal-server-sdk';

function clientReturning(status: number, body: unknown): {
  client: Client;
  lastRequest: () => any | undefined;
} {
  let captured: any;

  const client = new Client({
    // Dummy credentials for whatever scheme the operation requires. Auth is applied BEFORE the
    // transport, so an unauthenticated stub client throws instead of reaching your adapter.
    // Copy the property name and its inner field names from the `Configuration` interface in
    // src/configuration.ts; prefer an API-key/basic scheme over an OAuth one.
    {apiKeyProperty}: { /* dummy values */ },
    unstable_httpClientOptions: {
      // An axios adapter: receives the request config, returns an axios response.
      adapter: async (config: any) => {
        captured = config;
        return {
          // The SDK disables axios response transformation (`transformResponse: []`), so the
          // adapter must hand back the raw serialized body exactly as the wire would — a string.
          data: typeof body === 'string' ? body : JSON.stringify(body),
          status,
          statusText: '',
          headers: { 'content-type': 'application/json' },
          config,
        };
      },
    },
    httpClientOptions: {
      retryConfig: { maxNumberOfRetries: 0 },  // stubbed 5xx fails fast, no backoff wait
    },
  });

  return { client, lastRequest: () => captured };
}
```

If the operation needs **no** auth, drop the credentials line. If it needs OAuth, prefer stubbing an
API-key or basic scheme it also accepts — an OAuth stub sends a token-fetch call through the same
adapter and skews invocation counts.

## Test a success path

```typescript
test('returns deserialized body', async () => {
  const { client } = clientReturning(200, { id: 123 });
  const api = new {Controller}(client);

  const response = await api.{operation}(/* args — see the signature */);

  expect(response.statusCode).toBe(200);
  expect(response.result.id).toBe(123);
});
```

Operations return `ApiResponse<T>`, so the deserialized payload is at `.result` — never a bare property
on `response`.

`/* args — see the signature */` stands for the operation's arguments in the form it was generated in:
positional parameters, or a **single options object** bundling them by name when the operation has more
than one parameter and the build sets `CollapseParamsToArray`. Copy the shape from the method in
`src/controllers/` (that folder is named by the `ControllerNamespace` generator setting, so confirm the
name in your own `src/`) — a call written in the wrong form does not compile, so the test never runs.

## Test an error path

Endpoint methods throw `ApiError` on non-2xx (see **typescript-error-handling**).

The thrown value is either a typed `{ErrorResponse}Error` subclass (**Case A**) for statuses the operation
maps to an error model, or base `ApiError` (**Case B**) otherwise.

Error classes are named after the error **response model**, not the operation, and one operation can
throw several. Read the `req.throwOn(<status>, <ErrorClass>, ...)` and `req.defaultToError(...)` lines in
the operation's method in the controllers folder — and, where the method has none, the client-wide
`withErrorHandlers` block in `src/client.ts`, which registers the spec's global errors for every
operation — to get the class for the status code you are stubbing.

**Assert the concrete class, not `ApiError`** — every typed class derives from it, so a test expecting
`ApiError` passes for all of them and proves nothing about which one was raised.

**Case A — typed `{ErrorResponse}Error`:**

```typescript
// Typed error classes are re-exported from the package root; `.` and `./metadata`
// are the only exported subpaths, so there is no `/errors` import path.
import { {ErrorResponse}Error } from '@paypal/paypal-server-sdk';

test('throws typed error on API error', async () => {
  const { client } = clientReturning(422, { errors: ['bad input'] });
  const api = new {Controller}(client);

  await expect(
    api.{operation}(/* args — see the signature */)
  ).rejects.toThrow({ErrorResponse}Error);
});
```

**Case B — base `ApiError`:**

```typescript
import { ApiError } from '@paypal/paypal-server-sdk';

test('throws ApiError on non-2xx', async () => {
  const { client } = clientReturning(422, { errors: ['bad input'] });
  const api = new {Controller}(client);

  await expect(
    api.{operation}(/* args — see the signature */)
  ).rejects.toBeInstanceOf(ApiError);

  try {
    await api.{operation}(/* args — see the signature */);
  } catch (err) {
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).statusCode).toBe(422);
  }
});
```

## Assert the outgoing request

The stub captures the **axios request config**, so you can assert method, URL, headers and body off it:

```typescript
test('sends correct request', async () => {
  const { client, lastRequest } = clientReturning(200, {});
  const api = new {Controller}(client);

  await api.{operation}(/* args — see the signature */);

  const req = lastRequest()!;
  expect(req.method.toUpperCase()).toBe('POST');
  expect(req.url).toContain('/expected/path');

  // Query parameters are already encoded into req.url — `req.params` is never populated,
  // so an assertion against it passes vacuously.
  expect(new URL(req.url).searchParams.get('limit')).toBe('25');

  // req.data is the serialized request body (a string for JSON payloads; undefined for GETs):
  expect(JSON.parse(req.data)).toMatchObject({ expectedField: 'value' });
});
```

Field names here are axios's (`method`, `url`, `data`, `headers`), not the WHATWG `Request` shape, and
`req.method` arrives lower-case — log the captured config once if you are unsure what a given operation
produces.

## Notes

- **Disable retries in tests** (`httpClientOptions.retryConfig.maxNumberOfRetries: 0`) so a stubbed
  `5xx` fails on the first attempt instead of waiting out the backoff.
- To test that retries *do* fire, have the stub return `503` then `200` and count adapter invocations —
  but you must set **both** `maxNumberOfRetries` **and** a non-zero `maximumRetryWaitTime`, plus a small
  `retryInterval` to keep the test fast:
  `retryConfig: { maxNumberOfRetries: 2, retryInterval: 0.01, maximumRetryWaitTime: 60 }`.
  `maximumRetryWaitTime` is the total retry-wait budget, and when the generated default is `0` **no
  retry ever fires**, whatever `maxNumberOfRetries` says — you would wrongly conclude the method is
  outside the retry set. Which methods retry is baked in at generation time; check
  `DEFAULT_RETRY_CONFIG.httpMethodsToRetry` in `src/defaultConfiguration.ts` for the verb under test.
- Controllers are constructed from the client (`new {Controller}(client)`), so the stubbed client is the
  only transport you fake — but it must still carry credentials for the scheme the operation requires
  (see the helper above), since auth is applied before the request reaches your adapter.
- For DI-based code (e.g. NestJS), override the provider in your test module:
  ```typescript
  const { client } = clientReturning(200, {});
  const moduleRef = await Test.createTestingModule({ imports: [ApiModule] })
    .overrideProvider(Client)
    .useValue(client)
    .compile();
  ```
- To look up an operation's signature, its request type, or a `{ErrorResponse}Error`'s payload, read the
  SDK source `.ts` files — don't rely solely on the compiled `.d.ts` declarations, which may drop JSDoc
  comments and internal builder details.
