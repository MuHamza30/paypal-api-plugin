---
name: 'ruby-testing'
description: 'Unit-test code that calls an APIMatic-generated Ruby SDK — the SDK ships no mocking helpers, so the in-process seam is the `connection:` keyword argument (a Faraday connection you build with Faraday''s test adapter), while `http_callback:` is the observation seam the SDK''s own generated Minitest suite uses to assert status codes, headers and raw bodies. Covers stubbing success and error responses, asserting the outgoing request, asserting the right typed exception per operation, disabling retries so a stubbed 5xx fails fast, and substituting a stub client in your app''s wiring. Use when writing, mocking or stubbing tests for calls made through the PayPal Server SDK Ruby SDK — load it even after reading the constructor in the source, since the keyword list won''t tell you which argument is the seam or that `http_callback` cannot stub a response.'
---

# Testing code that uses an APIMatic Ruby SDK

The SDK has **no mocking helpers for consumers**. Its transport is **Faraday**, and the `Client`
constructor exposes two relevant keyword arguments:

- **`connection:`** — a `Faraday::Connection` you build yourself, used for every request. Building it
  with **Faraday's test adapter** is the in-process seam: no network, no global patching.
- **`http_callback:`** — an `HttpCallBack` whose `on_before_request` / `on_after_response` hooks
  **observe** the request and response. This is what the SDK's own generated tests use for assertions —
  it cannot stub anything.

**Match the project's existing test stack — don't impose one.** When the SDK was generated with tests, its
generated tests use Minitest and the gemspec adds `minitest` and `minitest-proveit` as development
dependencies; otherwise the gemspec carries runtime dependencies only and there is no `test/` directory.
Your application may use RSpec either way. Check the project first; the samples below use Minitest
**purely for reference** — they show the seam and *what* to assert, not a mandated framework.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g.
> `{controller_name}`, `{operation}`) — replace it with the concrete identifier from the source.

## A reusable stub helper

```ruby
require 'faraday'
require 'paypal_server_sdk'

def client_returning(status, body, path)
  stubs = Faraday::Adapter::Test::Stubs.new
  stubs.get(path) do
    [status, { 'Content-Type' => 'application/json' }, JSON.generate(body)]
  end

  connection = Faraday.new do |builder|
    builder.adapter(:test, stubs)
  end

  client = PaypalServerSdk::Client.new(
    connection: connection,
    max_retries: 0        # a stubbed 5xx fails on the first attempt, with no backoff wait
  )

  [client, stubs]
end
```

Faraday's test adapter is Faraday's own API, not the SDK's — check the Faraday version the SDK's gems
resolve to for the exact stub syntax, and use `stubs.verify_stubbed_calls` to assert every stub was hit.

## Test a success path

```ruby
def test_returns_deserialized_body
  client, = client_returning(200, [{ 'id' => 123, 'name' => 'Rex' }], '/pets')

  result = client.{controller_name}.{operation}

  assert_equal 123, result.data.first.{attribute}
end
```
**Reach through `.data`.** This SDK returns complete responses, so an operation hands back an `ApiResponse`
and the payload is one level down — `result.first` on the wrapper raises `NoMethodError`, and asserting
`result.status_code` alone passes without ever exercising the deserializer. Assert on `result.data`, and
assert the status code off the same object rather than through a callback.

Assert on the **deserialized model**, not the JSON — that is what exercises the SDK's mapping.



## Test an error path



Endpoint methods raise on error responses (see **ruby-error-handling**). Assert the **specific** class
the operation declares — the Errors table in `doc/controllers/{controller}.md` names it — rather than the
`APIException` base, or the test will pass for the wrong reason:

```ruby
def test_raises_typed_exception
  client, = client_returning(422, { 'code' => 422, 'message' => 'bad input' }, '/pets')

  error = assert_raises(PaypalServerSdk::{OperationException}) do
    client.{controller_name}.{operation}
  end

  assert_equal 'bad input', error.{message_attribute}
end
```

For an operation with no declared error model, assert `PaypalServerSdk::APIException` instead.

## Assert the outgoing request

Faraday's test stub block receives the request environment, so capture it there:

```ruby
def test_sends_correct_request
  captured = nil
  stubs = Faraday::Adapter::Test::Stubs.new
  stubs.post('/pets') do |env|
    captured = env
    [201, { 'Content-Type' => 'application/json' }, '{}']
  end
  connection = Faraday.new { |builder| builder.adapter(:test, stubs) }
  client = PaypalServerSdk::Client.new(connection: connection, max_retries: 0)

  client.{controller_name}.{operation}(body)
  # collapsed build (def {operation}(options = {})): {operation}('body' => body) — see ruby-calling-endpoints

  assert_equal :post, captured.method
  assert_includes captured.url.path, '/pets'
  assert_equal 'value', JSON.parse(captured.body)['expectedField']
end
```

Asserting the serialized body is worth doing at least once per model you build: an attribute you omitted,
or one whose wire name differs from the Ruby name, disappears silently (see **ruby-models**).

## Observe with `http_callback` instead

When you only need the status code, headers or raw body of a real (or stubbed) call, use the same
`HttpCallBack` pattern the SDK's generated tests use:

```ruby
class ResponseCatcher < PaypalServerSdk::HttpCallBack
  attr_reader :response

  def on_before_request(request); end

  def on_after_response(response)
    @response = response
  end
end

catcher = ResponseCatcher.new
client = PaypalServerSdk::Client.new(http_callback: catcher)

client.{controller_name}.{operation}

assert_equal 200, catcher.response.status_code
assert_equal expected_json, JSON.parse(catcher.response.raw_body)
```

`HttpResponse` exposes `status_code`, `reason_phrase`, `headers`, `raw_body` and `request`. On this SDK it is
a convenience rather than a necessity — complete responses are enabled, so a success path already gives you
`result.status_code` directly — but it is the only route to the raw body of a call whose payload was
deserialized, and it fires on failures too, so it is also how you inspect what an exception was raised
from.

## Notes

- **Disable retries in tests** (`max_retries: 0`) so a stubbed 5xx fails on the first attempt instead of
  waiting out the backoff. To test that retries *do* fire, stub the same path twice (503 then 200) and
  assert both stubs were consumed — but first check `retry_methods` in
  `lib/paypal_server_sdk/configuration.rb`, since a `POST` is commonly not retried at all.
- **A stubbed client is all you need.** Controllers are memoized readers on the client, not objects you
  construct, so there is nothing else to fake.
- **Substitute the client, not the controller.** Inject the `Client` (or your own wrapper over it) into
  the code under test and pass a stubbed one in tests — see the dependency-injection section of
  **ruby-client-initialization**.
- If the project already uses **WebMock** or **VCR**, stub at that level instead; the assertions above
  apply unchanged, and you keep one HTTP-stubbing mechanism across the suite.
- The SDK's own `test/` directory (present only when the SDK was generated with tests) is a working
  reference: `test/http_response_catcher.rb` plus a Minitest base class that builds the client with
  `Client.from_env(..., http_callback: HttpResponseCatcher.new)`.
