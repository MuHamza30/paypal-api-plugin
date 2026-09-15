# F002 — the default environment is silent, and five languages trap differently

**Severity: MEDIUM-HIGH (safety + correctness).** **Disposition: `accept-corpus` + `accept-rule`.**
Scope: **all six**, with wording **re-derived per language** — this is a case where copying is
actively wrong.

> ## CORRECTED FRAMING
>
> The first version of this finding said the default environment is "whichever member the spec declared
> first". **That is false in five of six languages.** The generator reads an **explicit**
> `ServerConfiguration.DefaultEnvironment` string from the API definition
> (`APIMatic.SdlMapper/SDL/ServerConfiguration.cs:37`, copied verbatim at `:52`); 46 references across
> the templates all read that string, and none falls back to index 0. The only value the generator
> ever invents is for a spec carrying nothing but a `BaseUri`, where it synthesizes one environment
> named `"production"` (`:16-31`).
>
> The hazard is real but upstream of the language: **whatever the API definition names as the default
> is compiled in as what a caller who sets nothing receives, and no template warns, logs or errors.**

## The generic claim worth stating — true of all six

A generated client selects an environment when the caller sets none. **Which one was decided by the
API definition, not by the SDK, and it is not necessarily a test environment.** Read the default out of
your own SDK rather than assuming it; the generated README's configuration table states it explicitly
(`Doc/Util/ConfigVariableTableBuilder.cs:79-100` emits
``**Default: `Environment.X`**``), and it is the most portable place to point a reader.

## What each pack says today `[MEASURED]`

| Pack | Status |
| --- | --- |
| java | **Best** — "The default environment is whatever the `Builder`'s `environment` field is initialized to; **read it rather than assume**." |
| typescript | Partial — names where to read it, does not say it may not be safe |
| csharp | **Weak** — the phrase appears once, in passing, inside a paragraph about a *missing config section* |
| php · python · ruby | **Silent** |

And the concrete gap the C# tokens leave: `CS/Skills/CSharpSkillTokens.cs:135-156` renders a table of
environment members and their servers but **never marks which one is the default**.

## Five genuinely different per-language traps — do NOT copy one wording

This is the `CLAUDE.md` "three different truths" rule in its sharpest form. Each is grounded in the
template that emits it:

| Lang | The trap | Ground |
| --- | --- | --- |
| **C#** | Enum members are emitted with **no explicit values**, in declaration order. So `default(Environment)`, `(Environment)0`, a zero-initialized field, or a failed `Enum.TryParse` all silently mean the **first declared** member — which differs from the Builder's default whenever the spec's `DefaultEnvironment` is not also first-listed. | `CS/Templates/Generic/EnvironmentConfigEnumTemplate.tt:16-29` |
| **Python** | **Worse than C#.** Enum values *are* the declaration index (`PRODUCTION = 0`, `SANDBOX = 1`), so the first member is both `0` and **falsy** — `if config.environment:` misbehaves — and `from_value(0)` / `from_value("0")` reaches it. `from_value` silently returns `None` for anything unrecognised. | `Python/Templates/Generic/ConfigurationGenerator.cs:159-172`, `:49-76` |
| **Ruby** | **The only language with a true first-declared fallback.** `Environment.from_value`'s default parameter is `Environments.First()` — **not** the configured `DefaultEnvironment`. Any `nil` or unrecognised value resolves to the first environment the spec listed, printing only a `warn` to stderr. | `Ruby/Templates/Generic/ConfigurationTemplate.tt:28-30`, `:46-48` |
| **PHP** | **No enum at all.** `Environment` is a class of `public const` strings generated with the validator explicitly disabled (`shouldCheckValue: false`), and the setter does no validation. A typo'd environment is an undefined array key, not an error. | `PHP/.../ProjectTemplateManager.cs:516-519`, `PhpConfiguration.cs:508-517` |
| **TypeScript** | **The only one that hard-fails.** Unknown environment ⇒ `throw new Error('Could not get Base URL. Invalid environment or server.')`. So "it silently resolves to something" is **wrong** for TS. Its trap is different: passing `{ environment: undefined }` explicitly *overrides* the default via object spread. | `TS/.../ClientTemplate.tt:176-191`, `:106-109` |

Java has no ordinal trap (enum refs default to `null`), but `environmentMapper` ends in an
unconditional `return "<default env's URL>"`, and `.environment(null)` compiles and NPEs at request
time rather than being rejected at set time.

## Proposed rule for `Corpus/CLAUDE.md`

> **State what an unset environment selects.** A generated client picks an environment when the caller
> sets none, and which one is a fact of the API definition, not of the SDK — it may be the live one.
> Point the reader at where their own SDK records it; never assert which member it is. Where the
> language additionally lets an *unset or unrecognised* value resolve to something other than that
> default, say so in that language's own terms — the mechanism differs in all six and a copied
> sentence is false in at least three.

**Guard** (pins the fact, not the wording): every pack's `-client-initialization` skill that mentions a
default environment must also direct the reader to read the initialised value from their own source.

## Bonus finding, separate and worth its own item

**The `Server` enum genuinely *is* spec-order, in all six languages.** Every language builds it from
`ServerConfiguration.Environments[0].Servers` — the **first declared** environment's server list, not
the *default* environment's. If the first-listed environment declares different server aliases than the
default one, the generated `Server` enum is derived from the wrong environment. Grounded in six
template sites (e.g. `CS/.../ServerConfigEnumTemplate.tt:12`,
`Python/.../ConfigurationGenerator.cs:177`, `Ruby/.../ConfigurationTemplate.tt:59`).

This is a **generator** observation, not a corpus one — per `CLAUDE.md`, it earns an issue, and at most
a corpus sentence describing what is actually emitted.
