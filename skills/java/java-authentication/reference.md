# Authentication reference (APIMatic Java)

Full matrix of auth schemes the APIMatic Java generator supports. Model classes live under
`<root>/authentication/`; only the **names** are generated per-API. When the API has a single scheme the
names come from a fixed per-type table; when it has several, they come from the security-scheme names in
the spec — so always confirm against `{Api}Client.java` and `doc/auth/`.

## Naming pattern

| Piece | Single-scheme API | Multi-scheme API |
| --- | --- | --- |
| Model class | `BasicAuthModel`, `BearerAuthModel`, `CustomHeaderAuthenticationModel`, `CustomQueryAuthenticationModel`, `CustomAuthenticationModel`, `ClientCredentialsAuthModel`, `AuthorizationCodeAuthModel`, `ResourceOwnerAuthModel` | `{SchemeName}Model` |
| Client `Builder` setter | camel-cased credentials-interface name — `.basicAuthCredentials(...)`, `.bearerAuthCredentials(...)`, `.customHeaderAuthenticationCredentials(...)`, `.customQueryAuthenticationCredentials(...)`, and for OAuth grants the suffix-less `.clientCredentialsAuth(...)`, `.authorizationCodeAuth(...)`, `.resourceOwnerAuth(...)` | `.{schemeName}Credentials(...)` for every scheme, OAuth included |
| Client getters | `get{X}Credentials()` + `get{X}Model()` (OAuth grants: `get{X}()` + `get{X}Model()`) | `get{SchemeName}Credentials()` + `get{SchemeName}Model()` |

A **custom auth scheme with no parameters** generates no model, no setter and no getter at all — the SDK
wires its manager itself.

Some single-scheme SDKs additionally carry **`@Deprecated` overloads that take bare strings** instead of
a model (e.g. `basicAuthCredentials(String username, String password)`,
`authorizationCodeAuthCredentials(String clientId, ...)`). They exist for backwards compatibility only —
if the `Builder` offers both, use the one that takes the `{Scheme}Model`.

## Basic

```java
.basicAuthCredentials(new BasicAuthModel.Builder(username, password).build())
```
Sends `Authorization: Basic base64(username:password)`.

## Bearer / access token

```java
.bearerAuthCredentials(new BearerAuthModel.Builder(accessToken).build())
```
A single-scheme bearer SDK also exposes a convenience `client.getAccessToken()`.

## API key — header or query

One constructor argument per scheme parameter; placement (header name or query parameter name) is fixed
by the generated scheme.

```java
.customHeaderAuthenticationCredentials(new CustomHeaderAuthenticationModel.Builder(token, apiKey).build())
.customQueryAuthenticationCredentials(new CustomQueryAuthenticationModel.Builder(token, apiKey).build())
```

## OAuth 2.0 — client credentials (machine to machine)

```java
.clientCredentialsAuth(new ClientCredentialsAuthModel.Builder(clientId, clientSecret)
        .oAuthScopes(Arrays.asList({OAuthScopeEnum}.{SCOPE}))   // only when the API declares scopes
        .build())
```

The scope enum is generated under `<root>/models/`; read the model's `oAuthScopes` setter for its exact
type name, which carries the scheme name when the API has several schemes.

This grant is **driven by the SDK**: before a call that needs it, the manager checks the cached token,
and fetches a new one when it is missing or expired. Nothing else is required of you.

Optional builder setters on this model only:

| Setter | Type | Purpose |
| --- | --- | --- |
| `oAuthToken(OAuthToken)` | the generated token model | seed a token you already hold |
| `oAuthTokenProvider(BiFunction<OAuthToken, ClientCredentialsAuth, OAuthToken>)` | callback | supply the token yourself — e.g. load it from a shared store; called whenever the cached token is missing or expired |
| `oAuthOnTokenUpdate(Consumer<OAuthToken>)` | callback | fires whenever the token changes — persist it here |
| `oAuthClockSkew(long)` | seconds | treat a token as expired this long before its real expiry |

The token model is a normal generated class under `<root>/models/`, usually called `OAuthToken`; its
exact name is generated per API, so read the model's `oAuthToken` setter for the real type. The **casing
is not**: every OAuth identifier the generator names itself — the token model `OAuthToken` and the
`oAuthClientId` / `oAuthClientSecret` / `oAuthToken` / `oAuthTokenProvider` / `oAuthOnTokenUpdate` /
`oAuthClockSkew` fields, `Builder` setters and getters — spells it `oAuth`/`OAuth` with a capital `A`,
so copy the spelling above. The *scheme* half of a class or setter name is different: in a multi-scheme
SDK it is built from the security-scheme name in the spec (`{SchemeName}Model`,
`.{schemeName}Credentials(...)`), so its casing is the spec author's rather than the generator's — take
that half from `{Api}Client.java`.

