---
name: 'java-calling-endpoints'
description: 'Call API operations on an APIMatic-generated Java SDK — operations are methods on a controller you get from the client with `client.get{Controller}()`; a blocking method declaring `throws ApiException, IOException` is always generated, with an `...Async()` twin returning `CompletableFuture<T>` when the SDK was built in asynchronous mode; parameters are either positional or collapsed into a single `{Operation}Input` object — the `CollapseParamsToArray` setting decides it for every operation with more than one parameter, so neither form is the norm and you read the signature; and every return type is wrapped in `ApiResponse<T>`, because this SDK was generated in complete-response mode. Use whenever invoking an endpoint, building a request body, working out which parameters are required, or consuming a response from the PayPal Server SDK Java SDK — load it even after reading the method signature in the source, since the signature won''t warn you about the sync/async pair or the collapsed `Input` object.'
---

# Calling endpoints on an APIMatic Java SDK

Operations are methods on a **controller you get from the client**, not on the client itself:

```java
import {rootPackage}.{Api}Client;
import {rootPackage}.{controllerPackage}.{Controller};
import {rootPackage}.exceptions.ApiException;

{Api}Client client = new {Api}Client.Builder()./* ... */.build();
{Controller} controller = client.get{Controller}();

{ReturnType} result = controller.{operation}(/* ... */);
```

Controllers live in `<root>/<controllerPackage>/`, one class per API group. **The package name and the
class suffix are both generator settings** — `controllers`/`Controller` by default. The `ControllerNamespace`
setting renames the package outright and wins when it is set; otherwise a controller postfix renames the
package *and* the class suffix together. The suffix may also be absent entirely, so the accessor may be
`getPetsController()`, `getPetsApi()` or `getPets()`. **Take the package from the `import` block at the top of
`{Api}Client.java` and the accessor from its `get...()` methods**; `doc/client.md` lists them all.

**This SDK was generated without `GenerateInterfaces`**, so each group emits a single `final` class and
no interface: `{Controller}.java` *is* the implementation, and it is the file to open for an exact
signature. The client implements `Configuration` directly.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{Controller}`,
> `{operation}`, `{Model}`) — replace it with the concrete identifier from the source.

## Two method shapes: blocking and `...Async()`

The **blocking** form is always generated:

```java
public {ReturnType} {operation}({params}) throws ApiException, IOException
```

When the SDK was generated in asynchronous mode, each operation also gets a twin:

```java
public CompletableFuture<{ReturnType}> {operation}Async({params})
```

- The blocking form declares **two checked exceptions** — `ApiException` for any non-2xx response and
  `IOException` for transport failures. XML operations add `JAXBException`. You must handle or declare
  both; see **java-error-handling**.
- The async form declares **no** checked exceptions: it wraps everything into a `CompletionException`,
  so the real error is at `exception.getCause()`.
- Both forms exist side by side and share one request builder, so picking either is purely a style
  choice. The generated `doc/controllers/*.md` shows only one of them — open the `.java` file to see
  both.

```java
// blocking
try {
    {ReturnType} result = controller.{operation}(/* ... */);
} catch (ApiException e) {
    // non-2xx
} catch (IOException e) {
    // network/transport
}

// async
controller.{operation}Async(/* ... */)
        .thenAccept(result -> { /* ... */ })
        .exceptionally(exception -> {
            Throwable cause = exception.getCause();   // the real error
            return null;
        });
