---
name: 'php-authentication'
description: 'Configure authentication on an APIMatic-generated PHP SDK client — every scheme is a separate `{Scheme}CredentialsBuilder::init(…)` passed to a matching `->{scheme}Credentials(…)` setter on the client builder; covers Basic, bearer token, API key in a header or query parameter, OAuth 2 client-credentials, authorization-code and resource-owner-password, and hand-written custom auth. Use the moment you set credentials, an API key, a token, or OAuth on the PayPal Server SDK PHP SDK — load it even after reading the builder setters in the source, since the method names don''t tell you which grants fetch a token automatically, that reattaching a token means rebuilding the client, or that there is no read-from-environment factory.'
---

# Authenticating an APIMatic PHP SDK client
Which schemes exist is decided by the API spec. **The authoritative list is the set of
`->{scheme}Credentials(…)` setters on `src/{Client}Builder.php`** (mirrored on
`src/ConfigurationInterface.php` as one `get{Scheme}CredentialsBuilder()` plus one credentials getter per
scheme, and documented one-file-per-scheme under `doc/auth/`). Start there — and note that only the
*setter* name is uniform: the credentials getter is `get{Scheme}Credentials()`, **except** when the API has
exactly one scheme and it is an OAuth 2 grant, where the `Credentials` suffix is dropped and it is
`get{Scheme}()`. Grepping only for the suffixed getter on such an SDK finds nothing and looks like the
scheme is absent.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{Client}`,
> `{scheme}`) — replace it with the concrete identifier from the source. In a `use` statement this is
> not optional: `\{…}` after a namespace is PHP's group-use syntax, so an unsubstituted placeholder is a
> parse error rather than an undefined class.

## The shape: one credentials builder per scheme

```php
use PaypalServerSdkLib\{Client}Builder;
use PaypalServerSdkLib\Authentication\{Scheme}CredentialsBuilder;

$client = {Client}Builder::init()
    ->{scheme}Credentials(
        {Scheme}CredentialsBuilder::init(/* required params, positionally */)
            // optional params are fluent setters
    )
    ->build();
```

Every credentials builder follows the same rules:

- `init(...)` takes the scheme's **required** parameters, positionally.
- Optional parameters are fluent setters returning `$this`.
- The builder is passed to the client builder, which merges its config — you never call
  `getConfiguration()` yourself.

**Where `{scheme}` comes from depends on how many schemes the API has.** With more than one, each name is
the spec's own scheme name. With exactly one, the generator **discards** the spec's name and uses a fixed
name for the auth *type* (`BasicAuth`, `BearerAuth`, `ClientCredentialsAuth`,
`CustomHeaderAuthentication`, …) — that is the default, and only the `UseSecuritySchemeNameForSingleAuth`
generator setting keeps the spec's name. So `->basicAuthCredentials(…)`, `->apiKeyCredentials(…)` and
`->oAuthCCGCredentials(…)` are what one *multi-scheme* spec produced; yours may differ. Only the *pattern*
`->{schemeName}Credentials(…)` holds — read the setters off `src/{Client}Builder.php`.

### Where the classes live

Credentials **builders** and auth **managers** are always in `PaypalServerSdkLib\Authentication`. The
credentials *interface* moves: `PaypalServerSdkLib\Authentication` when the API has more than one scheme,
but the **root** `PaypalServerSdkLib` namespace when it has exactly one. Its **name** moves too — it is
`{Scheme}Credentials`, except on an API whose single scheme is an OAuth 2 grant, where the `Credentials`
suffix is dropped and the interface is just `{Scheme}` (e.g. `ClientCredentialsAuth`). Let your editor or
a grep of `src/` settle the `use` statement rather than assuming.

## Basic auth

```php
$client = {Client}Builder::init()
    ->{scheme}Credentials(
        {Scheme}CredentialsBuilder::init(
            getenv('API_USERNAME'),
            getenv('API_PASSWORD')
        )
    )
    ->build();
