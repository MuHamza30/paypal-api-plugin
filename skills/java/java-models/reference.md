# Models reference (APIMatic Java)

The exact member shapes the Java generator emits, so you can recognise each one in the source.

## Plain model
This SDK was generated **without** immutable models, so this is the shape you will find:

```java
public class {Model} {
    private long id;          // required
    private String tag;       // optional

    public {Model}() { }
    public {Model}(long id, String tag) { ... }

    @JsonGetter("id")
    public long getId() { ... }

    @JsonSetter("id")
    public void setId(long id) { ... }

    @JsonGetter("tag")
    @JsonInclude(JsonInclude.Include.NON_NULL)
    public String getTag() { ... }

    @JsonSetter("tag")
    public void setTag(String tag) { ... }

    public Builder toBuilder() { ... }

    public static class Builder {
        public Builder() { }
        public Builder(long id) { ... }        // required fields only
        public Builder id(long id) { ... }     // every field gets a fluent setter
        public Builder tag(String tag) { ... }
        public {Model} build() { ... }
    }
}
```

The **`@JsonGetter`/`@JsonSetter` argument is the wire name**; the Java method name is a Pascal-cased
rendering of it and can differ (`user_name` → `getUserName()`). When the two disagree, the annotation
wins on the wire.

`@JsonInclude(JsonInclude.Include.NON_NULL)` appears on every optional field's getter — that is the
marker for "omitted when null".

## Nullable-optional field
```java
private OptionalNullable<String> tag;

@JsonGetter("tag")
@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonSerialize(using = OptionalNullable.Serializer.class)
protected OptionalNullable<String> internalGetTag() { ... }

public String getTag() { ... }                 // unwraps to the plain value

@JsonSetter("tag")
public void setTag(String tag) { ... }         // sets an explicit value, including null

public void unsetTag() { ... }                 // removes the field entirely
```

`OptionalNullable<T>` comes from `io.apimatic.core.types`. The `Builder` mirrors it with
`tag(String)` and `unsetTag()`.

## String-backed enum

```java
public enum {EnumType} {
    FIRST_VALUE,
    SECOND_VALUE;

    @JsonCreator
    public static {EnumType} constructFromString(String toConvert) throws IOException { ... }

    public static {EnumType} fromString(String toConvert) { ... }   // null when unmapped

    @JsonValue
    public String value() { ... }

    @Override
    public String toString() { ... }                                // returns the wire value

    public static List<String> toValue(List<{EnumType}> toConvert) { ... }
}
```

Usage:

```java
{EnumType} known   = {EnumType}.FIRST_VALUE;
{EnumType} parsed  = {EnumType}.fromString(rawFromSomewhereElse);   // may be null
String     onWire  = known.value();
```

`valueOf("first_value")` throws — it matches the **constant identifier** (`FIRST_VALUE`), not the wire
value. Always go through `fromString`.

## Integer-backed enum

Identical, with `Integer` substituted:

```java
public static {EnumType} fromInteger(Integer toConvert) { ... }

@JsonCreator
public static {EnumType} constructFromInteger(Integer toConvert) throws IOException { ... }

@JsonValue
public Integer value() { ... }

public static List<Integer> toValue(List<{EnumType}> toConvert) { ... }
```

## Unknown-tolerant enum

When the API allows values beyond the declared set, the enum gains one more member:

```java
public enum {EnumType} {
    FIRST_VALUE,
    SECOND_VALUE,
    _UNKNOWN;      // value() returns null
}
```

`fromString` maps anything unrecognised to `_UNKNOWN` instead of returning `null`, so a `switch` over
the declared members alone will miss it.

## oneOf / anyOf container

```java
package {rootPackage}.models.containers;

@JsonDeserialize(using = {Union}.{Union}Deserializer.class)
public abstract class {Union} {

    public static {Union} from{VariantA}({VariantAType} value) { ... }
    public static {Union} from{VariantB}({VariantBType} value) { ... }

    public abstract <R> R match(Cases<R> cases);

    public interface Cases<R> {
        R {variantA}({VariantAType} {variantA});
        R {variantB}({VariantBType} {variantB});
    }
}
```

- The case classes are `private static` — there is **no** public constructor and no `is{Variant}` guard.
- `from{Variant}(null)` returns `null` for nullable variant types rather than a wrapper around null.
- Some containers also declare `public abstract String getContentType();` or
  `public abstract Object value();`; both are optional and only appear for certain specs.

## Discriminated hierarchy

```java
@JsonTypeInfo(
        use = JsonTypeInfo.Id.NAME,
        include = JsonTypeInfo.As.EXISTING_PROPERTY,
        property = "{discriminatorField}",
        defaultImpl = {Parent}.class,
        visible = true)
@JsonSubTypes({
    @Type(value = {ChildA}.class, name = "{childAValue}"),
    @Type(value = {ChildB}.class, name = "{childBValue}")
})
@JsonInclude(Include.ALWAYS)
public class {Parent} { ... }
```

Each child's no-arg constructor sets the discriminator, and its `Builder` initializes the same field —
so you never set it by hand. `include = EXISTING_PROPERTY` with `visible = true` means the discriminator
is also a normal readable property on the model.

## Date and time types

| Spec type | Java type | Notes |
| --- | --- | --- |
| `date` | `java.time.LocalDate` | |
| `date-time` | `java.time.LocalDateTime` | default |
| `date-time` | `java.time.ZonedDateTime` | when the SDK was generated to keep timezone information |

The wire format is applied by Jackson annotations pointing at the generated `DateTimeHelper` (which
extends the runtime's `LocalDateTimeHelper` / `ZonedDateTimeHelper`) — RFC 3339 is the default, with
RFC 1123 and Unix-timestamp variants selected per field. You pass and receive `java.time` values; the
SDK handles the string form. `OffsetDateTime` is never generated.

## Notes

- A model's `toString()` is generated for every model and prints its fields — useful for a quick dump,
  but it is not JSON. Use `ApiHelper` (which extends `io.apimatic.core.utilities.CoreHelper`) if you need
  to serialize a model yourself.
- `equals`/`hashCode` are **not** generated for ordinary models — do not rely on value equality.
- Every field name you see in Java is a rendering of the wire name. When in doubt, read the
  `@JsonGetter("...")` argument, not the method name.
