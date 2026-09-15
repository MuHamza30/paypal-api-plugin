# Authentication reference (APIMatic Ruby)

The auth schemes the APIMatic Ruby generator emits. Unlike some generators, an APIMatic Ruby SDK ships
**only the schemes its API actually uses** — `lib/paypal_server_sdk/http/auth/` contains one file per
scheme, and each file holds the scheme class plus, for every scheme but custom authentication, its
`{SchemeCredentials}` data class. Both the class names and the parameter names are generated per-API, so
every identifier below in `{...}` must be read from the source.

Every credentials class that exists has the same members: a keyword-argument `initialize` that raises
`ArgumentError` on a missing required parameter, an `attr_reader` per parameter, `clone_with(...)`, and
`self.from_env`. `self.from_env` reads `{PARAM}` on a single-scheme SDK and `{SCHEME}_{PARAM}` when the
API declares several.

## Basic

```ruby
PaypalServerSdk::Client.new(
  {basic_auth_credentials}: PaypalServerSdk::{BasicAuthCredentials}.new(
    {username_param}: '...',
    {password_param}: '...'
  )
)
```

Sends `Authorization: Basic base64(username:password)`.

## API key — custom header

```ruby
PaypalServerSdk::Client.new(
  {api_key_credentials}: PaypalServerSdk::{ApiKeyCredentials}.new({api_key_param}: '...')
)
```

The scheme class subclasses the core header-auth base and sends the value under the header name the spec
declared.

## API key — custom query parameter

Same shape, but the scheme class subclasses the core query-auth base and appends the value to the query
string instead of sending a header.

## OAuth 2.0 — bearer token

```ruby
PaypalServerSdk::Client.new(
  {o_auth_credentials}: PaypalServerSdk::{OAuthCredentials}.new(access_token: 'ACCESS_TOKEN')
)
```

A bearer-token scheme has no token endpoint, so there is no `fetch_token` — you supply the token.

> **The OAuth credential keywords are fixed; the class and the client parameter are not.** The bearer
> scheme's keyword is `access_token:`; every other grant names its parameters `o_auth_`-prefixed
> (`o_auth_client_id:`, `o_auth_client_secret:`, `o_auth_token:`, `o_auth_scopes:`,
> `o_auth_redirect_uri:`, …) in **every** build, because the generator derives them from fixed constants.
> There is no `oauth_client_id:` spelling, and `EnforceStandardizedCasing` does not touch them. The
> auth-manager reader on the client is underscore-cased from the spec's own scheme name (`Oauth2` →
> `client.oauth2`, `OAuthCCG` → `client.o_auth_ccg`), so grep `@auth_managers[` in `client.rb`. The
> `{OAuthCredentials}` class name and the `{o_auth_credentials}` parameter it is passed under **are**
> generated per-API — read those off `Configuration#initialize` and the scheme's file under
> `lib/paypal_server_sdk/http/auth/`, or the *Getter* column of `doc/auth/*.md`.

## OAuth 2.0 — client credentials

Constructing the client is sufficient: the manager fetches a token on the first call that needs one and
holds it.

```ruby
credentials = PaypalServerSdk::{OAuthCredentials}.new(
  o_auth_client_id: '...',
  o_auth_client_secret: '...',
  o_auth_scopes: ['...']              # only when the spec declares scopes
)

client = PaypalServerSdk::Client.new({o_auth_credentials}: credentials)
```

To pre-warm a token, or to reuse one you persisted, fetch it and rebuild the client with it — a token you
fetch yourself is not written back into the immutable credentials object:

```ruby
token = client.{auth_key}.fetch_token
client = PaypalServerSdk::Client.new(
  config: client.config.clone_with(
    {o_auth_credentials}: credentials.clone_with(o_auth_token: token)
  )
)
```

Extra optional parameters on this grant: `o_auth_token_provider`, `o_auth_on_token_update`,
`o_auth_clock_skew`.

## OAuth 2.0 — authorization code (3-legged)

The SDK cannot open a browser for you, so this grant is two calls with your redirect handling in between:

```ruby
client = PaypalServerSdk::Client.new(
  {o_auth_credentials}: PaypalServerSdk::{OAuthCredentials}.new(
    o_auth_client_id: '...',
    o_auth_client_secret: '...'
  )
)

# 1. Send the user here; your redirect endpoint receives ?code=...
auth_url = client.{auth_key}.get_authorization_url(state: 'csrf-token')

# 2. Exchange the code for a token, then rebuild the client with it.
token = client.{auth_key}.fetch_token(authorization_code)
client = PaypalServerSdk::Client.new(
  config: client.config.clone_with(
    {o_auth_credentials}: client.config.{o_auth_credentials}.clone_with(o_auth_token: token)
  )
)
```

