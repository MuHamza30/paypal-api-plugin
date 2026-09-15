# F001 — the spec bundle omits an environment the published SDK has

**Severity: RAISED to HIGH (blocks production deployment).** **Disposition: `reject-spec`** — route to the spec owner, never a
corpus edit.

> ## CORRECTED 
>
> An earlier version of this finding claimed the published SDK **defaults to Production** because
> `Production` is the first enum member, and therefore that an agent who sets no environment would
> issue live calls. **That was wrong.** The C# enum's member order does not decide the client's
> default — a `Builder` field initializer does, and the published SDK's reads:
>
> ```csharp
> private Environment _environment = PaypalServerSdk.Standard.Environment.Sandbox;
> ```
>
> **The default is Sandbox.** There is no live-call hazard here. The generator never picks "the first
> declared environment" as the default in any language; it reads an explicit
> `ServerConfiguration.DefaultEnvironment` string from the API definition
> (`APIMatic.SdlMapper/SDL/ServerConfiguration.cs:37`, copied verbatim at `:52`).
>
> The separate, real ordinal traps that misled me are now tracked in **F002**, where they belong.

## What is actually true `[MEASURED]`

| | Our spec bundle | Published `paypal/PayPal-Dotnet-Server-SDK` |
| --- | --- | --- |
| Environments declared | `["Sandbox"]` | `Production`, `Sandbox` |
| `DefaultEnvironment` | `"Sandbox"` | resolves to `Sandbox` (Builder initializer) |
| Controllers | 5 business groups | identical 5 + `OAuthAuthorizationController` |

So the divergence is **one missing environment**, and the defaults agree.

## Why it still matters

The generated plugin's environments table renders a single row, `Environment.Sandbox | Server.Default`.
An agent asked to take the integration to production has no way to learn from the plugin that
`Environment.Production` exists in the SDK it installed. It would either invent a base-URL override —
which C# does not offer, there is no base-URL setter — or report the capability as absent.

It is also a standing reminder that **the bundle we were given is not the bundle the live SDKs were
generated from.** That bounds what any measurement taken through this plugin can claim about the SDK a
real user installs.

## Independently confirmed from the other direction — and it is worse than "completeness"

Round-01 A3 csharp, §D3, `[MEASURED]`:

> "`csharp-getting-started`, Environments table — incomplete. It lists exactly one row,
> `Environment.Sandbox` -> `Server.Default`. The generated `Environment.cs` declares **both
> `Production` and `Sandbox`**, and the environments map in `PaypalServerSdkClient.cs` has
> `Production` -> `https://api-m.paypal.com`. **Had I trusted the table I would have shipped a service
> with no route to production.** I only caught it because the skill's own surrounding prose tells you
> to read `Environment.cs`."

Two things follow.

**The severity is higher than first assessed.** This is not a missing row in a reference table; it is
an integration that cannot be deployed. The builder was saved only by the corpus's *other* instruction
-- read the enum yourself -- which is the rule working as designed and covering for the token.

**The token is faithful and the bundle is not.** `{{ENVIRONMENTS}}` rendered exactly what
`ServerConfiguration` declared. The spec bundle we were given is missing an environment the published
SDK has, so every plugin generated from it understates the SDK. Still `reject-spec`; still not a corpus
edit.

## Action

File against the spec bundle. **Pre-registered as a known non-finding** for the debrief loop: every
round will surface "the environments table lists only Sandbox", and triage must route it out without
re-litigating. The token resolved the spec faithfully; the corpus is not at fault.
