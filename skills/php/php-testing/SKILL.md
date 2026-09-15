---
name: 'php-testing'
description: 'Unit-test code that calls an APIMatic-generated PHP SDK — the client builder exposes no HTTP-client injection point and the base URL cannot be overridden, so the seams are your own interface over the SDK, PHPUnit test doubles of the generated (non-final) controller and client classes, and the `httpCallback(…)` hook for observing real traffic. Covers stubbing success and error paths, asserting the outgoing request, and why the SDK''s own generated `tests/` directory is not a template for yours. Use when writing, mocking or stubbing tests for calls made through the PayPal Server SDK PHP SDK — load it even after reading the builder in the source, since the setter list won''t tell you that there is no transport seam at all.'
---

# Testing code that uses an APIMatic PHP SDK

**Start from the constraint:** the client builder has **no HTTP-client setter**, no handler stack, no
adapter and no base-URL option. Nothing in `src/` lets you swap the transport. Any advice that starts
"inject a mock HTTP client" does not apply to these SDKs — check `src/{Client}Builder.php` and you will
find only configuration values, credentials builders, `httpCallback` and `proxyConfiguration`.

So the seam is **above** the SDK, not inside it.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{Client}`,
> `{Group}`, `{operation}`) — replace it with the concrete identifier from the source. `{Postfix}` is the
> `ControllerPostfix` generation setting: it is `Controller` by default but can be anything (`Api`,
> `Client`, …). Read the real accessor and class names from `src/{Client}.php` and the controller folder
> (`src/Controllers/` or `src/Apis/`, whichever `ls src/` shows); do not assume `Controller`.

**Match the project's existing test stack — don't impose one.** Check the consuming project's
`composer.json` and existing tests first. The examples below use PHPUnit purely for reference; they show
*what* to assert and *where* to cut, not a mandated framework.

## Preferred seam — your own interface

Wrap the operations you actually use behind a narrow interface you own, and fake that. This keeps your
tests independent of SDK internals and survives regeneration:

**Every operation in this SDK returns an `ApiResponse` wrapper** — it sets `ReturnCompleteHttpResponse`, so
the gateway unwraps it, and its status check is the whole HTTP-failure path because a non-2xx never raises
(see **php-calling-endpoints**).

```php
interface {Resource}Gateway
{
    /** @return {Model}[] */
    public function list{Resource}(): array;
}

final class Sdk{Resource}Gateway implements {Resource}Gateway
{
    public function __construct(private {Client} $client) {}

    public function list{Resource}(): array
    {
        // Wrap this in a try/catch for ApiException as well when
        // src/Exceptions/ApiException.php exists, to cover transport failures.
        // getResult() holds the error payload on a failure, so returning it straight
        // from a method typed `: array` is a TypeError.
        $response = $this->client->get{Group}{Postfix}()->{operation}();
        if (!$response->isSuccess()) {
            throw new {Resource}GatewayException(/* your own type, from getStatusCode() + getResult() */);
        }
        return $response->getResult();
    }
}
```

Test your application against `{Resource}Gateway`; test `Sdk{Resource}Gateway` itself separately (or
against the real API in a small integration suite). The gateway is also where error translation belongs —
it is the one place that knows how to unwrap an `ApiResponse` and what a failed one
means.

## Doubling the SDK classes directly

The generator emits **no `final` classes and no `final` methods**, so PHPUnit can double a controller or
the client itself. `createMock()` does not invoke the constructor, which matters because the client's
constructor builds a real HTTP client:

```php
$controller = $this->createMock({Group}{Postfix}::class);
$controller->method('{operation}')->willReturn($apiResponseDouble);
// The stub must match the declared return type, so a bare array on a method typed
// `: ApiResponse` fails at stub time — build the double as shown further down.

$client = $this->createMock({Client}::class);
$client->method('get{Group}{Postfix}')->willReturn($controller);

$service = new MyService($client);
```

Build the expected models with their real builders (see **php-models**) so the shapes stay honest:

```php
$expectedModel = {Model}Builder::init(/* required fields */)->build();
```

### The error path — first, how does a failure surface?
**A bad status does not raise in this SDK.** Every operation returns the wrapper and a non-2xx comes back
inside it — a `willThrowException` stub for a 4xx/5xx exercises a path production code never takes. So stub
the **returned failure** for every status case.

One file is still worth checking, because it decides whether anything is throwable at all:
`src/Exceptions/ApiException.php`. Where it exists a **transport** failure still raises `ApiException`;
where it is absent nothing raises and the classes under `src/Exceptions/` are plain error models. Cover
both paths that your build actually has. See **php-error-handling**.

*Wherever `src/Exceptions/ApiException.php` exists — throw what the real SDK throws*
(see **php-error-handling**). Here that is the transport-failure test and nothing else: a non-2xx never
raises in this SDK. `ApiException`'s
constructor takes `(string $reason, HttpRequest $request, ?HttpResponse $response)`, so the cheapest
faithful stand-in for a transport failure is a `null` response — which is also the case whose
`getCode()` is `0`:

```php
$controller->method('{operation}')
    ->willThrowException(new ApiException('Boom', $this->createMock(HttpRequest::class), null));
