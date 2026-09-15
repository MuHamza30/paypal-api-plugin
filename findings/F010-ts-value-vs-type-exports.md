# F010 — the TypeScript pack never mentions the value/type export split

**Severity: MEDIUM (near-miss compile error).** **Disposition: `accept-corpus`, typescript.**
Source: round-01 typescript sample 2, §H, `[MEASURED]`. Single-session — needs one corroboration
before landing, per the corroboration rule.

## What happened

The agent wrote, from memory:

```ts
import { CheckoutPaymentIntent, OrdersCapture, Order, Refund } from '@paypal/paypal-server-sdk';
```

then grepped `src/index.ts` before compiling and found the root barrel distinguishes the two kinds:

- `export { CheckoutPaymentIntent }` — a **value** (a real TS `enum`, present at runtime)
- `export type { Order }`, `export type { OrdersCapture }`, `export type { Refund }` — **types only**

It split the import and compiled clean. Its own root-cause line:

> "Root cause had I not checked: I wrote from memory rather than looking up. **The plugin did not state
> the value/type export split; the source did.**"

## Why it is a genuine gap rather than a defect

Nothing the pack says is wrong. The fact is simply absent, and it is one a reader will get wrong from
habit, because the two kinds sit side by side in the same barrel and look identical at the call site
until `tsc` runs. Under `isolatedModules` or `verbatimModuleSyntax` — increasingly the default — mixing
them is a hard error rather than a style warning.

## Why it is generic

It names no provider. **Every** APIMatic TypeScript SDK emits this split: model interfaces are types,
generated enums are values. The pack already tells the reader that enums are real TS `enum`s
(`typescript-calling-endpoints`) and that models are plain interfaces (`typescript-models`) — it simply
never joins those two facts into the consequence for an `import` statement.

Whether the split appears at all is flag-dependent: on a build with `HAS_ENUMS` false there are no
generated enum values to contrast with, so any wording must be gated or phrased so it stays true.

## Proposed fix

One or two sentences in `typescript-getting-started`, where the single import specifier is introduced:
the root barrel exports models as **types** and generated enums as **values**, so a single `import
{ ... }` mixing both fails under `isolatedModules`; read `src/index.ts` to see which kind a name is.

Do not enumerate which names are which — that is per-API and belongs in the source, not the pack.

## Before landing

- Corroborate: one more session hitting the same split, or a direct read of the TS barrel template
  confirming `export type` is emitted for models and bare `export` for enums.
- Check the `HAS_ENUMS` false branch reads sensibly.
