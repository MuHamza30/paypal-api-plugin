---
name: 'typescript-calling-endpoints'
description: 'Call API operations on an APIMatic-generated TypeScript/Node.js SDK — operations live on a controller class you instantiate yourself (not on the client), and an operation takes either positional parameters or one destructured options object depending on whether the build collapses parameters, always ending with `requestOptions`. Also covers building request models, TypeScript `enum`s, cancelling a call with `abortSignal`, and reading the response shape — always `ApiResponse<T>`, whose wrapped `T` varies per operation. Use whenever invoking an endpoint, building a request body, or consuming a response from the PayPal Server SDK TypeScript SDK — load it even after reading the method signature in the source, since the signature won''t warn you that the controller is constructed rather than accessed off the client, or that reaching a later optional parameter in the positional form needs `undefined` placeholders.'
---

# Calling endpoints on an APIMatic TypeScript SDK

Operations are **async methods on a controller class that you instantiate yourself** — they are *not*
properties on the client:

```typescript
import { Client, {Controller} } from 'paypal-server-sdklib';

const client = new Client({ /* ... */ });
const api = new {Controller}(client);          // you construct this
const response = await api.{operation}(/* ... */);
```

There is no `client.{apiGroup}.{operation}(...)` accessor and no operation sitting directly on the
client. Controllers live in `src/controllers/`, one file per API group, each extending a shared base
class. **Read the exported class name from that file** — the suffix is a generator setting, so the class
ends in `Api` or `Controller` depending on how the SDK was built. Operation names
follow no fixed verb/resource pattern — take the real name from the source.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{apiGroup}`, `{operation}`, `{resource}`, `{EnumType}`) — replace it with the concrete identifier from the source.

## Method signature convention

Every endpoint method is `async` (they all return a `Promise`), and there are **two parameter forms**.
Which one an operation uses is decided at generation time, so **read the method signature in
`src/controllers/` before writing the call** — both forms end with `requestOptions`.

```typescript
// Form A — positional, in declaration order
async {operation}(
  {requiredParam}: {RequestType},
  {optionalParam}?: string,
  requestOptions?: RequestOptions
): Promise<ApiResponse<{ReturnType}>>

