---
name: 'java-authentication'
description: 'Configure authentication on an APIMatic-generated Java SDK client — every scheme is a `{Scheme}Model` you build with its own nested `Builder` (required credentials in the Builder''s constructor, optional ones as fluent setters) and hand to a setter on the client Builder: `.{scheme}Credentials(model)`, or the suffix-less `.clientCredentialsAuth(model)` / `.authorizationCodeAuth(model)` when the API''s only scheme is an OAuth 2 grant; covers Basic, custom header, custom query (API key), bearer/access token, and the OAuth 2.0 grants. This API declares at least one scheme, so all of it applies. Use the moment you set credentials, an API key, a token, or OAuth on the PayPal Server SDK Java SDK — load it even after reading the Builder in the source, since the setter name alone won''t tell you it takes a built model object, that an unset credential defaults to empty strings and then fails the first call inside the SDK with an unchecked `AuthValidationException` instead of reaching the API for a `401`, or that only the client-credentials grant fetches a token for you.'
---

# Authenticating an APIMatic Java SDK client
> **This API declares at least one security scheme**, so everything below applies to this SDK:
> `<root>/authentication/` exists and `{Api}Client.Builder` carries a credential setter per scheme. Read
> those setters for the real names.

How you authenticate depends on the security scheme(s) the API uses. APIMatic surfaces each scheme as a
**`{Scheme}Model` data class plus a matching setter on the client `Builder`**. Set the ones your API uses
when you build the client (see `java-client-initialization`).

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{Api}Client`,
> `{Scheme}Model`) — replace it with the concrete identifier from the source.

To see which schemes a specific SDK accepts, read the **credential setters on `{Api}Client.Builder`** —
those are the source of truth. The models live in `<root>/authentication/`; the credentials *interface*
sits in the root package when the API has a single scheme and in `<root>/authentication/` when it has
several. `doc/auth/*.md` lists the same set in readable form.

## The universal shape

Every scheme follows the same three steps: build the model, pass it to the client builder, forget about
it.

```java
import {rootPackage}.{Api}Client;
import {rootPackage}.authentication.{Scheme}Model;

{Api}Client client = new {Api}Client.Builder()
        .{scheme}Credentials(new {Scheme}Model.Builder(
                        System.getenv("API_CLIENT_ID"),      // required credentials: Builder constructor
                        System.getenv("API_CLIENT_SECRET"))
                .build())
        .build();
```

- **Required credentials are constructor arguments** on the model's `Builder`, and each throws
  `NullPointerException` on `null` — so a missing environment variable fails at construction, not on the
  first call.
- **Optional credentials are fluent setters** on the same `Builder`.
- The model's own constructor is private; `build()` is the only way to make one. `model.toBuilder()`
  gives you a pre-populated `Builder` to derive a variant from.

## Basic auth

```java
.basicAuthCredentials(new BasicAuthModel.Builder(
                System.getenv("API_USERNAME"),
                System.getenv("API_PASSWORD"))
        .build())
```

## API key in a header

```java
.customHeaderAuthenticationCredentials(new CustomHeaderAuthenticationModel.Builder(
                System.getenv("API_KEY"))
        .build())
```

The header name is fixed by the generated scheme — you supply only the value. A scheme with several
parameters takes several constructor arguments, in the order the model's `Builder` declares them.

## API key in a query parameter

```java
.customQueryAuthenticationCredentials(new CustomQueryAuthenticationModel.Builder(
                System.getenv("API_KEY"))
        .build())
```

## Bearer / access token

```java
.bearerAuthCredentials(new BearerAuthModel.Builder(
                System.getenv("API_ACCESS_TOKEN"))
        .build())
```

## OAuth 2.0 — client credentials

This is the only grant the SDK drives end to end: it fetches a token on the first call that needs one,
caches it, and refetches when it expires.

```java
.clientCredentialsAuth(new ClientCredentialsAuthModel.Builder(
                System.getenv("API_OAUTH_CLIENT_ID"),
                System.getenv("API_OAUTH_CLIENT_SECRET"))
        .build())
```

The other grants (authorization code, resource-owner password) do **not** fetch anything on their own —
you drive the flow and rebuild the client with the token. See [reference.md](reference.md).

## Reading credentials back off the client

Each scheme produces two getters on the client:

- `get{Scheme}Credentials()` — the credentials *interface*, backed by the auth manager. This is where the
  OAuth methods (`fetchToken`, `isTokenExpired`, …) live.
- `get{Scheme}Model()` — the model you supplied, so you can `toBuilder()` a variant off it.

For OAuth grants in a single-scheme SDK the interface drops the `Credentials` suffix, so the getter is
`getClientCredentialsAuth()` / `getAuthorizationCodeAuth()` rather than `get...Credentials()`. Confirm
the exact names in `{Api}Client.java`.

## More schemes

For OAuth 2 **authorization code**, **resource-owner password**, token persistence and refresh
callbacks, and **multiple/combined** schemes (AND/OR), see [reference.md](reference.md).

## Notes

- **A forgotten credential setter fails inside the SDK — you never see the API's `401`.** Every
  credential field on the client `Builder` is initialized to a model built with **empty strings** for its
  required values. For a header/query/basic/bearer scheme an empty value fails auth validation *before*
  the request is built and nothing leaves the process. An OAuth 2 grant validates on the **token**
  instead: authorization-code and resource-owner-password fail locally the same way, but
  **client-credentials first attempts a token fetch** — a real POST to the token endpoint with the blank
  id/secret — and swallows its `ApiException`/`IOException` inside `getTokenFromProvider()`, leaving the
  token `null`. Either way the call then throws the unchecked `AuthValidationException`
  (`io.apimatic.core.exceptions`); it extends `RuntimeException`, so the
  `catch (ApiException | IOException)` ladder from **java-error-handling** does not cover it. Set every
  scheme the endpoints you call require.
- Set credentials when you build the client. To rotate them later, derive a new client with
  `client.newBuilder().{scheme}Credentials(...).build()` — the built client is immutable.
- Keep secrets out of source — read them from the environment (`System.getenv(...)`) or a secrets
  manager, never hardcode them. The samples above deliberately show that form.
