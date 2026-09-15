# F013 — REJECTED (minor): the `BaseApi` example is an illustration, and the rule around it is correct

**Reported as:** "`typescript-getting-started`, Package layout, base-controller naming. It gives `Base`
+ postfix with the example `BaseApi` in `src/controllers/baseApi.ts`. In 2.0.0 it is `BaseController`
in `src/controllers/baseController.ts`."
Source: round-02 attempt-1 typescript §D2, `[MEASURED]`, and self-graded "Minor".

**Verdict: no change. The pack states the rule correctly.**

## What the pack says

> "Its name is `Base` + the SDK's controller postfix (e.g. `BaseApi` in `src/controllers/baseApi.ts`
> when the postfix is `Api`); the postfix is fixed at generation time, so read the real class names off
> the `export class` lines in `src/controllers/`."

## What the generator does

`TS/Templates/Generic/Controllers/BaseControllerTemplateViewModel.cs:24` emits
`export class {{projectSettings.CodeUtilities.GetBaseControllerName()}}`. The name is derived, and for a
build whose postfix is `Controller` it is `BaseController` — confirmed in the published SDK, whose
`src/controllers/` holds `baseController.ts`.

So **`Base` + postfix is right**, the example is explicitly conditioned (*"when the postfix is `Api`"*),
and the very next clause tells the reader to read the real names off the source. The reporter did
exactly that and recorded the cost as nil.

## Why I am not "improving" it anyway

`CLAUDE.md`'s neighbouring rule — *never name a concrete `Environment` member as an illustration,
because an example member reads as a fact* — is the closest argument for changing it. But that rule
exists for members the corpus **asserts**; this one is a worked example of a stated rule, guarded by its
own condition and followed by an instruction to check.

There is also no token to resolve it with: TypeScript's provider supplies only `{{PACKAGE_NAME}}`, so
there is no `{{CONTROLLER_SUFFIX}}` equivalent to substitute the real postfix the way C# and Python can.
Replacing the example with a bare placeholder would trade a concrete illustration for a vaguer one and
gain nothing measurable.

Changing it would be scope creep against my own triage rubric: the claim is true, the cost was zero,
and no rule is broken.

## Worth noting about the source of this finding

It came from the **confounded** round-02 attempt (R002), where the pack carried a wrong package
identity. This particular observation does not depend on identity, which is why it was extracted rather
than discarded with the rest — but it is the only thing from that attempt that survived scrutiny, and
it did not survive it far.