`get_authorization_url(state: nil, additional_params: nil)` builds the URL from the spec's authorization
endpoint and the configured client id, redirect URI and scopes; `additional_params` is merged into the
query string. `fetch_token` takes the authorization code **positionally**, followed by an optional
`additional_params:`. This grant's credentials class also carries `o_auth_redirect_uri`. Read the
generated methods for the exact argument lists — an implicit-grant scheme has `get_authorization_url`
but no `fetch_token`.

## OAuth 2.0 — resource owner password

The user credentials are `o_auth_`-prefixed on this grant, not the bare `{username_param}` /
`{password_param}` a Basic scheme uses:

```ruby
PaypalServerSdk::{OAuthCredentials}.new(
  o_auth_client_id: '...',
  o_auth_client_secret: '...',
  o_auth_username: '...',
  o_auth_password: '...'
)
```

Then `fetch_token` as above.

## Token expiry and refresh (grants with a token endpoint)

- `client.{auth_key}.token_expired?(token)` reports whether the token **you pass it** has expired. It
  takes the `OAuthToken` as a required positional argument — it does not read the manager's own stored
  token, and no public reader exposes that one, so pass the token you got from `fetch_token` or from your
  store. `fetch_token` stamps an `expiry` onto the token from its `expires_in`, and a client-credentials
  scheme applies the clock-skew parameter when checking it.
- `client.{auth_key}.refresh_token(additional_params: nil)` exists when the grant issues refresh tokens.
- **A token you fetch yourself is not written back into the credentials object** — both it and the
  configuration are immutable, so attaching one ends with the same
  `credentials.clone_with(o_auth_token: token)` → `config.clone_with(...)` → `Client.new(config:)`
  sequence. The manager's own auto-fetched token, on a grant with a token endpoint, is attached for you.

## Multiple or combined schemes

When the API declares more than one scheme, each gets its own credentials class and its own named
parameter on `Client.new` — set every one the operations you call require:

```ruby
PaypalServerSdk::Client.new(
  {first_credentials}: PaypalServerSdk::{FirstCredentials}.new(...),
  {second_credentials}: PaypalServerSdk::{SecondCredentials}.new(...)
)
```

The client builds an `@auth_managers` hash keyed by scheme name and the generated per-operation code
applies whichever combination that operation declares — you do not compose them yourself. Note that with
multiple schemes each credentials class's `from_env` reads **scheme-prefixed** variables
(`{SCHEME}_{PARAM}`) rather than bare ones.

## Custom authentication — the scheme with no credentials class

A scheme the spec declares as *custom* generates **only** the scheme class (e.g.
`class CustomAuth < CoreLibrary::HeaderAuth`), whose `initialize` takes the whole configuration and whose
`error_message` and `valid` are TODO stubs for the SDK's maintainer to fill in. There is **no
`{SchemeCredentials}` class** to construct, even though the `{scheme}_credentials:` parameter still
exists on `Configuration#initialize` — referencing one raises `NameError`. The file's own contents are
the discriminator, not its name: a *custom header* scheme is an ordinary API-key scheme and does generate
its credentials class. A genuinely custom scheme has to be completed in the SDK or regenerated; it cannot
be configured from your application.

## No auth

An API with no security scheme generates no credentials classes and no credential parameters. An
operation that opts out of the API-wide scheme simply skips it; you still construct the client the same
way.

## Deprecated flat auth parameters (single-scheme SDKs)

Older single-scheme SDKs were generated so that `Client.new` also accepts each auth parameter directly
(`{username_param}:`, `{password_param}:`, …) alongside the credentials object. Where that code was
generated you will see a `create_auth_credentials_object` method in `configuration.rb`, and passing the
flat parameters prints a deprecation warning naming the credentials parameter to use instead. **Use the
credentials object.** The flat form exists only for backwards compatibility, and it is absent from SDKs
generated without it.

## Discovering what a specific SDK uses

1. List the credential parameters on `Configuration#initialize` in
   `lib/paypal_server_sdk/configuration.rb` — the **source of truth** for what this SDK accepts.
2. Open the matching file under `lib/paypal_server_sdk/http/auth/` for the credentials class, its required
   versus optional parameters, and the environment-variable names its `from_env` reads. A file holding
   only a scheme class is the custom-auth case above — there is nothing to build for it.
3. `doc/auth/` and the README's authentication section list the same parameters with their descriptions.
