---
name: 'ruby-calling-endpoints'
description: 'Call API operations on an APIMatic-generated Ruby SDK — operations are methods on a controller reached through a memoized reader on the client (`client.{controller}.{operation}`), never a class you instantiate; the parameter form is a generation-time setting (positional required plus keyword optionals, all-keyword, or `CollapseParamsToArray`''s single string-keyed `options = {}` hash), and every non-paginated operation returns an `ApiResponse` wrapper whose payload is on `.data`. Also covers building request models, enum constants, the `_query_parameters`/`_field_parameters` escape hatches, and binary and empty responses. Use whenever invoking an endpoint, building a request body, or consuming a response from the PayPal Server SDK Ruby SDK — load it even after reading the method signature in the source, since the signature won''t warn you that a collapsed `options = {}` hash is read with string keys (so keyword arguments silently send nothing), or that the payload is on `.data` rather than on the returned object itself.'
---

# Calling endpoints on an APIMatic Ruby SDK

Operations are methods on a **controller you reach through a reader on the client**. There is no
controller class to instantiate:

```ruby
require 'paypal_server_sdk'

client = PaypalServerSdk::Client.new
result = client.{controller_name}.{operation}
```

Each reader (`client.{controller_name}`) builds its controller once and memoizes it, so calling it
repeatedly is free. Controllers live in a folder under `lib/paypal_server_sdk/` — one file per API group.
The folder name (`controllers/`, `apis/`, …) follows the generator's controller-namespace setting and the
base class they extend (`BaseController`, `BaseApi`, …) its `ControllerPostfix` setting, so read the
`# Controllers` require block at the bottom of `lib/paypal_server_sdk.rb` for the real folder instead of
assuming one. **Read the reader names off `lib/paypal_server_sdk/client.rb`** (or the controller/API table
in `doc/client.md`); operation method names are snake_cased from the spec and follow no fixed
verb/resource pattern, so take the real name from the source too.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g.
> `{controller_name}`, `{operation}`, `{Model}`, `{EnumType}`) — replace it with the concrete identifier
> from the source.

## Method signature convention

**Which of three parameter forms an operation uses is decided when the SDK is generated, so read its
`def` line before writing the call.** The unset form is positional required parameters with keyword
optionals:

```ruby
def {operation}({required_param},
                {optional_param}: nil,
                _query_parameters: nil)
```

- **Required parameters are positional here**, in the order the spec declares them.
- **Optional parameters are keyword arguments**, so you pass only the ones you need and never a
  positional placeholder. They usually default to `nil`, but a parameter the spec gives a default gets
  that value — and it is **sent on the wire**, so omitting the argument does not mean the parameter is
  omitted from the request. Read the `def` line.
- **A generator setting can turn every parameter into a keyword argument** — some SDKs are built that
  way, so a required parameter reads `{required_param}:` instead. `CollapseParamsToArray` (or a
  per-endpoint collect-parameters flag) instead collects every parameter of an operation that has more
  than one into a single hash, `def {operation}(options = {})`, leaving no positional required parameters
  and no per-parameter keyword optionals — only the `_query_parameters:` / `_field_parameters:` escape
  hatches below can still trail the hash. **That hash is keyed by strings**, not symbols and not keyword
  arguments — the generated body reads each value back as `options['{param}']`, and a request body is
  `options['body']`. Writing `{param}: value` parses fine, builds a symbol-keyed hash, and every lookup
  returns `nil`: the request goes out with its template parameter unsubstituted and no query values,
  with **no Ruby-level error**. Write it as `{operation}('{param}' => value, ...)`, taking the names from
  the `@param` comments above the `def`. A single-parameter operation stays positional even in a
  collapsed build. **Read the `def` line** in the controller file (or the signature block in
  `doc/controllers/{controller}.md`) before writing the call — do not infer it from another operation.
- **A nullable parameter is a keyword argument even when it is required**, because `nil` has to be
  expressible.
- **`_query_parameters:` / `_field_parameters:`** appear on operations that accept extra, unmodelled
  query or form values. Each takes a `Hash` that is merged into the request — an escape hatch for
  parameters the spec doesn't name, not something you normally set.
- Methods **raise** on error responses — see **ruby-error-handling**.

## Reading the response



