---
name: 'java-models'
description: 'Construct and read the non-obvious model shapes of an APIMatic-generated Java SDK — the nested `Builder` (required fields in its constructor, optional ones as fluent setters), nullable-optional fields wrapped in `OptionalNullable<T>` with an extra `unset{Field}()`, enums converted with `fromString`/`fromInteger` rather than `valueOf`, oneOf/anyOf containers under `models/containers` built with static `from{Variant}(...)` factories and unwrapped with `match(...)`, `LocalDate`/`LocalDateTime`/`ZonedDateTime` date types, and Jackson-discriminated polymorphic hierarchies. Use when building a request body or reading a response field of the PayPal Server SDK Java SDK that is a union, enum, list/map, date or nullable-optional — anything that isn''t a plain String or number. Load it even after reading the field''s Java type in the source, since the type name alone won''t tell you that `valueOf` resolves the wrong thing, or that a union needs a static factory rather than a constructor.'
---

# Working with models in an APIMatic Java SDK

Most request/response data are plain generated classes with a nested `Builder` (covered in
`java-calling-endpoints`). This skill covers the **non-obvious model shapes** that trip integrations up.
The patterns are generic across APIMatic Java SDKs; take the real type names from your SDK source under
`<root>/models/`.

> Throughout this skill, `{...}` is a placeholder for a name you take from your SDK (e.g. `{Model}`,
> `{Union}`, `{Variant}`, `{EnumType}`) — replace it with the concrete identifier from the source.

## The Builder split: required vs optional

Every model class carries a nested `public static class Builder`. The split is mechanical:

- **Required fields are constructor arguments** on the `Builder`.
- **Optional fields are fluent setters** — and so are the required ones, so you can also override a
  constructor value later.
- `build()` returns the model; `model.toBuilder()` returns a `Builder` pre-populated from an existing
  instance, which is how you derive a variant without re-listing every field.

```java
{Model} m = new {Model}.Builder(requiredA, requiredB)
        .optionalC(value)
        .build();

{Model} variant = m.toBuilder().optionalC(other).build();
```

A model with **no** required fields has no `Builder` constructor arguments at all — start from
`new {Model}.Builder()`. A model that extends another names its clone method after itself
(`to{Model}Builder()`) instead of `toBuilder()`.

**This SDK was generated without immutable models**, so each field also has a plain `@JsonSetter` setter
and the model has a no-arg constructor alongside the public all-args one. A model with required fields
additionally gets a **no-arg `Builder()` overload**, so `new {Model}.Builder()` compiles even there and
the required values can be supplied with the fluent setters instead — the `Builder` constructor is the
form that keeps them checked.

## Optional vs nullable-optional — the `unset` case

Two different things get called "optional":

- **Optional** — a plain boxed type (`String`, `Integer`, `Boolean`, …). Leave it unset and it is omitted
  from the JSON: the getter carries `@JsonInclude(JsonInclude.Include.NON_NULL)`.
- **Optional *and* nullable** — the field can be *absent* or explicitly `null`, which are different on
  the wire. The private field is an `OptionalNullable<T>`, and the model gains a **fourth** member:

```java
{Model} m = new {Model}.Builder()
        .{field}(null)     // send an explicit  "{field}": null
        .build();

{Model} n = new {Model}.Builder()
        .{field}(value)
        .unset{Field}()    // omit "{field}" entirely
        .build();
```

The getter still returns the plain `T` (`get{Field}()`); `unset{Field}()` exists on both the model and the
`Builder`, so you can also clear the field after the fact. If you see an `unset{Field}()` method, that
field distinguishes absent from null — and `.{field}(null)` is *not* the same as leaving it alone.

## Enums

Generated enums are real Java `enum` types with a string (or integer) wire value attached. **Do not use
`valueOf`** — that resolves the Java constant identifier, which is an upper-cased, sanitized version of
the name and is often not the wire value. Use the generated factories:

```java
{EnumType} e = {EnumType}.{CONSTANT};              // known constant
{EnumType} f = {EnumType}.fromString("wire_value"); // from a wire value; null when unmapped
String wire  = e.value();                          // back to the wire value
```

