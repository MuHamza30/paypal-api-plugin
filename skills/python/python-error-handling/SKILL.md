---
name: 'python-error-handling'
description: 'Handle errors from the PayPal Server SDK Python SDK. Load before your first `try/except` around a call, or when translating SDK errors into your own. The raised class won''t tell you the status code is on `response_code`, that a typed subclass''s attributes may be unassigned when the error body isn''t a JSON object, or that catching the base class first swallows typed errors.'
---

# Error handling for an APIMatic Python SDK

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{operation}`,
> `{controller}`, `{Name}Exception`) — replace it with the concrete identifier from the source.

Operations **raise on non-success responses**. Everything raised by the SDK derives from `ApiException`
(`paypalserversdk/exceptions/api_exception.py`), which carries:

| Attribute | Meaning |
| --- | --- |
| `response_code` | the HTTP status code — **not** `status_code` |
| `reason` | the generated error message for the case that matched |
| `response` | the `HttpResponse`: `status_code`, `reason_phrase`, `headers`, `text`, `request` |

The raised class comes in **two shapes**, depending on the operation:

- **Typed subclass (Case A)** — the API documents an error model for that status, so a
  `{Name}Exception(ApiException)` exists under `paypalserversdk/exceptions/` and its fields are unboxed
  onto the exception instance.
- **Base `ApiException` (Case B)** — no error model for that response, so what you catch is
  `ApiException` itself. This is the common case.

## Which exception does an endpoint raise?

Open the operation's page under `doc/controllers/`. An operation that documents error models ends its
section with an **Errors** table mapping HTTP status codes (and a `Default` row, when it has one) to the
exception class raised for them. **No table — or no `doc/controllers/` page for that controller at all —
is not proof that nothing typed is raised:** grep `.local_error(` in the controller's own module under
`paypalserversdk/controllers/` for the status-to-class mapping the SDK actually registers.

A status no `.local_error(` covers falls through to the SDK's global default case, generated as a single
`default` entry naming a message and an exception class — the base `ApiException` unless the
spec documents a global error model, in which case a typed subclass sits there instead. Read
`BaseController.global_errors` on the `BaseController` class in
`paypalserversdk/controllers/` for the class and the wording rather than assuming
either.

## Catch the exception

### Case A — the operation has a typed exception

```python
from paypalserversdk.exceptions.api_exception import ApiException
from paypalserversdk.exceptions.{name}_exception import {Name}Exception

try:
    result = client.{controller}.{operation}()
except {Name}Exception as e:
    # typed fields, unboxed from the error body — read the class for their names
    print(e.response_code, getattr(e, '{typed_field}', None))
except ApiException as e:
    print(e.response_code, e.response.text)
```

**Order matters.** `{Name}Exception` subclasses `ApiException`, so an `except ApiException` placed first
swallows every typed error and the typed branch never runs. List the typed classes first, base last.

### Case B — the operation raises `ApiException`

```python
from paypalserversdk.exceptions.api_exception import ApiException

try:
    result = client.{controller}.{operation}()
except ApiException as e:
    print(f'HTTP {e.response_code}: {e.reason}')
    print(e.response.text)                       # raw body
    print(e.response.headers.get('Retry-After')) # response headers live here
```

`e.response.text` is the **raw body string**; nothing has parsed it for you. Use
`APIHelper.json_deserialize(e.response.text)` (or `json.loads`) if you need it structured, and be ready
for it not to be JSON at all.

## The typed-exception trap

A typed exception unboxes the error body in its `__init__`, but only after checking that the parsed body
is a JSON **object**:

```python
dictionary = APIHelper.json_deserialize(self.response.text)
if isinstance(dictionary, dict):
    self.unbox(dictionary)
```

When the server returns an empty body, HTML from a proxy, or a bare JSON scalar — exactly what a gateway
timeout or a WAF block looks like — `unbox` never runs and **the typed attributes are never assigned**.
Reading `e.{typed_field}` then raises `AttributeError` *from inside your except block*, replacing a
useful error with a confusing one. Always reach for them with `getattr(e, '{typed_field}', None)`, and
fall back to `e.response.text`.

## Not every failure is an `ApiException`

- **Transport failures** — connection refused, DNS failure, TLS error, read timeout — come out of the
  underlying `requests` transport, not as `ApiException`. Catch them separately (or let them propagate)
  rather than assuming an `except ApiException` covers a network outage.
- **OAuth token acquisition** — an *explicit* `fetch_token()` raises the SDK's OAuth provider exception
  (generated per SDK — grep `exceptions/`, the casing varies). The **automatic** pre-call fetch swallows
  it and raises `AuthValidationException` from `apimatic_core` instead, which is **not** an
  `ApiException` subclass. `ls paypalserversdk/exceptions/` for the provider exception's real module —
  its casing is generated per SDK. See **python-authentication**.
- **Client-side validation of a oneOf/anyOf parameter** fails *before* any request is sent, so the
  traceback points at your call site with no `response` to inspect — see **python-models**.
- **Missing or incomplete credentials** are detected by the generated auth handler, which carries its
  own `error_message` (e.g. `"BasicAuth: username or password is undefined."`), rather than producing a
  `401` from the server. Grep that string in `paypalserversdk/http/auth/` when a call fails with a
  message that names a scheme.

Because these arrive as different types, a bare `except ApiException` around your integration is not a
catch-all. Decide deliberately what the other branches do.

## Reporting

`ApiException.__str__` renders as `{Name}Exception(status_code=..., message=...)`, and typed subclasses
append their unboxed fields. **But some typed subclasses read those unboxed attributes in `__str__`
without guarding them** — the generator is inconsistent, so check the class before you rely on it. Where
it is unguarded and the body was not a JSON object, `str(e)` raises `AttributeError` in exactly the
gateway/WAF case above. Log `f'{e.response_code}: {e.reason}'` instead, or guard `str(e)` with `try`.
Keep `e.response.text` out of logs that might contain user data.

## Notes

- The status code is also on `e.response.status_code`; `e.response_code` is the same value and is what
  the generated code and docs use.
- Retries for transient statuses happen **before** the exception reaches you, but only for the methods
  and status codes configured on the client, whose defaults are fixed when the SDK is generated. Errors
  from a method outside that set surface with no retry at all. See
  **python-configuration-resilience**.
- The status code of a **successful** call is already on `result.status_code`. See **python-calling-endpoints**.