```

Sends `Authorization: Basic base64(username:password)`.

## Bearer token

```php
->{scheme}Credentials({Scheme}CredentialsBuilder::init(getenv('API_ACCESS_TOKEN')))
```

Sends `Authorization: Bearer <token>`.

## API key — header or query parameter

Placement (header vs query) and the wire name are fixed by the generated manager; you only supply the
value(s). An API key scheme may take **more than one** parameter — read `init(...)`'s signature.

```php
->{scheme}Credentials({Scheme}CredentialsBuilder::init(getenv('API_TOKEN'), getenv('API_KEY')))
```

## OAuth 2

All grants share `oAuthToken(?OAuthToken $token)` and `oAuthClockSkew(int $seconds)` on the credentials
builder, and `isTokenExpired(?OAuthToken $token = null)` on the manager.

> **The prefix is `oAuth`, with a capital A — not `oauth`.** Every generated setter is spelled
> `oAuthToken`, `oAuthScopes`, `oAuthClockSkew`, `oAuthTokenProvider`, `oAuthOnTokenUpdate`, and the
> token model is the class `OAuthToken`. PHP resolves *method* calls case-insensitively so a mis-cased
> setter happens to work, but a grep of `src/Authentication/` for the lowercase spelling finds nothing, a
> static analyser flags it, and a mis-cased **class** name in a `use` statement is a hard PSR-4 autoload
> failure. Copy the spelling from the source.

**What differs between grants is whether the SDK acquires the token for you:**

| Grant | `init(...)` takes | Token acquisition |
| --- | --- | --- |
| Client credentials | client id, client secret | **automatic** — fetched and refreshed on demand |
| Resource-owner password | client id, client secret, username, password | **automatic** |
| Authorization code | client id, client secret, redirect URI | **you drive it** — build the consent URL, then `fetchToken($code)` |
| Implicit | client id | **you drive it** — `buildAuthorizationUrl()` only; the manager has no `fetchToken` |

**`oAuthScopes([...])` is generated only where the spec declares scopes for that grant, and in practice
that is usually the authorization-code builder alone** — a client-credentials or resource-owner-password
builder frequently has no scopes setter at all, and calling one that was never generated is a fatal
`Call to undefined method`. Check the builder in `src/Authentication/` before adding the line. Scopes are
validated
against a generated scope class in `src/Models/`: a class of `public const` strings with a static
`checkValue()`. Its name is `OAuthScope` on a single-scheme API and `OAuthScope{Scheme}` when the API has
more than one scheme, where `{Scheme}` is that scheme's name spelled exactly as `src/Authentication/`
spells it — it comes from the spec, so its casing is whatever the spec gave it (e.g.
`OAuthScopeOAuthACG`), plus any enum postfix the build adds. Don't guess — follow the `use`
statement at the top of the credentials builder, or grep `src/Models/` for `checkValue`. Pass its
constants, not raw strings. If an operation's `doc/controllers/*.md` section has a *Requires scope*
block, omitting the scope is a 401/403 at call time, not a build error.

### Client credentials (and resource-owner password) — automatic

```php
$client = {Client}Builder::init()
    ->{scheme}Credentials(
        {Scheme}CredentialsBuilder::init(getenv('OAUTH_CLIENT_ID'), getenv('OAUTH_CLIENT_SECRET'))
            ->oAuthScopes([{ScopeClass}::{CONSTANT}])   // only if this builder has the setter
            ->oAuthOnTokenUpdate(function (OAuthToken $token): void {
                // persist $token so a restart doesn't re-authorize
            })
            ->oAuthTokenProvider(function (?OAuthToken $last, $manager): OAuthToken {
                // supply a stored token, or return $manager->fetchToken()
            })
    )
    ->build();
```

The manager fetches a token before the first call and re-fetches when the cached one is expired. Both
callbacks are optional; `oAuthOnTokenUpdate` is how you persist a token, `oAuthTokenProvider` how you
supply one you already have. **Neither is available on the authorization-code or implicit grants.**

### Authorization code — you drive the flow

```php
$authUrl = $client->get{Scheme}()->buildAuthorizationUrl();   // send the user here
// …after the redirect comes back with ?code=…
$token = $client->get{Scheme}()->fetchToken($_GET['code']);
```

Until a token is present, calls fail with `\InvalidArgumentException` ("Client is not authorized. An
OAuth token is needed to make API calls.") — **not** `ApiException`. Refresh with
`$client->get{Scheme}()->refreshToken()`.

Note the accessor name: for an API with a **single** OAuth scheme it is `get{Scheme}()`, without the
`Credentials` suffix; with multiple schemes it is `get{Scheme}Credentials()`. Read
`src/ConfigurationInterface.php` rather than guessing.

### Reattaching a stored token

A built client is immutable, so a token you loaded from storage is applied by **rebuilding**:

```php
$client = $client
    ->toBuilder()
    ->{scheme}Credentials($client->get{Scheme}CredentialsBuilder()->oAuthToken($token))
    ->build();
```

`get{Scheme}CredentialsBuilder()` returns `null` when the scheme's required parameters were never set —
guard it if the credentials are optional in your configuration.

## Custom auth

If the spec declares a custom scheme, the generator emits a `{Scheme}Manager` (so
`CustomAuthenticationManager` under the single-scheme fallback name, not `CustomAuthManager`) whose
`apply()` body is a `// TODO: Add your custom authentication here` stub — the signing is hand-written into
the SDK, never configured from your application. **A credentials builder and a `->{scheme}Credentials(…)`
setter are still generated when the scheme declares parameters**, and their values reach `apply()` through
the manager's `get{Param}()` getters; a parameterless custom scheme gets neither and there is genuinely
nothing to set. Read `src/{Client}Builder.php` before you either invent a setter or declare there is none.

## More schemes and full signatures

For the complete per-scheme matrix — every `init(...)` signature, the manager methods each grant
exposes, combined schemes, and how to discover the names in a specific SDK — see [reference.md](reference.md).

## Notes

- **There is no `fromEnvironment` factory.** Nothing in `src/` reads environment variables. Read them
  yourself with `getenv()` / `$_ENV` (or a `.env` loader) and pass the values in — never hardcode a
  secret, and never commit one.
- A generated SDK may also carry **deprecated flat setters** for individual auth parameters (e.g.
  `->oAuthClientId('…')`) when the API has exactly one scheme. Their docblocks point at the credentials
  setter; use the credentials builder instead.
- Setting credentials is a **construction-time** decision: there is no setter on a built client. See
  **php-client-initialization** for `toBuilder()` / `withConfiguration()`.
