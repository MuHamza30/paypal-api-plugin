# F009 — version skew: PayPal's published 2.0.0 pre-dates this generator (4 rejections)

**Reported as:** "getting-started says runtime dependencies are `@apimatic/core`, `@apimatic/schema`,
`@apimatic/axios-client-adapter`, `@apimatic/authentication-adapters` **and `tslib`**. In this SDK
`tslib` is a devDependency." Source: round-01 typescript debrief §D2, `[MEASURED]`.

**Verdict: REJECTED as a corpus defect. The corpus matches the current generator.**
The published SDK does not.

## The evidence `[MEASURED]`

The published `@paypal/paypal-server-sdk@2.0.0` really does carry `tslib` under `devDependencies` --
the agent read that correctly.

But the generator says the opposite, deliberately and with its reason recorded in place.
`TS/Templates/Generic/PackageFileCreator.cs:88-93`:

```csharp
// tsconfig sets importHelpers: true, so tsc emits require("tslib") in the
// compiled output; it must be a real dependency, not a devDependency, or
// consumers aren't guaranteed to have it installed at runtime.
package.AddDependency("tslib", "^2.5.0");
```

`AddDependency` and `AddDevDependency` are distinct methods on `PackageModel`. The current generator
puts `tslib` in `dependencies` on purpose, because `importHelpers: true` makes the compiled output
`require("tslib")` at runtime. **The corpus states what the generator emits, which is the rule.**

So the published package predates that decision, or its pipeline rewrote `package.json`. Either way
the divergence is between PayPal's artifact and today's generator -- not between the corpus and the
generator.

## The systemic implication, which is the real finding

This is the second confirmed instance, after **F001** (the spec bundle declaring one environment where
the published SDK declares two), that **PayPal's published 2.0.0 SDKs were not produced by the
generator this corpus documents.**

That bounds every defect this probe can report. When a debrief says "the skill claims X, the SDK does
Y", there are now three possible causes and they need different owners:

1. the corpus is wrong about the generator -> **corpus fix**
2. the generator is wrong -> **issue against the generator**
3. **the published SDK is older than the generator -> neither; the corpus is right for anything built today**

Cause 3 was not in the original triage rubric. It should be, and it is cheap to test: check the claim
against the template, not against the installed package.

## Triage rule to add

> Before accepting "the skill says X but the SDK does Y", check the **generator template**. If the
> template emits X, the finding is version skew in the probe's SDK, not a corpus defect. Do not
> "correct" the corpus toward a stale artifact -- that would make it wrong for every SDK generated
> from now on.

## Three more instances, from A3 typescript — same cause

The A3 typescript run reported five defects. **Three are this same version skew**, all `[MEASURED]`
against the installed package and all wrong about the corpus:

| Reported | Generator says |
| --- | --- |
| *"Module format: dual ESM/CJS (`exports` map…)" — there is no `exports` map* | `PackageFileCreator.GetExports()` emits one |
| *"`.` and `./metadata` are the only exported subpaths" — the reason is wrong, there are no subpaths at all* | those are **exactly** the two keys `GetExports()` emits |
| *`tslib` is a devDependency* | `AddDependency("tslib", "^2.5.0")`, deliberately |

`PackageFileCreator.cs:210-235` builds `exports` with `"."` and `"./metadata"`, each carrying
`import`/`require` arms. The corpus's claim — including the *reason* the agent called wrong — is
precisely right for anything this generator builds today.

So the installed `@paypal/paypal-server-sdk@2.0.0` has **no `exports` map, no `tslib` runtime
dependency, and one fewer environment** than the current generator would produce. That is a
substantial generation gap, not a rounding error.

## What this bounds

**A large fraction of "defects" reported against this probe are not corpus defects.** Four of the
reported defects so far resolve to "the artifact is older than the generator". Any summary of this
effort that counts reported defects without this filter overstates the corpus's fault rate badly.

It also means the *probe* is weaker than intended for TypeScript: the pack is being judged against an
SDK it does not describe. Where a claim matters, check the template.

## Tally so far

Three reported defects have now been rejected on exactly this discipline:

- **F007** `ApiHelper.JsonSerialize` "missing" -- inherited from the runtime base class; the call compiles
- **F009** `tslib` in the wrong section -- the generator deliberately puts it in `dependencies`
- the python "first enum member is falsy" claim -- true of `IntEnum`, and the generator emits `Enum`

Each was specific, plausible, and would have degraded correct prose. `CLAUDE.md`'s rule -- *"Ground
claims in the generator, then in the runtime package -- not in a sample SDK"* -- is carrying more
weight in this effort than any other single line in it.
