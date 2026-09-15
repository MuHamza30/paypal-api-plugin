# R002 — round-02 attempt 1 confounded by a wrong package identity (my error)

**Both round-02 A3 runs from the first attempt are discarded as a comparison.** The corpus fixes were
present; so was an unintended second change.

## What went wrong

Round-01's pack came from `apimatic plugin generate`, which read the real `plugin-config.json` — so its
identity tokens were correct: `@paypal/paypal-server-sdk`, version `2.0.0`.

Round-02's pack was rendered with `APIMatic.TestConsole`, whose
`Settings/globalSdkPluginConfig.json` ships `"EnableSdkPluginConfig": false`. With no `SdkPackage`
supplied, `{{PACKAGE_NAME}}`, `{{PACKAGE_VERSION}}`, `{{PACKAGE_SOURCE}}` and `{{INSTALL_COMMAND}}` all
fell back to build-derived values. The improved pack therefore said:

> `| npm package id | paypal-server-sdklib (version 2.29.0) |`
> `| Install | npm install file:../path/to/paypal-server-sdklib |`
> "This build was not configured for publishing … the SDK is the directory you were handed."

The builder caught it immediately and reported it as the pack's top defect:

> "**every import line in every skill in the pack reads `from 'paypal-server-sdklib'`** and had to be
> silently rewritten by me at every use."

So the two rounds differed in **two** ways — the fixes (intended) and the package identity (not). A
handicap that large swamps the signal, and any comparison drawn from it would be worthless.

## I had already seen this and failed to act on it

While verifying F004 I rendered the TS pack, saw `node_modules/paypal-server-sdklib/`, and noted it as
"a test-console artifact, not a corpus bug". That was correct — and I then built the round-02 pack with
the same tool without carrying the observation forward. The note was right; the follow-through was not.

## Why the documented override did not save me

`SdkPluginConfigSettings.FromConfig` is called with the tracked path, and only consults the untracked
`Settings/Local/` copy when the tracked file is **missing**:

```csharp
if (!File.Exists(filePath))          // Settings/globalSdkPluginConfig.json exists
{
    var localSelector = Path.Combine("Settings", "Local", "globalSdkPluginConfig.json");
    ...                               // never reached
}
```

The class doc-comment says the default is "overridable per developer by an untracked copy under
Settings/Local/", and `.gitignore` carries an entry for that directory — so the intent is explicit and
the logic does not implement it. **That is a test-console bug, not a corpus one**, and worth an issue
on its own: every developer who follows the documented override gets silently ignored defaults.

Worked around by swapping the tracked file, rendering, and restoring it. Tree verified clean afterwards.

## Corrected pack

| pack | identity | matches round-01? |
| --- | --- | --- |
| csharp | `PayPalServerSDK` 2.0.0 | yes |
| typescript | `@paypal/paypal-server-sdk` 2.0.0 | yes |
| python | `pip install paypal-server-sdk` | yes |

Round-02 is being re-run against this. The brief stays byte-identical to round-01, and the pack is now
the only variable it was always supposed to be.

## What survives from the discarded attempt

One observation that does not depend on identity, and it is a real corpus defect:

**`typescript-getting-started` gives `BaseApi` in `src/controllers/baseApi.ts` as the base-controller
example; this SDK generates `BaseController` in `baseController.ts`.** The skill does tell the reader to
read the real names off the `export class` lines, and the builder did, so it cost nothing — but the
example is wrong for this build. Tracked separately as **F013**.
