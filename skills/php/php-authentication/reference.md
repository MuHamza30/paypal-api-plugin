# Authentication reference (APIMatic PHP)

The full matrix of auth schemes the APIMatic PHP generator supports. Class **shapes** are identical
across SDKs; only the **names** are generated per-API, from the spec's scheme names or from the auth type
(see below).

## Naming rules

| Thing | Pattern | Namespace |
| --- | --- | --- |
| Client-builder setter | `->{scheme}Credentials({Scheme}CredentialsBuilder)` | — |
| Credentials builder | `{Scheme}CredentialsBuilder` | `PaypalServerSdkLib\Authentication` |
| Auth manager | `{Scheme}Manager` | `PaypalServerSdkLib\Authentication` |
| Credentials interface | `{Scheme}Credentials` — but plain `{Scheme}`, with the `Credentials` suffix dropped, when the API has exactly one scheme and it is an OAuth 2 grant (see the asymmetry below) | `PaypalServerSdkLib\Authentication` (multi-scheme API) or `PaypalServerSdkLib` (single-scheme API) |
| Client accessors | `get{Scheme}Credentials()` — `get{Scheme}()` under the same asymmetry, since the accessor is named after the interface — and `get{Scheme}CredentialsBuilder(): ?…` | on `ConfigurationInterface` |
| Per-scheme docs | `doc/auth/{slug}.md` | — |

`{Scheme}` is the spec's scheme name only when the API has **more than one** scheme. With exactly one the
generator discards the spec's name and uses a fixed name for the auth type — `BasicAuth`, `BearerAuth`,
`ClientCredentialsAuth`, `AuthorizationCodeAuth`, `ImplicitAuth`, `ResourceOwnerAuth`,
`ResourcePasswordAuth`, `CustomHeaderAuthentication`, `CustomQueryAuthentication`, `CustomAuthentication`.
That is the default; the `UseSecuritySchemeNameForSingleAuth` generator setting keeps the spec's name
instead. Grep the client builder to see which your SDK got.

**Single-scheme OAuth asymmetry:** when the API has exactly one scheme and it is an OAuth 2 grant, the
credentials *interface* drops the `Credentials` suffix, so the accessor reads `get{Scheme}()` rather than
`get{Scheme}Credentials()`. The builder accessor keeps its full name
(`get{Scheme}CredentialsBuilder()`). Read `src/ConfigurationInterface.php`.

## Every credentials builder

```php
{Scheme}CredentialsBuilder::init(/* required params, positionally */)
    ->{optionalParam}($value)   // one fluent setter per optional param
```

Constructor is private; `init(...)` is the only entry point. The client builder consumes it.

## Basic

```php
{Scheme}CredentialsBuilder::init(string $username, string $password)
```
Manager applies `Authorization: Basic base64(username:password)` as a required, non-empty header.

## Bearer token (OAuth 2 bearer)

```php
{Scheme}CredentialsBuilder::init(string $accessToken)
```
Manager applies `Authorization: Bearer <accessToken>`.

## API key — custom header

```php
{Scheme}CredentialsBuilder::init(string $param1 /*, string $param2, … */)
```
Each declared parameter becomes a required, non-empty **header** whose wire name comes from the spec.

## API key — custom query parameter

Same builder shape; each parameter becomes a required, non-empty **query parameter** instead.

## OAuth 2 — client credentials

```php
{Scheme}CredentialsBuilder::init(string $oAuthClientId, string $oAuthClientSecret)
    ->oAuthToken(?OAuthToken $token)
    ->oAuthScopes(?array $scopes)                 // only when the spec declares scopes for this grant
    ->oAuthClockSkew(int $seconds)
    ->oAuthTokenProvider(callable $provider)      // fn(?OAuthToken $last, {Scheme}Manager $m): OAuthToken
    ->oAuthOnTokenUpdate(callable $onUpdate)      // fn(OAuthToken $token): void
```

Manager: `fetchToken(?array $additionalParams = null): OAuthToken`,
`isTokenExpired(?OAuthToken $token = null): bool`.

Token handling is **automatic**: before a call the manager reuses a cached, unexpired token; otherwise it
calls your `oAuthTokenProvider` if you set one, else `fetchToken()`. Whatever it ends up with is passed
to `oAuthOnTokenUpdate` if you set that. If `fetchToken()` throws, the previously held token is kept —
which means a bad client id/secret surfaces later as `\InvalidArgumentException: Client is not
authorized…` from `validate()`, not as a fetch error.

## OAuth 2 — resource-owner password credentials

```php
{Scheme}CredentialsBuilder::init(
    string $oAuthClientId,
    string $oAuthClientSecret,
    string $oAuthUsername,
    string $oAuthPassword
)
    ->oAuthToken(?OAuthToken)
    ->oAuthScopes(?array $scopes)                 // only when the spec declares scopes for this grant
    ->oAuthClockSkew(int)
    ->oAuthTokenProvider(callable)
    ->oAuthOnTokenUpdate(callable)
```

Manager: `fetchToken(?array)`, `refreshToken(?array)`, `isTokenExpired(?OAuthToken)`. Same automatic
acquisition as client credentials.

