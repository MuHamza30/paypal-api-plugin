# F004 — five of six packs route the reader to a `doc/` tree their published package does not ship

**Severity: HIGH (navigation).** **Disposition: `accept-corpus` + `accept-rule`.** Scope: **five
languages need a fix; PHP needs none.** Source: round-01 TypeScript debrief §D1 `[MEASURED]`;
cross-registry verification mine, all six artifacts downloaded and listed.

## What the packs say

Every pack leans on `doc/` as the primary lookup route:

| Pack | `doc/` mentions in the rendered skills |
| --- | --- |
| python | 49 |
| **csharp** | **47** |
| ruby | 33 |
| php | 28 |
| typescript | 25 |
| java | 24 |

TypeScript's getting-started names `node_modules/@paypal/paypal-server-sdk/`, lists `doc/client.md`,
`doc/controllers/*.md`, `doc/models/*.md`, and calls grepping `doc/` **"the fastest way to find an
operation"**. `typescript-error-handling` sends the reader to `doc/controllers/{group}.md`.

## What is true — measured per registry

| Registry | `doc/` in the published package? | Top level | `doc/` in the repo at that tag? |
| --- | --- | --- | --- |
| npm | **No** | `LICENSE README.md dist/ package.json src/` | yes |
| PyPI (sdist) | **No** | `LICENSE MANIFEST.in PKG-INFO README.md …egg-info/ paypalserversdk/ pyproject.toml setup.cfg` | yes |
| NuGet | **No** | `_rels/ *.nuspec lib/ README.md LICENSE [Content_Types].xml package/ .signature.p7s` | yes |
| Maven | **No** (main jar `META-INF/ com/`; sources jar `META-INF/ com/`) | — | yes |
| RubyGems | **No** | `LICENSE README.md lib/` | yes |
| **Packagist / Composer** | **YES — 383 `doc/` entries** | `.gitattributes .gitignore .phan/ LICENSE README.md composer.json doc/ phpcs-ruleset.xml src/` | yes |

**PHP is the exception and its advice is already correct.** Composer installs from a GitHub zipball,
and the repo's `.gitattributes` at tag `2.0.0` is one line — `**/*.php -lf` — with **no `export-ignore`
directives**, so `vendor/paypal/paypal-server-sdk/doc/` genuinely exists after `composer install`.

**Maven's `-javadoc.jar` is not the `doc/` tree.** It is generated HTML API reference — `index.html`,
per-class pages under `com/` — with no `client.md`, no `controllers/orders.md`, no request/response
examples. A reader told "look in `doc/`" who finds the javadoc jar has found a different artifact
answering a different question. The `-sources.jar` is `.java` files only.

## The sharpest instance is C#, which also has the second-most `doc/` mentions

The `.nupkg` contains **zero `.cs` files** — `lib/netstandard2.0/PayPalServerSDK.dll` plus
`PayPalServerSDK.xml`. That XML is IntelliSense documentation: 1.5 MB, 5,746 `<member>` entries,
greppable for method names and summaries but carrying no usage examples. **A .NET developer following
"grep `doc/`" has neither markdown to grep nor source to read.**

This also contradicts the C# getting-started's "SDK source — read it in place" framing, which assumes
the source is on disk after install. For NuGet it is not.

## Severity is routing, not correctness

The round-01 build produced no wrong code from this: both skills give a working fallback and the agent
used it. The cost was wasted lookups and an unresolved contradiction. But a reader who trusts the
advertised fastest route finds nothing and is given no reason why.

## Why this is generic

It names no provider and no endpoint. It is a fact about **what an APIMatic-generated package ships
versus what its repository contains** — true for every API this generator builds. The
`{{SDK_SOURCE}}` / `{{PACKAGE_SOURCE}}` tokens already encode the installed-vs-repository distinction;
the prose does not respect it.

## The fix — five different truths, one per language

`CLAUDE.md`'s "a corrected claim must not be copied across packs" rule applies with unusual force here.
Each pack's package-local fallback is genuinely different, taken from the real listings:

| Pack | What to say |
| --- | --- |
| **php** | **No change.** `doc/` ships in `vendor/`. Verify before touching a line. |
| **typescript** | `doc/` is repository-only; the package ships TS source — `src/controllers/*.ts` (also compiled under `dist/cjs/`, `dist/esm/`). |
| **python** | `doc/` is repository-only; the package ships `paypalserversdk/controllers/*.py` plus `base_controller.py`. |
| **ruby** | `doc/` is repository-only; the gem ships `lib/paypal_server_sdk/controllers/*.rb` and `.../models/`. |
| **java** | `doc/` is repository-only; the main jar is `.class` files. The **sources jar** carries `com/…/controllers/*.java`. Do not point at the javadoc jar as if it were `doc/`. |
| **csharp** | `doc/` is repository-only **and no source ships at all**. The only package-local reference is `PayPalServerSDK.xml` (IntelliSense XML). Say so, and route the reader to the repository for anything else. |

## Proposed rule for `Corpus/CLAUDE.md`

> **Do not point at a path without saying which artifact contains it.** A generated SDK's repository
> and its published package are different trees, and which files survive packaging is a per-registry
> fact — npm's `files`, Python's `MANIFEST.in`, a `.nuspec`, a jar's layout, a gemspec's `files`,
> Composer's `.gitattributes`. A lookup route the reader cannot reach from `{{INSTALL_COMMAND}}` alone
> must say so and name the package-local alternative first. Verify per registry: in one measured case
> the package shipped no source at all, and in another the docs did ship.

**Guard** (pins the fact, not the wording): any skill naming a `doc/` path must, in the same section,
also name a path reachable from the installed package — or state that none exists.
