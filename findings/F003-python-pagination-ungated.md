# F003 — python ships pagination guidance for a surface that does not exist

**Severity: HIGH (breaks code).** **Disposition: `accept-corpus`.** Scope: **python only**.
Found before any integration ran, by rendering the pack and diffing it against the published SDK.
All claims `[MEASURED]`.

## What ships

`python-calling-endpoints/SKILL.md`, `## Paginated operations` — rendered, verbatim:

> **Only when this SDK ships pagination** — check for a `paypalserversdk/utilities/pagination/` package.
> Many SDKs have none; paging is then manual…
>
> When it is present, an operation the API marks as paginated returns a `PagedIterable` instead of a plain
> list. Iterate it directly for items, or `.pages()` for whole pages — the SDK fetches each page lazily as
> you advance:
>
> ```python
> for item in client.{controller}.{operation}():
>     process(item)
> ```

`python-configuration-resilience/SKILL.md` carries a matching page-level section, and the text above
cross-references it.

## What is true

The published `paypal/PayPal-Python-Server-SDK` ships `paypalserversdk/utilities/` containing exactly
`__init__.py` and `file_wrapper.py`. **There is no `pagination/` package and no `PagedIterable`.**
No PayPal spec declares a pagination strategy, so `HAS_PAGINATION` is false.

An agent following this section writes `for item in client.x.y():` expecting lazy paging. The operation
actually returns an `ApiResponse` whose `.body` is an ordinary object. The code does not work.

## Root cause — a pure corpus gap, no token work needed

`Python/Skills/PythonSkillTokens.cs:90` **declares `HAS_PAGINATION`**, and it is reachable: the
getting-started return-type row already uses it and renders correctly.

The flag is simply **not applied to the pagination sections**. Measured uses of `HAS_PAGINATION` per pack:

| Pack | uses |
| --- | --- |
| typescript | 13 |
| java | 11 |
| ruby | 9 |
| csharp | 8 |
| php | 2 (sufficient — PHP generates no pagination in any build) |
| **python** | **1** — a single clause in one getting-started table row |

Every other pack resolves the fact outright. Verbatim, for contrast:

- typescript — "**No operation in this API is paginated**, so no method returns a `PagedAsyncIterable`"
- java — "**This API declares no paginated operation.** The SDK ships no `<root>/utilities/pagination/` package"
- php — "**This API declares no paginated operation**, and PHP SDKs generate no pagination support in any case"

This is the measured instance of `CLAUDE.md`'s own warning that python "was excluded from the
re-grounding pass that hardened the other packs."

## Which rules it breaks

1. *"Where a claim varies with a setting, **gate it rather than hedge it**: the shipped skill then states
   the one fact true for the SDK beside it."* — python hedges ("Only when this SDK ships pagination —
   check for…"), pushing a lookup onto the reader that generation already knows the answer to.
2. *"**Always state the absence case.**"* — the absence case is not stated. Worse, the hedge is followed
   by a **positive code sample**, which reads as "these exist, go use them" — the exact failure mode the
   rule exists to prevent.

## The fix (in bounds, generic, no PayPal content)

Gate both sections on `{{#if HAS_PAGINATION}}` with a real `{{else}}` absence branch, **re-derived from
the Python generator** — not copied from the java or typescript wording, per `CLAUDE.md`'s rule that a
corrected claim must never be copied across packs. Engine constraints to respect: no nesting, keep a
blank line between adjacent blocks, and do not make a single table row conditional.

Guard: a `SkillsDataTests` case asserting the python pack's pagination content is reachable only through
`HAS_PAGINATION` — pinning the fact, not the wording.

## Note for the sweep

`python-configuration-resilience` must be fixed in the **same commit** as
`python-calling-endpoints` — they cross-reference each other, and `CLAUDE.md` warns that a partial fix
"is worse than none — it reads as verified."

## Context: python has the thinnest conditional coverage of the six `[MEASURED]`

Total `{{#if}}` applications across each pack's markdown:

| Pack | `{{#if}}` applications | distinct flags used |
| --- | --- | --- |
| ruby | 86 | 7 |
| java | 76 | 8 |
| php | 52 | 7 |
| typescript | 51 | 6 |
| csharp | 43 | 7 |
| **python** | **27** | 8 |

Python uses all eight of its declared flags at least once — so there are **no dead flags** — but it
applies them 27 times against 43–86 elsewhere. The pagination section is the proven, code-breaking
consequence; the remainder of the pack has not been audited and should not be assumed equivalent.

**Open item, not yet a finding:** `python-authentication` applies `HAS_AUTH` once (java and ruby apply
it 14 times, typescript 9, php 8). The rendered `reference.md` carries a `## No auth` section for an API
that does declare a scheme. The section is *placed inside the has-auth branch* and its text is not
false, so this is **not yet classified as a defect** — it needs a read of the surrounding conditional
structure before anyone edits it. Do not act on it as written.
