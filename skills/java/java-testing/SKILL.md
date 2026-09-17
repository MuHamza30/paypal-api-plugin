---
name: 'java-testing'
description: 'Unit-test code that calls the PayPal Server SDK Java SDK. Load before stubbing the SDK. The option list won''t tell you the in-process seam is your own `OkHttpClient` handed to `.httpClientInstance(...)`, that `HttpCallback` observes rather than substitutes, or that a stubbed 5xx needs `numberOfRetries(0)` to fail on the first attempt.'
---

# Testing code that uses an APIMatic Java SDK

The SDK ships **no mocking helpers for consumers**. Its transport is OkHttp
(`io.apimatic:okhttp-client-adapter`), and the injection point is
`httpClientConfig(b -> b.httpClientInstance(okHttpClient))`. Supplying an `OkHttpClient` whose
interceptor returns canned responses is the in-process seam: no network, no global patching, and no
dependence on the SDK's internals.

> There is **no base-URL override**. The URL comes from the `Environment`/`Server` enums, so pointing the
> client at a local mock server is not a matter of setting a property — either intercept inside OkHttp
> (below) or, if you really want a live socket, run a mock server and rewrite the request URL in an
> interceptor.

The samples below use JUnit 5 for reference only — mirror whatever the project already uses.

## What you can substitute, and what you cannot
**This SDK ships no interfaces, so there is nothing for a mocking library to
substitute.** The client is `public final` with a private constructor, and every controller is a
`public final` class with no interface behind it — most mocking libraries cannot subclass either without
extra configuration.

So put a narrow interface of your own in front of the SDK, implement it over the client — calling
`client.get{Controller}().{operation}(...)` and returning whatever your callers need — and test *that*
against the stub interceptor below. See **java-client-initialization**.

## A reusable stub helper

```java
import okhttp3.*;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.atomic.AtomicReference;

static final class Stub {
    final {Api}Client client;
    final AtomicReference<Request> lastRequest = new AtomicReference<>();

    Stub(int status, String body) {
        OkHttpClient okHttp = new OkHttpClient.Builder()
                .addInterceptor(chain -> {
                    Request request = chain.request();
                    lastRequest.set(request);
                    return new Response.Builder()
                            .request(request)
                            .protocol(Protocol.HTTP_1_1)
                            .code(status)
                            .message("")
                            .header("content-type", "application/json")
                            .body(ResponseBody.create(
                                    body.getBytes(StandardCharsets.UTF_8),
                                    MediaType.parse("application/json")))
                            .build();
                })
                .build();

        this.client = new {Api}Client.Builder()
                .httpClientConfig(configBuilder -> configBuilder
                        .httpClientInstance(okHttp)
                        .numberOfRetries(0))     // a stubbed 5xx fails on the first attempt
                // REQUIRED — this SDK declares a scheme; see below. Copy the setter name and the
                // model Builder's argument count from {Api}Client.Builder's own field initializers:
                .{scheme}Credentials(new {Scheme}Model.Builder(/* one "dummy" per required credential */)
                        .build())
                .build();
    }
}
```

> **Set every credential, even against a stub.** A blank credential fails auth validation — for a
> header/query/basic/bearer scheme before the HTTP client is ever called, and for an OAuth 2
> client-credentials grant only after a token fetch that goes *through your stub* (see the next
> paragraph). Either way the failure is an **unchecked** `AuthValidationException` — so a client built
> without one throws before your stubbed response can be returned, and every test in this skill fails with
> an error that says nothing about the thing you were testing. Read the real setter names off
> `{Api}Client.Builder`: most are `.{scheme}Credentials(model)`, but an API whose only scheme is an OAuth 2
> grant drops the suffix (`.clientCredentialsAuth(model)`).
>
> **For an OAuth 2 grant a dummy secret is not enough — seed a token.** Client-credentials,
> authorization-code and resource-owner-password managers validate on the *token*, not on the client
> id/secret (a plain bearer-token scheme does not — a non-blank dummy satisfies it). With no token,
> `validate()` fails and you get
> `AuthValidationException` instead of the response you stubbed. Worse, a **client-credentials** grant
> fetches one first, and it fetches it *through your stub* — the manager POSTs to the token endpoint, your
> interceptor answers that request, and a `Stub(422, ...)` built for an error-path test hands the `422` to
> the token call, where the resulting `ApiException` is swallowed and the token stays `null`. Attach a
> token on the model's `Builder` so no fetch happens and validation passes:
>
> ```java
> .clientCredentialsAuth(new ClientCredentialsAuthModel.Builder("dummy", "dummy")
>         .oAuthToken(new OAuthToken.Builder("stub-token", "Bearer").build())   // no expiry ⇒ never expired
>         .build())
> ```

The interceptor never calls `chain.proceed(...)`, so nothing leaves the process. Capturing
`chain.request()` gives you the fully built request — method, URL, headers, body — to assert on.

> `ResponseBody.create` swapped its argument order between OkHttp 3 and OkHttp 4. Check which version
> `io.apimatic:okhttp-client-adapter` pulls in (`mvn dependency:tree`) and flip the two arguments if the
> snippet does not compile.