```

## Parameters — positional, or one collapsed `Input` object

Which form an operation uses is decided by the **`CollapseParamsToArray`** code-generation setting (or the
per-endpoint `CollectParameters` flag), applied to every operation with **more than one non-constant
parameter**. Neither form is the norm — in a build with collapsing on, nearly every multi-parameter
operation takes an `{Operation}Input`, and in a build without it none does. **Read the signature in the
controller class the client's `get...()` accessor returns** before you write the call.

An operation that was not collapsed — and every single-parameter operation, whatever the setting — takes
its parameters **positionally**, in the order the method declares them:

```java
{ReturnType} result = controller.{operation}(petId, status);
```

Optional parameters are still positional — pass `null` for the ones you want to omit. Boxed types
(`Integer`, `Long`, `Boolean`, `Double`) are used precisely so that `null` is expressible.

A collapsed operation instead takes a **single `{Operation}Input` object** bundling every parameter. The
parameter is literally named `input`:

```java
public {ReturnType} {operation}(final {Operation}Input input) throws ApiException, IOException
```

`{Operation}Input` is an ordinary generated model in `<root>/models/`, built the same way as any other —
required values in the `Builder` constructor, optional ones as fluent setters (see **java-models**):

```java
{ReturnType} result = controller.{operation}(new {Operation}Input.Builder(requiredA, requiredB)
        .optionalC(value)
        .build());
```

Two operations in the same controller can differ: with collapsing on, one that takes a single parameter
still takes it bare, so check each signature rather than generalising from the first.

Some operations append one or both of `final Map<String, Object> queryParameters` and
`final Map<String, Object> fieldParameters` after the regular parameters. Those are escape hatches for
additional, unmodelled query or form values; pass `null` when you have none.

There is **no per-call options argument** — no request options, no per-call timeout, no cancellation
token. Timeouts and retries are client-level; see **java-configuration-resilience**.

## Building request bodies

A body parameter is a generated model. Required properties go in the model `Builder`'s constructor,
optional ones are fluent setters:

```java
import {rootPackage}.models.{Model};

{Model} body = new {Model}.Builder(requiredProp)
        .optionalProp(value)
        .build();

{ReturnType} result = controller.{operation}(body);
```

Models also expose a public all-args constructor, so `new {Model}(a, b, c)` works too — but the
positional constructor breaks the moment a regeneration adds a field, while the `Builder` does not.
This SDK was generated **without** immutable models, so every field also has a plain setter and the
model has a no-arg constructor.

For enums, unions, dates, collections and nullable-optional
fields, load **java-models**.

## Reading the response


> **This SDK was generated in complete-response mode** (`ReturnCompleteHttpResponse`), so every
> non-paginated return type is wrapped in `ApiResponse<T>` from `<root>/http/response/`. That applies
> to the whole SDK at once.

```java
ApiResponse<{Model}> response = controller.{operation}(petId);
int status          = response.getStatusCode();
Headers headers     = response.getHeaders();
{Model} value       = response.getResult();
```

The payload is on **`getResult()`** — the extra hop is what people forget. An operation with **no
response body** is wrapped too: it returns `ApiResponse<Void>` in the blocking form and
`CompletableFuture<ApiResponse<Void>>` in the async one, so even a `DELETE` hands you a status code.

A binary or file response comes back as `ApiResponse<InputStream>` — read and close the stream
yourself.

## File uploads

File parameters are `FileWrapper` from `<root>/utilities/`:

```java
import {rootPackage}.utilities.FileWrapper;

FileWrapper upload = new FileWrapper(new File("./photo.png"), "image/png");
{ReturnType} result = controller.{operation}(upload);
```

The one-argument `new FileWrapper(file)` constructor also exists when the content type is implied.

## Pagination — none in this SDK

**This API declares no paginated operation.** The SDK ships no `<root>/utilities/pagination/` package,
no `PagedIterable`/`PagedFlux`/`PagedSupplier` types, and no operation returns one. A list endpoint here
returns the ordinary shape described above: drive its own page/offset/cursor parameters yourself and
stop on whatever the API uses to signal the end.

## Finding the right method in the SDK source

- Start with `doc/controllers/*.md` — it names the controller class, the accessor, every operation, its
  parameters, its response type and its documented errors, with a copy-pasteable sample.
- Then open the controller source itself — `{Controller}.java` in whatever package the client imports it
  from — for the exact signature: the parameter list (positional or `Input`), the `throws` clause, and
  the return type.
- Request/response/enum types are under `<root>/models/`; typed exceptions under `<root>/exceptions/`.

## Next

- Errors and status codes → **java-error-handling**
- Enums, unions, dates, nullable-optional fields → **java-models**
- Retries, timeouts, logging → **java-configuration-resilience**