The provider is a plain `BiFunction`, and `BiFunction.apply` declares no checked exceptions — but
`fetchToken()` declares `throws ApiException, IOException`. Both are checked, so the fetch **must** be
caught inside the lambda and rethrown unchecked; calling `fetchToken()` bare there does not compile.

```java
.clientCredentialsAuth(new ClientCredentialsAuthModel.Builder(clientId, clientSecret)
        .oAuthTokenProvider((lastOAuthToken, credentialsManager) -> {
            OAuthToken stored = loadTokenFromDatabase();
            if (stored != null && !credentialsManager.isTokenExpired(stored)) {
                return stored;
            }
            try {
                return credentialsManager.fetchToken();
            } catch (ApiException | IOException e) {
                throw new IllegalStateException("OAuth token fetch failed", e);
            }
        })
        .oAuthOnTokenUpdate(oAuthToken -> saveTokenToDatabase(oAuthToken))
        .build())
```

Methods available on `client.getClientCredentialsAuth()`: `fetchToken()`,
`fetchToken(Map<String, Object> additionalParameters)`, `isTokenExpired()`,
`isTokenExpired(OAuthToken)` — plus `fetchTokenAsync(...)` when the SDK was generated in asynchronous
mode. `fetchToken` declares `throws ApiException, IOException`.

## OAuth 2.0 — authorization code (3-legged)

**Not automatic.** The SDK builds the authorization URL and exchanges the code, but you drive the flow
and then **rebuild the client** with the token:

```java
// 1. send the user here
String authUrl = client.getAuthorizationCodeAuth().buildAuthorizationUrl();

// 2. your redirect endpoint receives ?code=...
OAuthToken token = client.getAuthorizationCodeAuth().fetchToken(authorizationCode);

// 3. re-instantiate the client with the token
client = client.newBuilder()
        .authorizationCodeAuth(client.getAuthorizationCodeAuthModel().toBuilder()
                .oAuthToken(token)
                .build())
        .build();
```

The model's `Builder` constructor takes the client id, the client secret and the redirect URI, in that
order. Other methods on the credentials interface: `buildAuthorizationUrl(String state)`,
`buildAuthorizationUrl(String state, Map<String, String> additionalParameters)`,
`fetchToken(String authorizationCode, Map<String, Object> additionalParameters)`, `refreshToken()`,
`refreshToken(Map<String, Object> additionalParameters)`, `isTokenExpired()`.

Until a token is attached, calls fail with an auth error whose message says an OAuth token is needed —
that is the symptom of skipping step 3.

## OAuth 2.0 — resource owner password

Same as authorization code minus the browser step: no `buildAuthorizationUrl`, and the model's `Builder`
constructor takes the client id, the client secret, the username and the password. Call `fetchToken()`
and rebuild the client exactly as in step 3 above.

## Token persistence & refresh

- The token model is a normal generated class in `<root>/models/` — Jackson-serializable, so you can
  store it as JSON.
- For client credentials, `oAuthOnTokenUpdate` is the persistence hook and `oAuthTokenProvider` is the
  load hook; together they survive a process restart without a new token round-trip.
- For the other grants there is no automatic refresh: call `refreshToken()` yourself and rebuild the
  client, or catch the auth failure and redo the flow.

## Combined / multiple schemes

There is no combined credentials object. Set **every** scheme the endpoints you call require; the
generated controller applies the AND/OR composition per operation:

```java
{Api}Client client = new {Api}Client.Builder()
        .basicAuthCredentials(new BasicAuthModel.Builder(u, p).build())
        .apiKeyCredentials(new ApiKeyModel.Builder(key).build())
        .build();
```

- **AND** — every scheme in the group is applied to the request.
- **OR** — the first satisfied scheme is used.

Which operations need which group is baked into each controller method; `doc/controllers/*.md` and the
`.withAuth(...)` block in the controller source show it.

## No auth
Some APIs (or individual operations) need no credentials — build the client without any credential
setter. **This API is not one of them at the API level**: it declares at least one scheme, so
`<root>/authentication/` exists. Individual operations may still require none — the `.withAuth(...)` block
in the controller method is what decides per operation.

## Discovering what a specific SDK uses

1. Open `{Api}Client.java` and list the `Builder` methods ending in `Credentials` (or the suffix-less
   OAuth ones) — this is the **source of truth** for what the SDK accepts.
2. Open the matching `{Scheme}Model` under `<root>/authentication/` and read its `Builder` constructor
   for the required credentials, and its fluent setters for the optional ones.
3. `doc/auth/*.md` restates the same thing with a worked snippet.
