---
name: 'ruby-configuration-resilience'
description: 'Tune an APIMatic-generated Ruby SDK client — retries (`max_retries`, `retry_interval`, `backoff_factor`, `retry_statuses`, `retry_methods`) whose defaults are baked in at generation time so you must read `Configuration#initialize` rather than assume them, the Faraday `timeout`/`connection`/`adapter` transport arguments, `ProxySettings`, environment selection with no free-form base URL, and the built-in `logging_configuration` this SDK was generated with. Use whenever adjusting retry policy, timeouts, transport, the environment, paging, or logging on the PayPal Server SDK Ruby SDK — load it even after reading the keyword arguments in the source, since the list does not reveal that the retry defaults vary per SDK, that `retry_methods` holds symbols, or that some operations are generated to retry regardless of that list.'
---

# Configuration & resilience for an APIMatic Ruby SDK

Every setting is a **keyword argument passed at construction** (see **ruby-client-initialization**).
There is no options object and nothing you can change on a live client — derive a new configuration with
`config.clone_with(...)` and build a new client instead.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{Name}` for an
> `Environment` constant, `{controller_name}`, `{operation}`) — replace it with the concrete identifier
> from the source.

```ruby
client = PaypalServerSdk::Client.new(
  environment: PaypalServerSdk::Environment::{Name},
  timeout: 30,
  max_retries: 3,
  retry_interval: 1,
  backoff_factor: 2,
  retry_statuses: [408, 429, 500, 502, 503, 504],
  retry_methods: %i[get put],
  proxy_settings: PaypalServerSdk::ProxySettings.new(address: 'http://localhost', port: 8888)
)
```

## Retries

The five retry arguments above are the complete set. **Their defaults are fixed when the SDK is
generated, not by the runtime** — `max_retries`, `retry_statuses` and `retry_methods` in particular differ
from SDK to SDK because they come from the code-generation settings for that build.

> Do not assume a default. Read the keyword-argument defaults on `Configuration#initialize` in
> **`lib/paypal_server_sdk/configuration.rb`** — that is the generated, authoritative value for this SDK,
> and `doc/client.md` restates each one in a **Default:** column. Retries may well be off
> (`max_retries: 0`).

Notes:

- `retry_methods` holds **symbols**, written with the `%i[...]` literal (`%i[get put]`), not strings.
- Only the methods in that list are retried. If `post`/`patch`/`delete` are absent — the common case —
  those failures surface with no retry. Add one only if the operation is idempotent.
- An operation the spec marks as retriable is generated with `.endpoint_context('forced_retry', true)`
  and is retried **regardless of `retry_methods`**. Run `grep -rn forced_retry lib/paypal_server_sdk/` to
  see whether any operation you call does this — the controller files sit in a folder named after this
  SDK's controller namespace (`controllers/`, `apis/`, …), so do not assume the path.
- `retry_interval` is the pause in seconds before the first retry; `backoff_factor` multiplies it for
  each subsequent attempt.
- `retry_statuses` is a plain `Array` of integers.
- **Retries are off unless this build turned them on — check, do not assume either way.** The count is
  the `Retries` code-generation setting baked into `Configuration#initialize`: `0` in a default build,
  but specs do raise it. **Read the `max_retries:` default in `lib/paypal_server_sdk/configuration.rb`
  before you reason about resilience.** At `0` nothing is retried and passing `max_retries: 0` is a
  no-op; above `0` every listed status and method is retried underneath your code whether you wanted it
  or not. Raise it only where nothing above the SDK already retries — a background job, a Sidekiq
  worker, a failover wrapper, or your own orchestration loop. Retry layers multiply rather than add:
  `max_retries: 3` is **four** requests per attempt, so inside a 3-attempt job it is twelve requests
  against an API whose rate limit counts every one.

## Timeout and transport

`timeout` is the connection timeout, in **seconds**, handed to Faraday. Because retries happen inside
the SDK, a call that retries can take substantially longer than `timeout` — budget for
`timeout × (max_retries + 1)` plus the backoff intervals when you set a deadline of your own.

Two lower-level arguments let you take over the transport entirely:

- **`adapter:`** — the Faraday adapter symbol to perform requests with.
- **`connection:`** — a `Faraday::Connection` you built yourself, used for every request. This is the
  hook for custom middleware, instrumentation or a stubbed transport (see **ruby-testing**).

## Proxy

```ruby
client = PaypalServerSdk::Client.new(
  proxy_settings: PaypalServerSdk::ProxySettings.new(
    address: 'http://localhost',
    port: 8888,
    username: 'user',
    password: 'pass'
  )
)
```