## Test a success path

**This SDK was generated in complete-response mode**, so an operation returns `ApiResponse<{Model}>`
(from `<root>/http/response/`) and the payload is one hop further in, behind `.getResult()` — which also
means the status code is assertable directly:

```java
@Test
void returnsDeserializedBody() throws Exception {
    Stub stub = new Stub(200, "{\"id\": 123, \"name\": \"Rex\"}");
    {Controller} controller = stub.client.get{Controller}();

    ApiResponse<{Model}> response = controller.{operation}(/* args */);

    assertEquals(200, response.getStatusCode());
    assertEquals(123L, response.getResult().getId());
}
```

Assert on the **deserialized model** — that is what exercises the SDK's own mapping. No operation in
this SDK is paginated, so every operation follows the shape above.

The test method declares `throws Exception` because the blocking operation declares `ApiException` and
`IOException` — see **java-error-handling**.

## Test an error path

Which exception to expect depends on the operation: a typed subclass under `<root>/exceptions/` where the
spec gave that error a **modelled body**, base `ApiException` otherwise. At least one error here has such
a body, so both kinds are live. Take the type from the operation's own `.localErrorCase(...)` calls and
not from the `doc/` Errors table (**java-error-handling**). **Assert the concrete type, not `ApiException`** — every typed subclass derives from it, so a test
expecting `ApiException` passes for all of them and proves nothing.

```java
@Test
void throwsTypedErrorOnDocumentedFailure() {
    Stub stub = new Stub(422, "{\"code\": 422, \"message\": \"bad input\"}");
    {Controller} controller = stub.client.get{Controller}();

    {Operation}Exception e = assertThrows({Operation}Exception.class,
            () -> controller.{operation}(/* args */));

    assertEquals(422, e.getResponseCode());
    assertEquals("bad input", e.getMessageField());   // NOT getMessage()
}

@Test
void throwsApiExceptionOnUndocumentedFailure() {
    Stub stub = new Stub(500, "{}");
    {Controller} controller = stub.client.get{Controller}();

    ApiException e = assertThrows(ApiException.class,
            () -> controller.{operation}(/* args */));

    assertEquals(500, e.getResponseCode());
}
```

For an `...Async()` operation, the failure arrives wrapped: assert on `ExecutionException.getCause()`
(from `future.get()`) or `CompletionException.getCause()` (from `join()`).

## Assert the outgoing request

```java
@Test
void sendsCorrectRequest() throws Exception {
    Stub stub = new Stub(200, "{}");
    stub.client.get{Controller}().{operation}(/* args */);

    Request request = stub.lastRequest.get();
    assertEquals("POST", request.method());
    assertTrue(request.url().encodedPath().endsWith("/expected/path"));
    assertEquals("value", request.url().queryParameter("expectedParam"));
}
```

To read the serialized body, buffer it:

```java
okio.Buffer buffer = new okio.Buffer();
request.body().writeTo(buffer);
assertTrue(buffer.readUtf8().contains("\"expectedField\":\"value\""));
```

## The other seam: `HttpCallback`

If you would rather observe than stub — for an integration test against a real or mock server — register
an `HttpCallback` on the client and capture the request/response pair:

```java
final AtomicReference<Context> captured = new AtomicReference<>();

{Api}Client client = new {Api}Client.Builder()
        .httpCallback(new HttpCallback() {
            @Override public void onBeforeRequest(Request request) { }
            @Override public void onAfterResponse(Context context) { captured.set(context); }
        })
        .build();

// after a call:
assertEquals(200, captured.get().getResponse().getStatusCode());
```

The parameters are the runtime's `Request`/`Context` interfaces — **not** the SDK's own `HttpRequest` /
`HttpResponse` — but no cast is needed for what you assert on. Take the imports from the `Callback`
interface the SDK's `HttpCallback` extends, and note that this `Request` is a **different type** from
`okhttp3.Request` in the stub above.

## Notes

- **Keep `numberOfRetries(0)` in the stub**, so a stubbed `5xx` fails on the first attempt even if the
  code under test sets a count on the same builder. To test that retries *do* fire, set a count yourself
  and count interceptor invocations while returning `503` then `200` (see
  **java-configuration-resilience**).
- **Controllers come from the stubbed client** (`stub.client.get{Controller}()`), so a stubbed client is
  all you need — there is nothing else to fake.
- **For DI-based code**, override the client bean in the test context (Spring:
  `@TestConfiguration` with a `@Bean` returning the stubbed client, or `@MockBean` on your own wrapper). Prefer stubbing the transport over mocking the SDK types — the client and every controller are `public final` with no interface, which most mocking libraries cannot subclass without extra configuration.
- To look up an operation's signature or its request type, read the SDK
  source: the controller class the client exposes through its `get...()` accessor (its package is named
  per SDK — `<root>/controllers/` by default, but a controller postfix or a renamed controller namespace
  moves it), plus `<root>/models/` and `<root>/exceptions/`. The doc page
  for the same group is the alternative: a controller postfix moves only the source package, but a
  renamed controller namespace moves the doc folder too — `ls doc/` rather than assuming
  `doc/controllers/`.
