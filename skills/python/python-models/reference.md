# Models reference (APIMatic Python)

## The `APIHelper.SKIP` sentinel

`APIHelper.SKIP` (from `paypalserversdk/api_helper.py`) is the "this value was not supplied" marker. It
appears in two places:

```python
def __init__(self, {required_attr}=None, {optional_attr}=APIHelper.SKIP):
    self.{required_attr} = {required_attr}
    if {optional_attr} is not APIHelper.SKIP:
        self.{optional_attr} = {optional_attr}
```

- **Constructor defaults** — every attribute listed in `_optionals` defaults to `SKIP`, and the guard
  above means an unsupplied one is never assigned — **except where the spec gives the attribute a
  default**, in which case the constructor takes that literal instead of `SKIP` and always assigns it.
  `doc/models/*.md` carries a **Default** column; that is the tell. Required attributes default to `None` and are always
  assigned.
- **`from_dictionary` usually tests truthiness, not key presence** — for a plain scalar optional the
  generated line is `x = dictionary.get('x') if dictionary.get('x') else SKIP`. So a **present but
  falsy** value — `0`, `""`, `[]` — is indistinguishable from an omitted key and the attribute ends up
  absent. A zero amount or an empty list cannot be read back. Where you need that distinction, read
  `e.response.text` / the raw dict rather than the model. **Boolean, model-typed and nullable optionals
  are the exception** — the generator tests `"x" in dictionary.keys()` for those, so a present `False`
  survives. Read the attribute's own line in `from_dictionary` rather than assuming either rule.
- **`from_dictionary`** — an optional key missing from the response resolves to `SKIP` too, so a
  response model has exactly the same absent-attribute behaviour as one you built yourself.

Checking for an optional value:

```python
value = getattr(model, '{optional_attr}', None)   # or: hasattr(model, '{optional_attr}')
```

Never pass `APIHelper.SKIP` explicitly — just omit the argument.

## Wire names — the `_names` map

```python
_names = {
    "mtype": "type",            # Python attribute -> API property
    "person_type": "personType",
}
```

Attribute names are snake_cased, and Python keywords and builtins are renamed (`type` becomes `mtype`).
When a response field seems missing, check `_names` before assuming the API omitted it. `from_dictionary`
and `additional_properties` both key off this map: unknown fields are exactly those whose API name is not
in `_names.values()`.

## `additional_properties`

A model generated with an additional-properties bag takes it as the **last** constructor argument and
fills it on deserialization with every field not in `_names`:

```python
additional_properties = {k: v for k, v in dictionary.items() if k not in cls._names.values()}
```

```python
model = {Model}({attr}='...', additional_properties={'x-custom': 1})
model.additional_properties.get('x-vendor-field')
```

A model with no such parameter **drops** unknown response fields silently.

## `_nullables` — only when the SDK generates it

**Check the model class first: many SDKs emit only `_names` and `_optionals`.** `_nullables` is
generated only for a model that has a nullable attribute.

An attribute that is both optional and listed in `_nullables` distinguishes "absent" from "explicitly
null": `from_dictionary` tests `"key" in dictionary` rather than truthiness for it, so an explicit
`null` in the payload arrives as `None` instead of collapsing into the absent case. Boolean and
model-typed optionals get that same key-presence test whether or not the model emits `_nullables`.

## Enum shape

```python
class {EnumType}(object):
    MEMBER_ONE = "member_one"     # string-backed
    MEMBER_TWO = "member_two"

    @classmethod
    def from_value(cls, value, default=None): ...
```

Integer-backed enums are identical with `int` values. Because a member *is* its value:

```python
{EnumType}.MEMBER_ONE == 'member_one'          # True
some_model.{enum_attr} == {EnumType}.MEMBER_ONE  # plain == works
```

`from_value` matches an `int` directly, or a `str` case-insensitively against both member names and
member values, returning `default` when nothing matches — useful for values arriving from config or user
input. Note it returns the **value**, not a member object.

There is no exhaustiveness checking and no unknown-value guard: a value the API adds later arrives as a
plain string or int your code has no member for.

## Date and date-time

| Wire format | Attribute type | Built with |
| --- | --- | --- |
| full-date | `datetime.date` | parsed by `dateutil.parser` in `from_dictionary` |
| RFC 3339 date-time | `APIHelper.RFC3339DateTime` | `APIHelper.apply_datetime_converter(value, APIHelper.RFC3339DateTime)` |
| HTTP (RFC 1123) date-time | `APIHelper.HttpDateTime` | same, with `APIHelper.HttpDateTime` |
| Unix timestamp | `APIHelper.UnixDateTime` | same, with `APIHelper.UnixDateTime` |

The model constructor applies the converter for you, so pass an ordinary `datetime.datetime`. Each
wrapper exposes `from_datetime(dt)` and `from_value(wire_value)` if you need to convert by hand; see
`doc/rfc3339-date-time.md`, `doc/http-date-time.md` and `doc/unix-date-time.md`.

## Union types (oneOf / anyOf)

- There is **no generated class** for a union — the value is one of the variants directly, and the
  parameter's type in the docstring reads `{VariantA} | {VariantB}`.
- The variant list lives in `doc/models/containers/<union-name>.md`, together with an initialization
  snippet per variant.
- Validation runs client-side inside the request builder, through
  `UnionTypeLookUp.get("<union-name>").validate(value)` in
  `paypalserversdk/utilities/union_type_lookup.py`. It fires **before** the HTTP request.
- Narrow a returned union with `isinstance`; for a discriminated union, the discriminator is an ordinary
  attribute on each variant, so branching on it works too.

## Files

`FileWrapper(file, content_type="application/octet-stream")` from
`paypalserversdk/utilities/file_wrapper.py` pairs an open binary stream with the content type to send.

## Notes

- Models define `__repr__` and `__str__`, so printing one shows every attribute — the quickest way to
  see what a response actually contained.
- Model classes are plain objects: no validation runs on the attributes you set, so a wrong type
  surfaces as a server-side error rather than a local one. Union parameters are the exception.
