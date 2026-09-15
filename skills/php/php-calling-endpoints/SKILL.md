---
name: 'php-calling-endpoints'
description: 'Call API operations on an APIMatic-generated PHP SDK — operations live on a controller you get from the client via `$client->get{Group}{Postfix}()`, and an operation''s parameters are either positional or collapsed into a single associative `array $options` keyed by camelCase parameter name, decided by the `CollapseParamsToArray` generator setting rather than by how many parameters there are; an optional positional parameter carries whatever default the spec gave it (not always `null`). Also covers building request models with their builders, passing enum constants, file uploads, and reading the `ApiResponse` wrapper every operation in this SDK returns. Use whenever invoking an endpoint, building a request body, or consuming a response from the PayPal Server SDK PHP SDK — load it even after reading the method signature in the source, since the signature won''t tell you the `$options` key names, that an enum parameter is typed `string`, or that the `ApiResponse` return means no error status will ever throw at you.'
---

# Calling endpoints on an APIMatic PHP SDK

Operations are methods on a **controller you get from the client** — you never construct one:

```php
$controller = $client->get{Group}{Postfix}();   // e.g. getShipmentsController() — or getShipmentsApi()
                                                // if ControllerPostfix renamed the suffix
$result = $controller->{operation}(/* … */);
```

The client memoizes each controller, so calling the accessor repeatedly is free. Controllers live in
`src/{Postfix}s/` — the directory and namespace are the *pluralised* `ControllerPostfix` CodeGen setting
(or the `ControllerNamespace` override), so `src/Controllers/` when the postfix is `Controller`,
`src/Apis/` when it is `Api` — and each extends a shared `Base{Postfix}` (`BaseController`, `BaseApi`, …).
Find the folder with `ls src/`: it is the one that is not `Models`/`Http`/`Utils`/`Authentication`/
`Exceptions`/`Logging`/`Proxy`. **Read the accessor names from `src/{Client}.php`.** Operation names
follow no fixed verb/resource pattern; take the real name from the source or from `doc/controllers/`.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{Group}`,
> `{Postfix}`, `{operation}`, `{Model}`, `{EnumClass}`) — replace it with the concrete identifier from
> the source.

## Two parameter conventions — check which one the method uses

Neither form is the norm and neither follows from the parameter count. An operation with **more than one**
non-constant parameter is collapsed into a single `array $options` when the SDK was generated with the
`CollapseParamsToArray` setting on (or when that endpoint opted in individually), and stays positional
otherwise; an operation with one parameter is positional either way. A whole SDK usually comes out one way,
so **read a signature in the controller folder before you write the first call** — do not carry a call
shape over from another SDK.

### 1. Positional parameters

```php
public function {operation}(
    string $requiredParam,
    ?int $optionalParam = null,
    ?{Model} $body = null
): {ReturnType}
```

- Required parameters come first and are typed non-nullably.
- Optional parameters are `?T`. Most default to `null`, but a parameter with a default in the spec is
  generated with **that** default (`?int $limit = 25`, `?bool $includeExpired = false`). **Copy the
  default out of the signature before skipping a parameter** — passing `null` past a non-null default
  omits it from the request instead of sending the default, silently changing the call. Prefer PHP 8
  named arguments, or re-supply the printed default, over padding with `null` (these SDKs still support
  PHP 7.2, so named arguments depend on your project's minimum version).
- **An enum parameter is typed `string` or `int` in the signature, not as the enum class.** The enum
  class only appears in the docblock and in `doc/controllers/*.md` (rendered as `?string(EnumName)`).
  Pass a constant off the generated class: `{EnumClass}::{CONSTANT}`. A closed enum's
  value is validated when the request is serialized, so an unlisted string fails there rather than at the
  call site.

### 2. A single `array $options` (the "collect parameters" mode)

In a collapsing build every parameter of such an operation — headers, query, path **and the request
body** — becomes one key of one associative array:

```php
public function {operation}(array $options): {ReturnType}
```

```php
$collect = [
    'body'        => $body,                        // the request model, when the operation has one
    'xSomeHeader' => 'value',
    'resultType'  => {EnumClass}::{CONSTANT},
    'limit'       => 50,
];

$result = $controller->{operation}($collect);
```

The keys are the **camelCased parameter names**. Nothing type-checks them, so a misspelled *optional*
key silently drops that parameter from the request rather than erroring. Get the exact key list from the
*Parameters* table in `doc/controllers/{group}.md`, or from the `->extract('…')` calls in the generated
method body.

### A trailing `?array $fieldParameters = null`

An operation whose endpoint accepts arbitrary extra **form** fields ends with
`?array $fieldParameters = null` — token endpoints generated from an OAuth scheme are the usual case.
Pass a plain `['key' => 'value']` map; the generated body wires it through
`AdditionalFormParams::init(...)`, so the entries go into the **request body**, not the query string.
It stays the last argument in a collapsing build too:
`{operation}(array $options, ?array $fieldParameters = null)`.

## Building request models

A request body is a generated model class. Required fields are constructor arguments on its **builder**;
optional fields are fluent setters:

```php
use PaypalServerSdkLib\Models\Builders\{Model}Builder;

$body = {Model}Builder::init(
    /* required fields, positionally, in the builder's own order */
)
    ->{optionalField}($value)
    ->build();