This SDK was generated with `ReturnCompleteHttpResponse`, so every **non-paginated** operation returns an
`ApiResponse` and the deserialized payload moves to `.data`. That applies to the whole SDK at once —
there is no per-operation exception:

```ruby
response = client.{controller_name}.{operation}

if response.success?
  puts response.data
elsif response.error?
  warn response.errors
end
```

`ApiResponse` carries `status_code`, `reason_phrase`, `headers`, `raw_body`, `request` and `data`. The
payload is on **`.data`** — not `.body` and not `.result` — so a sample written against a bare-value SDK
(`pets.each { ... }` straight off the call) iterates the wrapper and fails. You can confirm the build in
one glance: `lib/paypal_server_sdk/http/api_response.rb` exists only when the setting is on.

**Binary responses have no model.** An operation whose response handler declares neither a
`deserialize_into` nor `is_response_void(true)` hands you the raw body — `doc/controllers/{controller}.md`
types it `Binary`. Read the bytes off `response.data` (or `response.raw_body`) and write them out
yourself. A handler marked `is_response_void(true)` is the empty-response case instead and yields `nil`
data, so check for that qualifier before concluding a response is binary.



## Building request models

A request body parameter is a model instance from `lib/paypal_server_sdk/models/`. Construct it and pass
it like any other parameter:

```ruby
# keyword form (EnableModelKeywordArgsInRuby)
body = PaypalServerSdk::{Model}.new({required_attr}: {required_value}, {optional_attr}: {optional_value})

# positional form
body = PaypalServerSdk::{Model}.new({required_value}, {optional_value})

result = client.{controller_name}.{operation}(body)

# collapsed signature (def {operation}(options = {})) — the body travels under the string key 'body'
result = client.{controller_name}.{operation}('body' => body)
```

**Open the model's `initialize`** and use the form it declares — `EnableModelKeywordArgsInRuby` decides
which one the generator emitted, and calling the wrong one raises `ArgumentError: wrong number of
arguments`. The SDK's own `doc/models/` examples use whichever form this build has. Attributes you leave
out are omitted from the serialized JSON entirely. See **ruby-models** for the details, and for anything that isn't a plain
string or number (enums, oneOf/anyOf unions, collections, dates, file uploads).

## Enums

An enum is a class of constants, not a Ruby symbol — frozen strings, or plain `Integer`s when the spec
declares an integer-based enum. Pass the constant; passing a raw value directly is safe only for a string
enum, since an integer enum's constant holds a number and a string would go on the wire as one:

```ruby
client.{controller_name}.{operation}(PaypalServerSdk::{EnumType}::{SOME_CONSTANT})
client.{controller_name}.{operation}('server_provided_value')   # string enums only
```

The constants live in `lib/paypal_server_sdk/models/{enum_type}.rb`. See **ruby-models**.

## Worked example — a list call with optional filters

```ruby
# Signature (illustrative — read the real one):
#   def {operation}({status},
#                   page: nil,
#                   per_page: nil)

result = client.{controller_name}.{operation}(
  PaypalServerSdk::{EnumType}::{SOME_CONSTANT},
  page: 1,
  per_page: 100
)

result.data.each do |item|
  puts item.{attribute}
end
```

The same call against a collapsed signature — `def {operation}(options = {})` — is one string-keyed hash,
required and optional parameters together:

```ruby
result = client.{controller_name}.{operation}(
  '{status}' => PaypalServerSdk::{EnumType}::{SOME_CONSTANT},
  'page' => 1,
  'per_page' => 100
)
```

## Finding the right method in the SDK source

- `doc/controllers/` — one markdown page per controller: the reader used to obtain it, the class name,
  every operation's `def` line, its response type, a copy-pasteable example, and an **Errors table**
  mapping status codes to exception classes. **Grep here first.**
- the controller files under `lib/paypal_server_sdk/` (folder named per the `# Controllers` require block
  in `lib/paypal_server_sdk.rb`) — the authoritative signatures, plus the request builder that shows the
  HTTP method, the route and how each parameter is sent (path, query, header, form, body).
- `lib/paypal_server_sdk/models/` — request/response models and enums.
- `lib/paypal_server_sdk/exceptions/` — the exception classes named in the Errors tables.

## Next

- Errors and status codes → **ruby-error-handling**
- Pagination, retries, timeouts → **ruby-configuration-resilience**
- Unions, collections, dates, enums → **ruby-models**
