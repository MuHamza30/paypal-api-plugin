---
name: 'ruby-authentication'
description: 'Configure authentication on an APIMatic-generated Ruby SDK client — every scheme **except custom authentication** is a generated immutable credentials class (`{Scheme}Credentials`) whose constructor takes keyword arguments and raises ArgumentError on a missing required one, passed to `Client.new` as its own named parameter; covers Basic, custom header, custom query API key, OAuth 2.0 bearer token and client-credentials, the credentials-class-free custom-auth stub, plus reading credentials from the environment. Use the moment you set credentials, an API key, a token, or OAuth on the PayPal Server SDK Ruby SDK, or need to know which schemes it accepts — load it even after reading the constructor in the source, since the parameter name alone doesn''t tell you it takes a built credentials object, that a custom-auth parameter has no class to build, that the object is immutable so attaching a fetched token means rebuilding the client, or that secrets belong in environment variables.'
---

# Authenticating an APIMatic Ruby SDK client

How you authenticate depends on the security scheme(s) the API uses. APIMatic surfaces **one named
parameter per scheme on `Client.new`**, and — for every scheme except custom authentication — **an
immutable credentials class to pass to it** (see `ruby-client-initialization`).

> **Custom authentication is the exception.** Its file holds only the scheme class (e.g.
> `class CustomAuth < CoreLibrary::HeaderAuth`), built from the whole configuration and shipped as a
> stub for the SDK's maintainer to complete. There is **no `{SchemeCredentials}` class to build**, even
> though the `{scheme}_credentials:` parameter still exists. Check the scheme's file for a credentials
> class before writing the call — a custom scheme means the SDK has to be completed or regenerated, not
> configured from your application.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g.
> `{scheme}_credentials`, `{SchemeCredentials}`, `{auth_key}`) — replace it with the concrete identifier
> from the source.

## Finding which schemes this SDK accepts

Three places, in order of directness:

1. `lib/paypal_server_sdk/configuration.rb` — the credential parameters on `Configuration#initialize`.
   **These are the source of truth**; the same parameters appear on `Client#initialize`.
2. `lib/paypal_server_sdk/http/auth/` — one file per scheme, each holding the scheme class and, for every
   scheme but custom authentication, its `{SchemeCredentials}` data class. The credentials class is a
   **sibling** of the scheme class in the same module, not nested inside it, so it is referenced as
   `PaypalServerSdk::{SchemeCredentials}`. A file with only a scheme class is a custom-auth stub — there
   is nothing to build for it.
3. `README.md` / `doc/auth/` — the generated authentication section, which names each scheme and lists
   its parameters and their types.

## The shape: build a credentials object, pass it by name

```ruby
require 'paypal_server_sdk'

client = PaypalServerSdk::Client.new(
  {scheme}_credentials: PaypalServerSdk::{SchemeCredentials}.new(
    {param}: ENV.fetch('MY_API_PARAM'),
    {other_param}: ENV.fetch('MY_OTHER_PARAM')
  )
)
```

Every generated credentials class has the same shape:

- **`initialize`** takes **keyword arguments**, one per auth parameter. A required parameter has no
  default and is checked immediately — `raise ArgumentError, '{param} cannot be nil' if {param}.nil?` —
  so a missing credential fails at construction, not on the first call. Optional parameters default to
  `nil` (or the value the spec declares).
- **`attr_reader`** for each parameter, and nothing else: the object is **immutable**.
- **`clone_with(...)`** returns a new instance with the arguments you pass replacing the current values.
  It fills each missing argument with `||=`, so it cannot clear a value back to `nil`/`false`.
- **`self.from_env`** builds the object from `UPPER_SNAKE` environment variables and returns `nil` when
  every required variable is unset. On an API with a **single** scheme the variable is named after the
  parameter (`{PARAM}`); with **multiple** schemes it is prefixed with the scheme (`{SCHEME}_{PARAM}`).
  Grep `from_env` in the scheme's file for the exact names.

The credentials you passed are readable back as `client.config.{scheme}_credentials`.

## Basic auth

```ruby
client = PaypalServerSdk::Client.new(
  {basic_auth_credentials}: PaypalServerSdk::{BasicAuthCredentials}.new(
    {username_param}: ENV.fetch('API_USERNAME'),
    {password_param}: ENV.fetch('API_PASSWORD')
  )
)
```

The scheme class Base64-encodes the two values into an `Authorization: Basic ...` header for you.

## API key — custom header or custom query parameter

An API key scheme is generated as either a header scheme or a query scheme; the header/parameter name is
fixed by the generated class, so all you supply is the value:

```ruby
client = PaypalServerSdk::Client.new(
  {api_key_credentials}: PaypalServerSdk::{ApiKeyCredentials}.new(
    {api_key_param}: ENV.fetch('API_KEY')
  )
)
```

Open the scheme's file under `lib/paypal_server_sdk/http/auth/` to see whether it subclasses the header or
the query auth base and which name it sends the key under.