$result = $controller->{operation}($body);
```

`{Model}Builder::init(...)`'s parameter list *is* the required-field list. Anything not in it is
optional. See **php-models** for enums, unions, dates, collections and the optional-vs-null distinction.

`new {Model}(...)` also works and takes the same required arguments, but then you set optionals through
`set{Field}(...)` on the instance; the builder is what the generated docs use.

**File uploads** take a `FileWrapper` — but `src/Utils/FileWrapper.php` is generated **only** when some
operation declares a file parameter, and most APIs declare none. `ls src/Utils/` before you import it: a
`use` of a class that was never generated is a fatal PSR-4 autoload failure, not a warning.

```php
use PaypalServerSdkLib\Utils\FileWrapper;

$controller->{operation}(FileWrapper::createFromPath('/path/to/file.png'));
```

## Reading the response
**This SDK returns `ApiResponse`, on every operation.** It sets `ReturnCompleteHttpResponse`, so
`src/Http/ApiResponse.php` is generated, nothing here hands back the bare deserialized value, and the
generated methods carry no `@throws ApiException` annotation because in this shape an error status is not
raised at all. Confirm it on any method in the controller directory: the return type reads `ApiResponse`.

The wrapper exposes `getStatusCode(): ?int`, `getHeaders(): ?array`, `getResult()`, `isSuccess()`,
`isError()`, `getBody()` (the raw body) and `getRequest()`.

**An error *status* does not throw.** Every operation's handler chain ends `->returnApiResponse()`, and the
runtime returns the wrapper on a failure rather than raising. So `isSuccess()` is the check that matters,
and that is why the generated `doc/controllers/*.md` examples branch on it with no `try/catch`. What still
raises is a **transport** failure — DNS, refused connection, TLS, timeout — as `ApiException` with a
`getCode()` of `0`, wherever `src/Exceptions/ApiException.php` exists. See **php-error-handling**.

**On a failure, `getResult()` is typed only when the SDK was generated to map error types into the
wrapper** — the `PhpMapErrorTypesInCompleteResponse` setting, which makes each handler chain also call
`mapErrorTypesInApiResponse()`. Without that call, and it is commonly absent, a failure leaves
`getResult()` holding the **raw decoded body** — a plain PHP array, not an error model — so
`->getMessage()` on it is a fatal "call to a member function on array". That call and
`src/Exceptions/ApiException.php` are two faces of one setting, so either check answers it:
`grep -rl mapErrorTypesInApiResponse src/` finds it on **every** operation when `ApiException.php` is
absent and on **none** when it exists. In the no-mapping build, read the error out of `getResult()` as an
array or take `getBody()` and decode it yourself.

### Worked example — a list call

No error status throws, so the status check *is* your handling for HTTP failures. Keep a `try/catch`
around it only for transport failures, and only where `src/Exceptions/ApiException.php` exists:

```php
$apiResponse = $controller->{operation}({EnumClass}::{CONSTANT}, 50);

if (!$apiResponse->isSuccess()) {         // every non-2xx lands here, mapped or not
    $error = $apiResponse->getResult();   // a mapped error model ONLY if this SDK maps error
                                          // types into the wrapper — otherwise a raw array
    // translate into your own error — see php-error-handling
    return;
}
foreach ($apiResponse->getResult() as $item) {
    echo $item->get{Field}(), PHP_EOL;
}
```

> **`ApiException` is not present in every SDK** — check whether `src/Exceptions/ApiException.php`
> exists. If it does not, this SDK maps errors into the response: `getResult()` on a failure is the
> deserialized error model, the classes under `src/Exceptions/` are those *models* rather than throwables,
> and a `try/catch` around the call catches nothing at all — not even a transport failure.

Wherever you catch `ApiException`, **import the class**: `use PaypalServerSdkLib\Exceptions\ApiException;`
at the top of the file. Written inline as `catch (PaypalServerSdkLib\Exceptions\ApiException $e)` inside a
namespaced file the name resolves **relative to the current namespace**, so the catch never matches.
Import it, or root-anchor it as `\PaypalServerSdkLib\Exceptions\ApiException`.

**The status→class map lives in the handler chain.** This SDK sets `ThrowForHttpErrorStatusCodes`, so each
operation carries `->throwErrorOn('<status>', ErrorType::init(…))` entries naming the generated error class
for each documented status. Whether they actually raise depends on the return shape above — in the wrapper
shape they select a mapping target at most and never a throw. See **php-error-handling**.

> This SDK does **not** set `Nullify404`, so no operation converts a `404` into `null` — a `404` surfaces
> exactly like any other non-2xx (see **php-error-handling**), and no `nullOn404()` appears anywhere in
> the controllers. Where a return type *is* nullable, that is the API's own optional response model.

## Paging a list endpoint
**This API declares no paginated operation**, and PHP SDKs generate no pagination support in any case — no
iterator, no page wrapper, no `paginate()`. A list endpoint here is an ordinary operation: if it takes
`page`/`offset`/`cursor`/`limit` parameters, advance them yourself. See **php-configuration-resilience**
for the loop shape.

## Finding the right method in the SDK source

- `doc/controllers/{group}.md` — fastest route: the operation list, a parameters table (with the
  `$options` key names and any enum type), the response type, and a runnable example. Start here.
- `src/{Postfix}s/{Group}{Postfix}.php` (e.g. `src/Controllers/ShipmentsController.php`, or
  `src/Apis/ShipmentsApi.php` in a renamed build) — the exact signature, plus
  the request-builder body showing which parameter maps to a header, a query parameter, a path segment or
  the form/body.
- `src/{Client}.php` — the `get{Group}{Postfix}()` accessor names.

## Next

- Request/response field shapes → **php-models**
- Errors and status codes → **php-error-handling**
- Retries, timeouts, manual paging, logging → **php-configuration-resilience**
