---
name: 'typescript-authentication'
description: 'Configure authentication on an APIMatic-generated TypeScript/Node.js API client — each scheme is an optional credentials property on the config object, set as an object literal when constructing the client (or later via withConfiguration) — Basic (username/password), Bearer token, API key (header or query), custom schemes, and OAuth 2.0 (client-credentials, authorization-code, password) plus combined AND/OR schemes. Use the moment you set credentials, an API key, a token, or OAuth on the PayPal Server SDK TypeScript SDK, or need to know which schemes its config object exposes — load it even after reading the config interface in the source, since the property type doesn''t tell you that OAuth fields are `oAuth`-prefixed (capital A), that the property name need not match the scheme name in the spec, how to persist a token across restarts, or that only the first satisfied OR alternative is ever sent.'
---

# Authenticating an APIMatic TypeScript SDK client

**This API declares at least one security scheme.** APIMatic surfaces each one as an **optional credentials
property on the config object**; set the one(s) your API uses when constructing the client (see
`typescript-client-initialization`).

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{basicAuthProperty}`, `{Controller}`) — replace it with the concrete identifier from the source.

To see which schemes a specific SDK accepts, read the **credentials properties on the `Configuration`
interface in `src/configuration.ts`** — they are generated per-API and are the source of truth both for
which schemes exist and for the exact inner field names of each. The property name is generated too and
is **not** always the scheme name from the spec (a scheme named `APIKeyHeader` can surface as
`customHeaderAuthenticationCredentials`), so read it rather than deriving it. `doc/auth/*.md` gives the
same information as a table plus a working snippet, and names the property in its "Auth credentials can
be set using `...` object in the client" line.

A **custom** scheme is the exception to all of that: its property is typed `any`, it gets no `doc/auth`
page, and its real shape and behaviour live in the generated `customAuthenticationProvider` in
`src/authentication.ts` — the one part of that file that is not just a re-export of the shared `@apimatic`
auth adapters. See [reference.md](reference.md).

**Standing rule: pass credentials as the scheme's object literal.** Even a single-value scheme is
`{ <fieldName>: '...' }`. An SDK may additionally carry a deprecated top-level bare-string field (e.g.
`apikey?: string`) that the client folds into that object for you — it is legacy; set the object.

## Basic auth

```typescript
import { Client } from '@paypal/paypal-server-sdk';

const client = new Client({
  {basicAuthProperty}: {
    username: '...',
    password: '...',
  },
});
```

## Bearer token

```typescript
const client = new Client({
  {bearerAuthProperty}: { accessToken: 'ACCESS_TOKEN' },
});
```

## API key (header or query)

The key is sent as a header or a query parameter — placement and name are fixed by the generated
scheme, and its `doc/auth/` page names which (`custom-header-signature.md` vs
`custom-query-parameter.md`). Each **inner key is a literal parameter name**, so it often needs
quoting. A scheme declares **one key per parameter** and every one of them is required, so one key is
common but not the rule:

```typescript
const client = new Client({
  {apiKeyProperty}: { '{keyName}': 'API_KEY' },   // e.g. { 'X-Api-Key': '...' }
  // a two-parameter scheme needs both: { 'token': '...', 'api-key': '...' }
});
```

Read the real inner keys off the `Configuration` interface or the `## Auth Credentials` table in
`doc/auth/*.md` — one table row per required key, each the header or query parameter name, not a
fixed word like `apiKey`. Supplying a subset fails to compile — TypeScript names the missing required
key(s) (`TS2741` when exactly one is missing, `TS2739` when several are).

## OAuth 2.0 — client credentials

Every OAuth credential field is **`oAuth`-prefixed** — capital A:

```typescript
import { Client } from '@paypal/paypal-server-sdk';

const client = new Client({
  {oAuthProperty}: {
    oAuthClientId: '...',
    oAuthClientSecret: '...',
  },
});
```

**`oAuthScopes` is not on every grant.** The generator emits it only for a grant whose spec declares
scopes — typically the authorization-code grant, and not client-credentials. If the credentials object in
`src/configuration.ts` has no `oAuthScopes`, passing it is a TS excess-property error (TS2353) and there
is no generated scope enum to import. Where it does exist it is typed to that enum, not `string[]`, and
its members are listed under `### Scopes` in that grant's `doc/auth/*.md`.

The token is cached in memory on the client instance and refreshed lazily, immediately before a request,
when it is missing or past its `expiry`. There is **no 401-triggered invalidation** — a `401` propagates
as an `ApiError`. To refresh early, set `oAuthClockSkew` (seconds subtracted from expiry); it is unset by
default. If the token endpoint returns no `expires_in`, no `expiry` is recorded and the token is never
auto-refreshed.

## OAuth 2.0 — persisting and restoring a token

Every OAuth2 grant gets these lifecycle fields on its credentials object, alongside the grant's own:

```typescript
const client = new Client({
  {oAuthProperty}: {
    oAuthClientId: '...',
    oAuthClientSecret: '...',
    oAuthToken: previouslySavedToken,                      // seed/restore a known token
    oAuthOnTokenUpdate: (token) => saveTokenToDatabase(token),   // fires on every refresh
    oAuthTokenProvider: async (lastOAuthToken, authManager) =>   // supply the token yourself
      (await loadTokenFromDatabase()) ?? authManager.fetchToken(),
    oAuthClockSkew: 60,                                    // seconds; refresh this early
  },
});
```

`oAuthTokenProvider` is invoked whenever the cached token is missing or expired.

> ### ⚠ Your `oAuthTokenProvider` must never reject
>
> **A single rejection disables the client for the rest of the process.** The adapter keeps the token as
> a *promise* shared by every call on that client, and each call reads it before replacing it:
>
> ```ts
> let token = await lastOAuthToken;                 // ← throws here on every later call
> lastOAuthToken = refreshOAuthToken(token, ...);   // ← never reached
> ```
>
> When your provider rejects, that rejected promise is what gets stored. The next call throws on the
> **first** line and so never reaches the line that would replace it, and neither does the call after
> that. One transient failure — the database holding your token being briefly unreachable — is
> permanent for the lifetime of that client object, and no retry setting touches it because the failure
> happens before any request is built.
>
> So make the provider **resolve** in every path. Where you cannot produce a token, return an expired
> one rather than throwing: the adapter will treat it as needing refresh and call you again next time,
> which is the recoverable behaviour you want.
>
> ```ts
> oAuthTokenProvider: async (lastOAuthToken, authManager) => {
>   try {
>     return (await loadTokenFromDatabase()) ?? (await authManager.fetchToken());
>   } catch (err) {
>     recordTheFailureSomewhere(err);
>     return { ...(lastOAuthToken ?? {}), expiry: 0n } as typeof lastOAuthToken;  // expired ⇒ retried next call
>   }
> },
> ```
>
> The sample above this note takes the short form for readability. In anything long-lived, wrap it.

To re-credential an **already-constructed** client, use `client.withConfiguration({...})` — it returns a
**new** client rather than mutating the existing one:

```typescript
client = client.withConfiguration({
  {oAuthProperty}: { ...creds, oAuthToken: token },
});
```

## More schemes

For OAuth2 **authorization-code (3-legged)**, **resource-owner password**, **custom** schemes,
**multiple/combined** schemes (AND/OR), and **no-auth**, see [reference.md](reference.md).

## Notes

- A given SDK only exposes the credentials properties for the schemes its API uses; those names are generated per-API (hence the `{...Property}` placeholders above).
- Set credentials when constructing the client, or attach them later with
  `client.withConfiguration({...})`, which returns a new client.
- Keep secrets out of source — load them from environment variables (`process.env.MY_API_KEY`) or a secrets manager, never hardcode them.