// Form B — one destructured options object, when parameters are collapsed
async {operation}(
  { {requiredParam}, {optionalParam} }: { {requiredParam}: {RequestType}; {optionalParam}?: string },
  requestOptions?: RequestOptions
): Promise<ApiResponse<{ReturnType}>>
```

- **The rule:** an operation is generated in Form B when it has **more than one** non-constant parameter
  **and** the build sets `CollapseParamsToArray`, or the endpoint itself sets a collect-parameters flag.
  Everything else is Form A. A single-parameter operation is therefore always positional, even in a
  collapsed build. Do not infer the form from one operation — check the one you are calling.
- **In Form A**, to reach a later optional parameter, pass `undefined` for the ones you skip (see the
  next section). **In Form B** you simply omit the key, and the skipping problem does not arise.
- **Read the parameter list at the top of the method in `src/controllers/`** — order, names and
  optionality come from the operation itself, not from any fixed convention. `doc/controllers/{group}.md`
  lists the same parameters plus a runnable call.
- **`requestOptions`**: the **last** parameter. It currently carries only `abortSignal` for per-call
  cancellation — there is no per-request timeout or header override on it; timeouts, retries and headers
  are configured on the client (see `typescript-configuration-resilience`).
- **Return type** varies by operation — see [Reading the response](#making-the-call-and-reading-the-response).
- Operations **throw `ApiError`** on non-2xx — see `typescript-error-handling`.

## List/search endpoints — mind the positional order

List/search operations often declare many optional parameters. **In Form A** they are positional, so you
must count them: pass `undefined` for every parameter you skip ahead of the one you want. (In Form B this
section does not apply — name the keys you want and omit the rest.)

```typescript
// {operation}(status?: {EnumType}, serviceLevel?: {EnumType}, createdAfter?: string,
//             limit?: number, cursor?: string, requestOptions?: RequestOptions)
const response = await api.{operation}(
  {EnumType}.SomeConstant,   // status
  undefined,                 // serviceLevel — skipped
  undefined,                 // createdAfter — skipped
  100                        // limit
);
```

**Open the method in `src/controllers/` and count the parameters** before writing the call — miscounting
binds your value to the wrong parameter and still compiles when the types happen to match.


## Building request models

Request bodies are plain objects conforming to a TypeScript interface. Required properties must be set; optional ones are `undefined` by default and are omitted from the JSON when not provided:

```typescript
const body: {RequestType} = {
  requiredProp: value,   // required — must be provided
  optionalProp: value,   // optional — leave out to omit from the request
};
```

A request body's **shape varies**: some are **flat** (scalar members directly on the object), others
**nest an inner resource object**. Open the request model interface under `src/models/` to see its real
required/optional members before writing the literal.

## Enums

Enums are generated as real TypeScript `enum` declarations exported from the SDK (member = wire value).
Reference the member — a bare string literal is **not** assignable to an enum-typed property:

```typescript
someProp: {EnumType}.SomeConstant;   // correct
someProp: 'server_provided_value';   // compile error — not assignable to {EnumType}
```

See **typescript-models** for the declaration shape and the unknown-value caveat.

## Union types, collections, and dates

Some properties are not plain scalars: discriminated union types, `Array<T>` collections, and date/time fields that are plain strings in a format the SDK neither converts nor checks. If a request property or response field is one of these, see **typescript-models** for how to construct and read it.

## Making the call and reading the response

Every operation returns `ApiResponse<T>` — the deserialized value is on `.result`, with
`.statusCode`, `.headers` and the raw `.body` alongside it:

```typescript
const response = await api.{operation}({pathOrQueryParam}, body);
console.log(response.statusCode);
const resource = response.result;
```

**The wrapped `T` varies per operation** — read the method's declared return type in the SDK source and
handle it accordingly:

- **A model** — `Promise<ApiResponse<{Payload}>>`: read `response.result`.
- **An array** — `Promise<ApiResponse<{ItemType}[]>>`: iterate `response.result`.
- **Nothing** — `Promise<ApiResponse<void>>`: `await` it and read `response.statusCode`.
- **Binary** — `Promise<ApiResponse<NodeJS.ReadableStream | Blob>>` for file/label downloads: stream or
  buffer `response.result`; it is not JSON.


**No operation in this API is paginated**, so no method returns a `PagedAsyncIterable` — every one is
`async` and awaited. Endpoints in the same family can still differ in the wrapped `T` — one a model, its
sibling an array — so let each method's declared return type guide how you read it.

## AbortSignal / cancellation

Pass an `AbortSignal` as the `abortSignal` member of `requestOptions` — the **last** parameter in both
forms, so how you reach it follows the form the operation was generated in (see [Method signature
convention](#method-signature-convention)):

```typescript
const controller = new AbortController();
setTimeout(() => controller.abort(), 30_000);

// Form A — async {operation}({id}: string, {optionalParam}?: string, requestOptions?: RequestOptions)
// pad the optionals you skip:
const a = await api.{operation}({id}, undefined, { abortSignal: controller.signal });

// Form B — async {operation}({ {id}, {optionalParam} }: {...}, requestOptions?: RequestOptions)
// nothing to pad — requestOptions follows the options object:
const b = await api.{operation}({ {id} }, { abortSignal: controller.signal });
```

`abortSignal` is the **only** member of `RequestOptions`, and the key is `abortSignal`, not `signal`. An
aborted call rejects with `AbortError`, not `ApiError`.

## Worked example — a list/GET call

No operation in this API is paginated, so a list endpoint is an ordinary awaited call returning
`Promise<ApiResponse<{ItemType}[]>>`:

```typescript
// Form A signature (illustrative), read from src/controllers/:
//   {operation}(
//     filter?: string,
//     startDate?: string,
//     limit?: number,
//     requestOptions?: RequestOptions
//   ): Promise<ApiResponse<{ItemType}[]>>

const response = await api.{operation}('some_filter', undefined, 20);
for (const item of response.result) {
  console.log(item.id);
}
```

In Form B the same call names the keys instead — `api.{operation}({ filter: 'some_filter', limit: 20 })` —
with no `undefined` placeholders. To walk a long list, drive the endpoint's own paging parameters yourself
and stop when a response comes back with fewer items than you asked for; see
**typescript-configuration-resilience**.

## Finding the right method in the SDK source

Read these from the SDK **source** files, not by inspecting the compiled `.d.ts` only — the source has JSDoc comments, the full params interface, and the request-builder internals.

- Every operation lives on a **controller class** in `src/controllers/`, one file per API group. `ls` that
  directory to find the group, then read the exported class name off its `export class` line — the
  suffix is a generator setting (`Api`, `Controller`, …), so do not assume it.
- Request/response/enum types live under `src/models/`; error types under `src/errors/`.

## Next

- Errors and status codes → **typescript-error-handling**
- Retries, timeouts, logging → **typescript-configuration-resilience**
- Union types, collections, dates, enums → **typescript-models**