```

That transport case is the **only** throwing path to cover here — there is no status-carrying
`ApiException` to stub, because a non-2xx comes back inside the wrapper instead.

Cover the **typed subclass** as well: this API documents at least one error model, so `src/Exceptions/`
carries a class for it, and a catch ladder in the wrong order (base before typed) only shows up in a test
that throws the typed one. It reaches a test the same way `ApiException` does — so if nothing above raises
on a status, nothing raises this either, and it arrives as a payload instead.

*Stub a returned failure for the status path.*
`isSuccess()`, `isError()`, `getStatusCode()` and `getResult()` are all non-final, so a
`createMock(ApiResponse::class)` can stand in:

```php
$failure = $this->createMock(ApiResponse::class);
$failure->method('isSuccess')->willReturn(false);
$failure->method('getStatusCode')->willReturn(409);
$failure->method('getResult')->willReturn($conflictErrorModel);   // or the raw decoded array — see below

$controller->method('{operation}')->willReturn($failure);
```

Every non-2xx arrives this way in a wrapper build, so assert that your gateway translated the failure.
What `getResult()` should return in the stub depends on the same second file: with `ApiException.php`
absent it is the deserialized error model shown above; with `ApiException.php` present nothing maps the
failure body, so stub the **raw decoded array** instead — a mock returning a model there hides exactly the
"call to a member function on array" bug the test exists to catch. See *"errors arrive inside the
response"* in **php-error-handling**. (If you need a real instance rather than a double,
`ApiResponse::createFromContext($decodedBody, $result, HttpContext $context)` is the factory.)

## Observing real traffic — `httpCallback`

`httpCallback(…)` is the SDK's only introspection hook. It takes a `CoreCallback`; the SDK's own
`PaypalServerSdkLib\Http\HttpCallBack` is that class, constructed with two optional callables:

```php
use PaypalServerSdkLib\Http\HttpCallBack;

$captured = null;

$client = {Client}Builder::init()
    ->httpCallback(new HttpCallBack(
        function ($request) use (&$captured): void {
            $captured = $request;                    // HttpRequest, before it is sent
        },
        function ($context) use (&$captured): void {
            $captured = $context;                    // HttpContext: getRequest() + getResponse()
        }
    ))
    ->build();
```

Assert off the captured objects — `getHttpMethod()`, `getQueryUrl()`, `getHeaders()`, `getParameters()`
on the request; `getStatusCode()`, `getHeaders()`, `getRawBody()` on the response.

> **`httpCallback()` silently ignores anything that is not a `CoreCallback`** — its body is
> `if (!$httpCallback instanceof CoreCallback) { return $this; }`. A hand-rolled closure or a plain
> object is accepted by PHP and then does nothing, with no error. Use `HttpCallBack`.

This is an **observation** hook, not a stub: the request is still sent. It belongs in integration tests.

## Asserting URL resolution without a request

```php
$this->assertSame('https://expected.host/base', $client->getBaseUri());
```

Cheap, offline, and it catches the single most common misconfiguration — the wrong `Environment`
constant.

## HTTP-level fakes

Because the base URL is derived from `Environment` constants and nothing else, you can only point the
SDK at a local fake server if the API itself declares an environment (or a server parameter) whose URL
you control. Check `src/Environment.php` and `src/{Client}Builder.php`. If neither exists, an HTTP-level
fake is not reachable from configuration — use the interface or double seams above rather than editing
the SDK or patching `/etc/hosts`.

Where a suitable environment *does* exist, point at it and keep the failure fast:

```php
$client = {Client}Builder::init()
    ->environment(Environment::{LOCAL_CONSTANT})
    ->timeout(2)                 // seconds
    ->enableRetries(false)       // a stubbed 5xx fails immediately instead of backing off
    ->build();
```

Retries are already off in a freshly generated SDK, but be explicit — a project-wide factory may have
turned them on.

## Do not copy the SDK's own `tests/` directory

If the SDK shipped with `tests/`, those are **live integration tests against the real API**, generated
from the spec's examples. They use a generated `tests/ClientFactory.php` plus `CallbackCatcher` and
`CoreTestCase` from `apimatic/core`, and they make real network calls with configuration and credentials
read from environment variables.
They are a useful reference for *how an operation is invoked*, not a model for unit-testing your own
code — and running them will hit the live API.

## Notes

- **Never assert on `ApiHelper::stringify` output or `__toString()`** — it is a readable dump, not a
  stable contract.
- To check what a request body will serialize to without sending anything,
  `json_encode($model)` — models implement `\JsonSerializable`.
- A success-path status code **is** available here — `getStatusCode()` on the returned `ApiResponse` — so
  assert it directly rather than reaching for `httpCallback`. Where the SDK genuinely exposes nothing you
  need, assert at the seam you own instead.
