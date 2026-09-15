---
name: 'ruby-error-handling'
description: 'Handle errors from an APIMatic-generated Ruby SDK — an error response raises. The classes involved are `APIException` and the typed subclasses under `exceptions/`, whose attributes are unboxed from the response body, and Faraday transport failures never surface as `APIException`. Use the moment you write a begin/rescue around a call on the PayPal Server SDK Ruby SDK, translate its errors into your own, or need a status code — load it even after reading the raised class in the source, since the class name alone won''t tell you the rescue order that keeps a typed handler reachable, or where the status code actually lives.'
---

# Error handling for an APIMatic Ruby SDK

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{operation}`,
> `{controller_name}`, `{OperationException}`) — replace it with the concrete identifier from the source.

Endpoint methods **raise** on error responses. Everything they raise descends from
`PaypalServerSdk::APIException`, which extends `CoreLibrary::ApiException` from the `apimatic_core` gem —
so a single `rescue PaypalServerSdk::APIException` is a complete safety net for API errors.



## Which exception does an operation raise?

Two shapes, and the difference is generated per operation:

- **Typed subclass** — when the spec documents an error model for a status code, the generator emits a
  class under `lib/paypal_server_sdk/exceptions/` (`class {OperationException} < APIException`) and the
  operation registers it for that status. The instance carries the error body's fields as ordinary
  attributes.
- **Base `APIException`** — everything else. This is the common case: many operations document no error
  model at all.

You do not have to guess. Two places tell you:

1. **`doc/controllers/{controller}.md`** — each operation has an **Errors table** mapping HTTP status
   code → description → exception class. Read it first — but it is generated from the spec rather than
   from the SDK's own registrations, so **confirm each class it names actually exists** under
   `lib/paypal_server_sdk/exceptions/` before you rescue it.
2. The operation in the controller folder under `lib/paypal_server_sdk/` — where this build registers
   errors at all (the callout below settles that), its response handler holds a
   `.local_error('{status}', '{message}', {ExceptionClass})` line per documented error, and a
   `GLOBAL_ERRORS` constant on the controller base class supplies the catch-all. The folder
   (`controllers/`, `apis/`, …) is named from the SDK's controller-namespace setting and the base class
   (`BaseController`, `BaseApi`, …) from its `ControllerPostfix` setting, so
   `grep -rn GLOBAL_ERRORS lib/paypal_server_sdk/` for the real constant rather than assuming either name.

> **Both halves of that registration exist because this SDK was generated to raise for HTTP error
> statuses.** The client wires the `GLOBAL_ERRORS` constant into its global configuration — the
> `'default'` entry is what turns *any* unsuccessful response into `APIException` — and each response
> handler carries its own `.local_error(...)` lines for the statuses the spec documents. **One setting
> gates both**, so they are present or absent together: there is no build in which only the declared
> statuses raise. `grep -rn global_errors lib/paypal_server_sdk/client.rb` shows the wiring.

## Rescue the exception
Ruby matches `rescue` clauses top-down, and every typed exception is a **subclass** of `APIException`.
Put the typed clauses first — an `APIException` clause above them swallows everything:

```ruby
require 'paypal_server_sdk'

begin
  result = client.{controller_name}.{operation}
  # use result
rescue PaypalServerSdk::{OperationException} => e
  # typed: the error body's fields are attributes on e — read the class under exceptions/ for the names
  warn "typed error: #{e.{error_attribute}}"
rescue PaypalServerSdk::APIException => e
  # everything else the API returned
  warn "API error: #{e}"
end
```

What you can read off the raised object:

- **`e.message`** — on the base `APIException`, the reason the exception was raised with (the
  `error_message` registered for that status, e.g. `'HTTP response not OK.'` for the global catch-all),
  *not* the API's error text. **On a typed subclass this inverts:** when the error model happens to have a
  `message` field, the generated `attr_accessor :message` shadows `Exception#message`, so `e.message`
  returns the API's error text — and, when the body carried no `message`, the `nil`-or-`SKIP` value
  described below rather than the base class's reason. Check the class under `exceptions/` for an
  `attr_accessor :message` before relying on either meaning.
- **`e.to_s` / `e.inspect`** — on the base `APIException`, a summary including the **status code** and
  reason, so `puts e` is the fastest way to see what came back. Typed subclasses **override both** to
  print their unboxed model fields only — no status code, empty when the body did not parse, and
  `#<Object:0x...>` for each field the `SKIP` sentinel below is holding. Use the routes below when you
  need the status from a typed error.
- The base class also carries the reason and the underlying `HttpResponse`. **Read
  `lib/paypal_server_sdk/exceptions/api_exception.rb` and the `CoreLibrary::ApiException` it extends for
  the exact accessor names** before reaching for the status code or the raw body off the exception —
  they come from the gem, not from generated code.
- **Typed subclasses only** — one attribute per field of the error model, populated by `unbox` from the
  parsed response body in the constructor. `doc/models/{operation-exception}.md` lists them.
- A typed exception is only populated from a response body that parses as the error model, and **an
  unset attribute is not always `nil`**. When the body is not JSON at all, `unbox` returns before
  assigning anything and every attribute reads `nil`. When it parses but a key is missing, only
  attributes the error model marks *required* fall back to `nil`; an **optional** one is left holding the
  class's private `SKIP` sentinel — a bare `Object`, so `e.{error_attribute}.nil?` is never true,
  `e.{error_attribute}&.each` raises `NoMethodError`, and interpolating it prints `#<Object:0x...>`.
  `SKIP` is a `private_constant`, so you cannot compare against it: read the class's `unbox` under
  `exceptions/` to see which attributes get `nil` and which get the sentinel, and guard by type
  (`e.{error_attribute}.is_a?(String)`) rather than by nil check.

### Getting the status code reliably

If you need the status code as data rather than as text, take it from a place that documents it:
- **`ApiResponse#status_code`** — this SDK returns complete responses, so every successful non-paginated
  call hands you the status code directly (see **ruby-calling-endpoints**). `ApiResponse` also answers
  `success?` / `error?` and exposes `errors`, so you can branch on the response instead of rescuing.
- **`http_callback`** — an `HttpCallBack` whose `on_after_response(response)` receives the
  `HttpResponse`, which documents `status_code`, `reason_phrase`, `headers` and `raw_body`. This fires
  for successes and failures alike, which makes it the seam the generated tests use (see
  **ruby-testing**).

## Do not swallow non-API failures

`rescue PaypalServerSdk::APIException` catches API errors only. Transport failures — connection refused,
DNS failure, TLS error, a timeout — come out of **Faraday**, not the SDK, and are not `APIException`.
Rescuing `StandardError` around a call to be safe will therefore hide bugs in your own code as well;
rescue the Faraday error classes explicitly if you need to handle transport failures, and let anything
else propagate.

## Notes

- Retries for transient statuses happen **before** the exception is raised, and only for the status
  codes and HTTP methods baked into this SDK's `Configuration` defaults. Those defaults commonly exclude
  `POST`/`PATCH`/`DELETE`, whose errors then surface with no retry at all — see
  **ruby-configuration-resilience**.
- A call can also fail **before any request is sent**, when the configured auth scheme cannot be
  satisfied. Each generated auth class defines `error_message`, naming the missing or expired credential;
  see **ruby-authentication**.
- When you translate SDK errors into your own domain errors, do the translation once in a wrapper around
  the client rather than at every call site, and keep the original exception as the cause.
