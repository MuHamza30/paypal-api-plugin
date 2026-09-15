---
name: 'python-authentication'
description: 'Set credentials on the PayPal Server SDK Python SDK. Load before configuring any auth scheme — Basic, API key in header or query, bearer token, or an OAuth 2 grant. The kwarg name won''t tell you it takes a constructed `{Scheme}Credentials` object, that missing required values raise `ValueError` at construction, or how a fetched OAuth token is re-attached.'
---

# Authenticating an APIMatic Python SDK client

How you authenticate depends on the security scheme(s) the API uses. APIMatic generates **one keyword
argument per scheme** on the client (and on `Configuration`), and — for every scheme except custom
authentication — **a credentials class to pass to it**, under `paypalserversdk/http/auth/`. You build
the object and pass it in when constructing the client (see `python-client-initialization`).

> **Custom authentication is the exception.** Its module holds only the handler class (e.g.
> `class CustomAuth(HeaderAuth)`), constructed from the whole `Configuration` and shipped as a stub with
> an empty `auth_params` and `# TODO` comments where the credential values belong. There is **no
> `{Scheme}Credentials` class to build**, even though the `{scheme}_credentials` keyword argument still
> exists on the client. Nothing here is configurable from your application: custom authentication means
> *the SDK itself* has to be completed or regenerated. **A custom *header* or custom *query-parameter*
> scheme is not this case** — those are ordinary API-key schemes with a credentials class and a
> `doc/auth/` page. Check the scheme's module for a credentials class before writing the call.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g.
> `{basic_auth_credentials}`, `{controller}`) — replace it with the concrete identifier from the source.

> **Every identifier in this skill is a placeholder — none of these snippets will import as written.**
> Auth module names, class names, constructor arguments and client kwargs are all generated from the
> scheme as *your* spec declares it, so they differ between SDKs: one has `api_key.py`/
> `ApiKeyCredentials`, another `api_key_header.py`/`ApiKeyHeaderCredentials`. **OAuth has one rule, and
> it is about how many schemes the API declares, not about casing:** when OAuth 2 is the *only* scheme,
> the generator overrides the module to `o_auth_2.py` and the handler class to `OAuth2` whatever the
> spec named the scheme, and names the credentials class for the grant (e.g.
> `ClientCredentialsAuthCredentials`). In a **multi-scheme** SDK there is no override — module and
> classes are snake/Pascal-cased straight from the scheme's own name, so the casing is whatever the spec
> wrote. The client's auth-manager property is snake_cased from the scheme *key*, which is a third name
> again: `client.oauth_2` sitting alongside `o_auth_2.py` is normal. `ls paypalserversdk/http/auth/`
> rather than guessing any of them. The snippets below show the **shape**: a credentials object you
> construct and pass as its own kwarg.
>
> **Get the real names from two places, before you write any auth code:** the `*_credentials`
> parameters on `Configuration.__init__` in `paypalserversdk/configuration.py`, and the scheme's page
> under `doc/auth/` — which carries the exact import line, arguments and kwarg. `doc/client.md` also
> shows a fully literal snippet. Only the schemes your API actually uses are generated.

## Basic auth

```python
import os

from paypalserversdk.http.auth.{basic_module} import {BasicCredentials}
from paypalserversdk.paypal_serversdk_client import PaypalServersdkClient

client = PaypalServersdkClient(
    {basic}_credentials={BasicCredentials}(
        username=os.environ['{API}_USERNAME'],
        password=os.environ['{API}_PASSWORD'],
    )
)
```

## API key — header or query parameter

The key is sent as a header or a query parameter; which one, and under what name, is fixed by the
generated scheme. The credentials class takes **one argument per generated auth parameter**, named after
the parameter in the spec — read `doc/auth/` for the real names:

```python
from paypalserversdk.http.auth.{api_key_module} import {ApiKeyCredentials}

client = PaypalServersdkClient(
    {api_key}_credentials={ApiKeyCredentials}(
        {api_key_param}=os.environ['{API}_KEY'],
    )
)
```

## OAuth 2.0 bearer token

```python
from paypalserversdk.http.auth.{bearer_module} import {BearerCredentials}

client = PaypalServersdkClient(
    {bearer}_credentials={BearerCredentials}(
        access_token=os.environ['{API}_ACCESS_TOKEN'],
    )
)
```

## OAuth 2.0 — client credentials

```python
from paypalserversdk.http.auth.{ccg_module} import {CcgCredentials}

client = PaypalServersdkClient(
    {ccg}_credentials={CcgCredentials}(
        {client_id_arg}=os.environ['{API}_CLIENT_ID'],
        {client_secret_arg}=os.environ['{API}_CLIENT_SECRET'],
    )
)
```

The SDK fetches the token on the first call that needs it and refreshes it when it expires. To persist
the token across restarts, or to hand the SDK a token you already hold, pass the
`o_auth_on_token_update` and `o_auth_token_provider` callbacks. They are **constructor arguments of the
credentials class**, not parameters of the client or `Configuration` — reading
`Configuration.__init__` will not show them — and they keep those exact names whatever the scheme is
called. See [reference.md](reference.md).

## More schemes

For OAuth 2 **authorization code** (the redirect flow, including `get_authorization_url`,
`fetch_token`, `is_token_expired` and `refresh_token`), **resource-owner password**, **custom
authentication**, **multiple schemes**, and reading credentials from the environment, see
[reference.md](reference.md).

## Notes

- **Every argument the credentials class marks as required is validated in `__init__`** — passing
  `None` raises `ValueError` immediately, at client construction, not at the first call.
- A given SDK only exposes the credentials classes and kwargs for the schemes its API uses; the class
  names and their argument names are generated per-API, which is why the samples above carry
  `{...}` placeholders.
- Credentials are set when the client is constructed. To change them later, build a new credentials
  object (or call `clone_with(...)` on the existing one), pass it through
  `client.config.clone_with(...)`, and construct a new client from that configuration — mutating the
  live client does not work.
- Keep secrets out of source. Read them from environment variables or a secrets manager, never
  hardcode them. Each credentials class also has a `from_environment()` classmethod that builds it from
  a fixed set of `UPPER_SNAKE` variables — see [reference.md](reference.md).
