---
name: 'typescript-models'
description: 'Construct and read the non-obvious model shapes of an APIMatic-generated TypeScript/Node.js SDK — oneOf/anyOf union types (built as plain object literals, narrowed on the way back with the generated is{Variant} type guards), TypeScript `enum` declarations (reference the member, never a bare string), collections (Array<T>), date/time fields that are plain strings the SDK neither converts nor format-checks, and unknown-field behavior. Use when building a request body or reading a response field of the PayPal Server SDK TypeScript SDK that is a union, enum, list/map, or date — anything that isn''t a plain string/number — or when an unmodeled JSON field is dropped on deserialization. Load it even after reading the field''s type in the source, since the type name alone won''t tell you that a union is validated at call time and rejects a value matching more than one variant.'
---

# Working with models in an APIMatic TypeScript SDK

Most request/response data are plain TypeScript objects conforming to interfaces (covered in `typescript-calling-endpoints`). This skill covers the **non-obvious model shapes** that trip integrations up. The patterns are generic across APIMatic TypeScript SDKs; take the real type names from your SDK source.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{Union}`, `{Variant}`, `{EnumType}`, `{RequestType}`) — replace it with the concrete identifier from the source.

## Union types (oneOf / anyOf)

When a field can be one of several types, APIMatic generates a union type in
`src/models/containers/`: a TypeScript union of the variant interfaces, plus a runtime schema built with
either `oneOf([...])` or `anyOf([...])`. **The two behave differently and you must tell them apart.**

The only reliable signals are the container file's doc comment (`This is a container type for one-of
types` / `... for any-of types`) and the literal `oneOf([...])` / `anyOf([...])` call in its source. Do
**not** infer it from the schema's runtime `type()` string or from the error text — an `anyOf` container
also reports itself as `OneOf<...>`.

### Construct — a plain object literal

There are **no factory helpers**. Assign the variant's own shape directly:

```typescript
const body: {RequestType} = {
  {field}: {                 // typed as {Union}
    type: '{variantTag}',    // the variant's discriminating value, if it has one
    someField: 'value',
  },
};
```

### Read — narrow with the generated type guards

The union's namespace exports one `is{Variant}` guard per variant. These are **read-side only**:

```typescript
import { {Union} } from '@paypal/paypal-server-sdk';

if ({Union}.is{Variant}(response.result.{field})) {
  // narrowed to {Variant}
}
```

A tagged union can also be narrowed on its discriminator:

```typescript
switch (response.result.{field}.type) {
  case '{variantTag}':
    break;
}
```

### The failure mode to know about

Both kinds are validated **when you make the call**, client-side, before any HTTP request.

`oneOf` requires **exactly one** variant to match, so two errors come out of it:

- `Matched more than one type` — your value satisfies several variants at once. Usually the variants'
  discriminating field is unconstrained in the generated schema, so no value can disambiguate them. Set
  the discriminator explicitly; if it still fails, the union cannot be satisfied from the typed API and
  the SDK needs regenerating.
- `Could not match against any acceptable type` — a required field is missing, or the discriminator
  value is not one the variant accepts.

`anyOf` requires **at least one** match, so it can only ever raise the second error — a value matching
several variants is accepted. Don't chase a "matched more than one" diagnosis on an `anyOf` container.

Open the container file under `src/models/containers/` and read the variant list and each variant's
schema before debugging your own input.

## Collections

List/array properties are `Array<T>` (or `T[]`); maps are `Record<string, V>`. Assign a plain array or object directly:

```typescript
const body: {RequestType} = {
  {listProp}: ['A', 'B'],                           // Array<string>
  {mapProp}: { key: 'value' },                      // Record<string, string>
};
```

An `undefined` collection is omitted from the JSON; an explicit `null` is sent **as `null`** (only allowed
where the property type includes `| null`); an empty array `[]` is serialized as `[]`.

## Dates & numbers

- Date/time fields are plain `string` properties with a bare `string()` runtime schema — **the SDK does
  no date conversion and no format checking**. The expected format is whatever the spec declares, and it
  is not always ISO-8601: a field documented `Format: dd-MM-yyyy HH:mm:ss` rejects
  `new Date().toISOString()` at the API, with nothing client-side to warn you. Read the field's own doc
  comment in `src/models/` (or its `@param` line in `src/controllers/`) and format the string to match.
- Money/quantities may be `string`, `number`, or a union; the model's property type is the source of truth.
- Numeric IDs are typically `number`.

## Enums

Enums are TypeScript `enum` declarations exported from the SDK (member = wire value). Reference the
member — never a bare string literal:

```typescript
import { {EnumType} } from '@paypal/paypal-server-sdk';

request.{enumProp} = {EnumType}.SomeConstant;
```

An enum member is still assignable to `string` (or `number` for a numeric enum), so reading one into your
own code needs no conversion. There is **no raw-string escape hatch**: `'value' as {EnumType}` does not
compile (TS2352, the types do not overlap), and even forced through with
`'value' as unknown as {EnumType}` the runtime schema rejects it before any HTTP request —
`stringEnum(X)` accepts declared members only. Unknown values are tolerated only when the enum's own file
builds its schema as `stringEnum(X, true)`, which the generator emits solely for specs that mark the enum
as accepting additional values.

See [reference.md](reference.md) for the full enum declaration shape.

## Unknown / future fields

Whether unknown response fields survive depends on the SDK's additional-properties generator setting.
The discovery signal is the model's own file — its schema call, and the interface beside it:

- **`expandoObject({...})`**, paired with an index signature **`[key: string]: unknown`** on the
  interface → unknown response fields land on the object **itself** (`result.someUnknownField`), and any
  extra key you set on a request object is serialized into the body as-is. There is **no**
  `additionalProperties` member: reading `result.additionalProperties` gives `undefined`, and writing
  `{ additionalProperties: {...} }` still compiles against the index signature and sends a literal
  `additionalProperties` key.
- **`typedExpandoObject({...}, '<key>', <valueSchema>)`** → the extras are collected under the **named
  member given as its second argument**, typed `Record<string, T>` on the interface. Read that name off
  the call; it is not a fixed word.
- A plain **`object({...})`** → unknown fields are dropped on deserialization.

Check the model interface and its schema before concluding a field was lost — reaching for a regenerate
or a hand-rolled parse is a dead end when the data is already sitting on the returned object.
