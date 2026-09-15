# F007 — a REJECTED defect, and the real generic fix hiding behind it

**Reported as:** "`csharp-models` recommends `ApiHelper.JsonSerialize`, which does not exist in
`PayPalServerSDK` 2.0.0 — would have been a compile failure on every route."
Source: round-01 csharp debrief §D1, tagged `[MEASURED]`.

**Verdict: REJECTED. The corpus is right and the reporting agent was wrong.**
Derived fix below is `accept-corpus`, csharp only.

## Why it was rejected `[MEASURED]`

The agent ran `grep -n "public static string JsonSerialize" Utilities/ApiHelper.cs`, got nothing, and
concluded the member was absent. But the generated file is fourteen lines:

```csharp
public class ApiHelper : CoreHelper { }
```

An empty subclass. The member is **inherited** from the runtime package:
`apimatic/core-lib-csharp`, `APIMatic.Core/Utilities/CoreHelper.cs` —
`public static string JsonSerialize(object obj, JsonConverter converter = null)`.

C# resolves an inherited public static member through the derived type name, so
`ApiHelper.JsonSerialize(...)` compiles. The skill's claim is correct, and it is correct in all six
places it appears across four csharp files.

`Corpus/CLAUDE.md` predicted this failure exactly:

> **Ground claims in the generator, then in the runtime package — not in a sample SDK.** Reading
> generated output cannot settle a claim whose logic lives in `apimatic-core` / `core-lib-php` /
> `io.apimatic:core`. Several "defects" reported against these packs were the pack being right and the
> auditor reading only the generated controller.

**Cost of the wrong conclusion was real:** the agent wrote `Infrastructure/WireValue.cs`, doing
reflection over `EnumMemberAttribute`, to replace a one-line call that would have worked.

## The genuine, generic fix this exposes

The skill says `ApiHelper` is in `{{ROOT_NAMESPACE}}.Utilities`, which is true — and a reader who opens
that file finds an **empty class** and reasonably concludes the method is not there. One agent did
exactly that and paid for it.

So the fix is not to the claim but to its navigability: say that the generated helper is a thin
subclass whose members come from the runtime package, and that the file itself is empty by design.
That is true of every APIMatic C# SDK, names no provider, and forecloses this specific wrong turn.

Consider whether the sibling packs have the same shape (php's `core-lib-php`, java's
`io.apimatic:core`) before touching them — the base class and its name differ per language, and this
must not be copied.

## Process lesson — worth more than the fix

This is the first finding the loop produced that was **confidently wrong**, tagged `[MEASURED]`, and
would have caused four files of correct prose to be "corrected" into workarounds.

What caught it was the corroboration rule: a single-session finding needs an independent check against
the generator **and the runtime** before acceptance. Grepping the generated SDK is not that check.

**Add to triage:** before accepting any "this member does not exist" defect, resolve the type's base
class. An APIMatic SDK's `Utilities/` is largely empty subclasses of runtime types.