Only `address` is required. `ProxySettings.from_env` reads `PROXY_ADDRESS`, `PROXY_PORT`,
`PROXY_USERNAME` and `PROXY_PASSWORD`, returning `nil` when no address is set — `Client.from_env` calls
it for you.

## Base URL / environment

There is **no free-form base-URL argument**. The URL is looked up in the `ENVIRONMENTS` hash in
`lib/paypal_server_sdk/configuration.rb`, keyed by the environment you selected and then by the server each
endpoint chooses. Server template parameters declared by the spec become their own `Configuration`
arguments and are substituted by `get_base_uri`.

Both keys are frozen string constants on the `Environment` and `Server` classes in that
same file. **Read the `Environment` class for the real constant names before naming one** — they come from
the spec's server list, and a production member may not exist.

Beyond that constant and any server template parameters, **there is no way to retarget the base URL.**
The client wires `get_base_uri` in as its base-URI executor and issues fully-resolved absolute URLs, so a
`connection:` you build with its own `url:` is ignored — pointing a Faraday connection at
`http://localhost:4010` will not send requests there. Use `connection:` to *intercept* requests instead
(a Faraday test-adapter stub — see **ruby-testing** — or a rewriting middleware), and `proxy_settings:` to
route through a proxy.

## Pagination
**No operation in this API is paginated**, so `lib/paypal_server_sdk/utilities/pagination/` is not generated
at all: there is no `PagedIterable`, no `PagedResponse` and nothing to call `.pages` or `.items` on. Every
list endpoint is a plain call returning the whole response the API sent.

Where such an endpoint takes paging parameters of its own (a `page`/`limit`/`cursor` the spec declares as
ordinary parameters), **you drive the loop**: pass the next value yourself and stop when a page comes back
with fewer items than you asked for, or when the API's own next-page field is empty. Nothing in the SDK
advances that state for you, and nothing bounds the loop — cap the iterations and the total items in your
own code.

## Logging

Logging is **built in**: this SDK was generated with logging enabled, so `Configuration#initialize` takes a
`logging_configuration:` argument and the classes live in `lib/paypal_server_sdk/logging/`. It is still
**off until you pass one** — the argument defaults to `nil`.

```ruby
client = PaypalServerSdk::Client.new(
  logging_configuration: PaypalServerSdk::LoggingConfiguration.new(
    log_level: Logger::INFO,
    mask_sensitive_headers: true,
    request_logging_config: PaypalServerSdk::RequestLoggingConfiguration.new(
      log_body: true,
      log_headers: true,
      headers_to_exclude: ['authorization'],
      include_query_in_path: true
    ),
    response_logging_config: PaypalServerSdk::ResponseLoggingConfiguration.new(
      log_body: true,
      log_headers: false
    )
  )
)
```

`log_body` and `log_headers` default to `false`, so a configuration you leave empty logs almost nothing.
Both request and response configurations also accept `headers_to_include` and `headers_to_unmask`, and
`LoggingConfiguration.from_env` reads `LOG_LEVEL`, `MASK_SENSITIVE_HEADERS` and the
`REQUEST_*`/`RESPONSE_*` variables.

Leave the logger argument unset to use the SDK's **built-in console logger**. To route logs elsewhere,
subclass `PaypalServerSdk::AbstractLogger` (`lib/paypal_server_sdk/logging/sdk_logger.rb`, documented in
`doc/abstract-logger.md`) and implement `log(level, message, params)` — `message` is a **template** and
`params` carries the values to interpolate. A stdlib `::Logger` does not satisfy that contract: its `log`
is an alias of `add(severity, message, progname)`, so the SDK's `params` hash is swallowed as `progname`
and the template is emitted with its placeholders unsubstituted, with no error to tell you.

`http_callback:` is still available alongside all of this, and is the seam to use when you want the
request/response objects themselves rather than log lines.

### Verify on the wire (first run of any new integration)

Log the first execution of any new call and inspect the output. A wrong environment, an unsubstituted
path parameter or a mis-serialized value produces no in-band signal — the only symptom is a runtime
`404`/`422`.

Checklist for the first logged request:

1. the **verb** matches the operation;
2. the **path** has no literal `{placeholder}` left unsubstituted;
3. each **path segment** is the value the API expects;
4. the parameters you set actually appear in the query string or body — an attribute the model omitted
   silently will simply be missing (see **ruby-models**).

Turn it back down once verified.