## OAuth 2.0 — bearer token

When the API declares a bearer token you already hold, it is just another credentials parameter:

```ruby
client = PaypalServerSdk::Client.new(
  {o_auth_credentials}: PaypalServerSdk::{OAuthCredentials}.new(
    access_token: ENV.fetch('ACCESS_TOKEN')
  )
)
```

A bearer-token scheme's keyword is `access_token:`. Every other grant names its credentials
`o_auth_`-prefixed — `o_auth_client_id:`, `o_auth_client_secret:`, `o_auth_token:`, `o_auth_scopes:` — in
**every** build: the generator derives them from fixed constants, so there is no `oauth_client_id:`
spelling and `EnforceStandardizedCasing` does not change them. (The auth-manager reader on the client is
underscore-cased from the spec's own scheme name, not from that setting — a scheme the spec calls
`Oauth2` reads `client.oauth2`, one it calls `OAuthCCG` reads `client.o_auth_ccg`; grep `@auth_managers[`
in `client.rb` for the real reader.) The class name and the `Client.new` parameter *are* per-SDK, so
confirm those, and the parameter list, against the credentials class's `initialize` in
`lib/paypal_server_sdk/http/auth/` or the *Getter* column of `doc/auth/*.md` — a wrong keyword raises
`ArgumentError: unknown keyword`.

## OAuth 2.0 — client credentials

Constructing the client with the client id and secret is **enough**. The auth manager fetches a token
lazily on the first call that needs one, holds it, and re-fetches when it expires — nothing to attach:

```ruby
client = PaypalServerSdk::Client.new(
  {o_auth_credentials}: PaypalServerSdk::{OAuthCredentials}.new(
    o_auth_client_id: ENV.fetch('O_AUTH_CLIENT_ID'),
    o_auth_client_secret: ENV.fetch('O_AUTH_CLIENT_SECRET')
  )
)

result = client.{controller_name}.{operation}   # the token is fetched here, on demand
```

Confirm it for your SDK before relying on it: the grant's class under `lib/paypal_server_sdk/http/auth/`
has a `valid` method that assigns the token it fetched, and `doc/auth/` says so in prose. A **bearer-token
scheme has no token endpoint**, so there is nothing to auto-fetch — you supply the token yourself.

Fetch a token explicitly only to pre-warm one, inspect it, or reuse one you persisted. A token you fetch
this way is **not** written back into the credentials object, which is immutable — so attaching it means
cloning the credentials, cloning the configuration, and rebuilding the client:

```ruby
begin
  token = client.{auth_key}.fetch_token

  credentials = client.config.{o_auth_credentials}.clone_with(o_auth_token: token)
  config = client.config.clone_with({o_auth_credentials}: credentials)
  client = PaypalServerSdk::Client.new(config: config)
rescue PaypalServerSdk::APIException => e
  # handle the failed token request
end
```

`{auth_key}` is a reader the client exposes for each OAuth scheme. Grep `@auth_managers[` in
`lib/paypal_server_sdk/client.rb` — the reader you call is the **method whose body returns that entry**,
not the camelCase hash key it looks up. The manager also exposes `token_expired?(token)`, which takes the
`OAuthToken` you hold (from `fetch_token`, or from your own store) as a **required argument** — it does not
inspect the manager's stored token, and no public reader exposes that one — and, when the grant issues
refresh tokens, `refresh_token(additional_params: nil)`. A token from either is re-attached with the same
clone-and-rebuild sequence above.

A client-credentials scheme's credentials class additionally accepts:

| Parameter | Type | Purpose |
| --- | --- | --- |
| `o_auth_token_provider` | `proc { \|OAuthToken, OAuth2\| }` | callback the SDK invokes to obtain/refresh the token |
| `o_auth_on_token_update` | `proc { \|OAuthToken\| }` | callback fired when the token is updated |
| `o_auth_clock_skew` | `Integer` | seconds of skew allowed when checking token expiry |

To make a token survive a restart, **verify the update callback actually fires** in your SDK before
building persistence on it. The dependable route is to save the token `fetch_token` returned to you and
pass it back into the credentials object on the next boot.

## More schemes

For OAuth 2.0 **authorization code (3-legged)**, **resource-owner password**, **multiple or combined
schemes**, **custom authentication**, and the **deprecated flat auth parameters** some single-scheme SDKs
still accept, see [reference.md](reference.md).

## Notes

- A given SDK only exposes the credential parameters for the schemes its API uses; those names are
  generated per API, hence the `{...}` placeholders above.
- Credentials are set when the client is constructed. There is no setter — changing them means a new
  client (via `config.clone_with`).
- Keep secrets out of source. Load them from `ENV` or a secrets manager, or let `Client.from_env` do it —
  never hardcode them.
- Each scheme class defines `error_message`, the string reported when the scheme cannot be satisfied
  (e.g. a nil credential, or an expired OAuth token). If a call fails on authentication, that message
  names the parameter that is missing.