Integer-backed enums use `fromInteger(Integer)` and `Integer value()` instead. Both kinds also expose
`constructFromString`/`constructFromInteger` (the Jackson entry point, which **throws `IOException`** on
an unmapped value) and a `toValue(List<{EnumType}>)` helper for converting a whole list.

Some enums carry an extra `_UNKNOWN` member, generated when the API allows values outside the declared
set — `fromString` returns `_UNKNOWN` rather than `null` for anything it does not recognise, and
`_UNKNOWN.value()` is `null`. Check the enum for that member before writing an exhaustive `switch`.

## oneOf / anyOf union types

When a field can be one of several types, APIMatic generates an **abstract container class** under
`<root>/models/containers/`.

### Construct — a static factory, never a constructor

The case classes are private; the only way in is a static `from{Variant}(...)` factory on the container:

```java
import {rootPackage}.models.containers.{Union};

{Union} u = {Union}.from{Variant}(variantValue);
```

### Read — `match` with a `Cases<R>` visitor

The container declares `public abstract <R> R match(Cases<R> cases);` and a nested
`public interface Cases<R>` with one method per variant. That visitor is the exhaustive way to unwrap it:

```java
String described = u.match(new {Union}.Cases<String>() {
    @Override
    public String {variantA}({VariantAType} value) {
        return "A: " + value;
    }

    @Override
    public String {variantB}({VariantBType} value) {
        return "B: " + value;
    }
});
```

Open the container file and read its `Cases` interface for the real variant method names and types —
they come from the spec, not from a fixed convention. A discriminated union additionally registers its
discriminator values in the container's deserializer, so the right case is selected automatically on the
way in.

## Collections, dates and numbers

- List properties are `java.util.List<T>`; map properties are `java.util.Map<String, V>`. Assign an
  ordinary `Arrays.asList(...)` / `HashMap`. A `null` collection is omitted from the JSON when the
  property is **optional** (its getter carries `@JsonInclude(NON_NULL)`); a **required** collection has
  no such annotation, so `null` is written as an explicit `null`. An empty collection is always
  serialized.
- **Date/time fields are real `java.time` types**, not strings: `LocalDate` for a plain date, and either
  `LocalDateTime` or `ZonedDateTime` for a date-time, depending on whether the SDK was generated to keep
  timezone information. **`OffsetDateTime` is never generated** — if your own code uses it, convert at
  the boundary. The wire format (RFC 3339, RFC 1123, or a Unix timestamp) is handled for you by the
  generated `DateTimeHelper` through Jackson annotations; you never format the value yourself.
- Numeric and boolean fields are Java primitives (`long`, `int`, `double`, `boolean`) when required and
  non-nullable, and boxed (`Long`, `Integer`, `Double`, `Boolean`) as soon as they are optional or
  nullable — precisely so that `null` is expressible.

## Polymorphic (discriminated) hierarchies

A model with subtypes carries Jackson's `@JsonTypeInfo` / `@JsonSubTypes` at the top of the class, and
each child's own constructor and `Builder` pre-set its discriminator value, so you never assign it.
Deserialization picks the concrete subclass for you — a response field typed as the parent may
therefore hold a child at runtime. Use
`instanceof` (or read the discriminator getter) to narrow, and construct the **child** class directly
when sending one.

## Unknown / future fields

By default a model declares its properties explicitly and unknown JSON fields are dropped on
deserialization. Two opt-in shapes preserve them, and which one an SDK uses is fixed at generation time:

- the model `extends BaseModel`, inheriting a `getAdditionalProperties()` map; or
- the model holds its own `AdditionalProperties` field and exposes a public
  `getAdditionalProperty(String name)` — the map itself stays private, wired to Jackson via
  `@JsonAnyGetter`/`@JsonAnySetter`, and the `Builder` gains `additionalProperty(String name, T value)`
  for the write side.

Check the model class for either a base class or an `additionalProperty` member. If neither is present,
an unmodelled field cannot be read through the SDK — regenerate it or parse that response yourself.

See [reference.md](reference.md) for the exact emitted member shapes.