## OAuth 2 — authorization code

```php
{Scheme}CredentialsBuilder::init(
    string $oAuthClientId,
    string $oAuthClientSecret,
    string $oAuthRedirectUri
)
    ->oAuthToken(?OAuthToken)
    ->oAuthScopes(?array $scopes)                 // only when the spec declares scopes for this grant
    ->oAuthClockSkew(int)
```

Manager: `buildAuthorizationUrl(?string $state = null, ?array $additionalParams = null): string`,
`fetchToken(string $authorizationCode, ?array $additionalParams = null): OAuthToken`,
`refreshToken(?array $additionalParams = null): OAuthToken`, `isTokenExpired(?OAuthToken $token = null)`.

**No `oAuthTokenProvider` / `oAuthOnTokenUpdate`** — those setters are not generated for this grant.
Nothing is automatic: without a token, a call throws `\InvalidArgumentException` rather than making a
request.

Typical flow:

```php
$manager = $client->get{Scheme}();                 // or get{Scheme}Credentials() on a multi-scheme API

// 1. send the user to the provider
$url = $manager->buildAuthorizationUrl();

// 2. on your redirect endpoint
$token = $manager->fetchToken($_GET['code']);

// 3. persist $token, then rebuild the client with it
$client = $client->toBuilder()
    ->{scheme}Credentials($client->get{Scheme}CredentialsBuilder()->oAuthToken($token))
    ->build();
```

## OAuth 2 — implicit grant

Builder takes the client id plus `->oAuthToken(?OAuthToken)` and `->oAuthClockSkew(int)`. The manager
exposes `buildAuthorizationUrl(...)` and `isTokenExpired(...)` but **no `fetchToken` and no
`refreshToken`** — by design, the token arrives in the redirect fragment and you set it yourself.

## Custom auth

The manager follows the same `{Scheme}Manager` rule as every other scheme, so a single-scheme API with
default naming gets `CustomAuthenticationManager`, not `CustomAuthManager`. Its `apply()` is always a
stub — the signing itself is hand-written into the SDK:

```php
public function apply(RequestSetterInterface $request): void
{
    // TODO: Add your custom authentication here
}
```

**Whether your application configures anything depends on the parameters the scheme declares.** With
none, the manager is constructed with the client alone and there is no credentials interface, no
credentials builder and no client-builder setter. With one or more, all three are generated exactly as
for any other scheme — `{Scheme}CredentialsBuilder::init(…)` into `->{scheme}Credentials(…)` — the
manager's constructor becomes `(array $config, ConfigurationInterface $client)`, and your values reach
`apply()` through the `get{Param}()` getters the stub's own comment lists. `ls src/Authentication/` and
grep `src/{Client}Builder.php` for the setter to see which you have.

## The token model

`OAuthToken` is generated into `PaypalServerSdkLib\Models` with a matching
`Models\Builders\OAuthTokenBuilder`. Accessors: `getAccessToken()`, `getTokenType()`, `getExpiresIn()`,
`getScope()`, `getRefreshToken()`, `getExpiry()` / `setExpiry()`. Note the casing — `OAuth`, not
`Oauth`. The exact class name is subject to the same model post-fixing as every other model, and **class
names are case-sensitive to Composer's PSR-4 autoloader even though PHP itself is not**: a `use` of
`OAuthToken` looks for `src/Models/OAuthToken.php` and dies with "Class not found" on any case-sensitive
filesystem (Linux CI, Docker). Copy the exact spelling from `src/Models/`.

## Scopes

Scopes are a generated class in `src/Models/` (a class of `public const` strings plus a static
`checkValue()`), passed as `->oAuthScopes([{ScopeClass}::{CONSTANT}, …])` — **but only on the grants
whose builder actually has the setter.** The generator emits it per grant from the spec's declared
scopes, and commonly only the authorization-code builder gets one; `grep -l oAuthScopes
src/Authentication/` answers it for your SDK in one line. The class is named `OAuthScope` on a
single-scheme API and `OAuthScope{Scheme}` when the API has
more than one; take the name from the credentials builder's own `use` statement. Constant names are
derived from the scope strings and are frequently mangled, so read the class.

## Combined / multiple schemes

An API can require more than one scheme. Set every credentials builder the operation needs on the client
builder; the generated client composes them. `doc/auth/` has one file per scheme, and the per-operation
requirement is in `doc/controllers/*.md`.

## Discovering what a specific SDK uses

1. `src/{Client}Builder.php` — the `->…Credentials(…)` setters are the definitive list.
2. `src/ConfigurationInterface.php` — the matching credentials accessor and `get…CredentialsBuilder()` for
   each scheme. The credentials accessor is `get{Scheme}Credentials()`, or `get{Scheme}()` under the
   single-scheme OAuth asymmetry above — read the file rather than grepping only for the suffixed name.
3. `src/Authentication/` — the builders and managers; `init(...)`'s signature is the required-parameter list.
4. `doc/auth/*.md` — a worked snippet per scheme, generated from the same data.
