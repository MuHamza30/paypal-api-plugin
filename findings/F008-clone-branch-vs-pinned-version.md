# F008 — the skills tell you to clone the default branch while pinning a different package version

**Severity: MEDIUM-HIGH (silently reads the wrong release).** **Disposition: `accept-corpus`, likely
all six — the `{{SDK_SOURCE}}` clone instruction is shared.** Source: round-01 csharp debrief §D2,
`[MEASURED]` by the agent; not yet independently re-verified.

## What happened

The csharp pack pins `Version="2.0.0"` in its install section, and separately instructs a clone of the
repository, taking the branch explicitly — which resolves to `main`.

The agent followed both literally and got a checkout whose `.csproj` reads `<Version>2.4.0.0</Version>`
— **a different release from the one it was compiling against.** It noticed, ran
`git ls-remote --tags`, found tag `2.0.0`, and re-cloned at the tag.

## Why this matters more than a version string

The whole premise of the SDK-source section is that **the source is authoritative for signatures**.
The skills repeatedly say to confirm every name and signature against it. If the clone is a different
release from the package, then following the instruction as written means grounding every contract
fact in code you are not compiling against — and the more faithfully a reader follows the skill, the
more wrong they can be.

The agent did **not** establish that 2.4.0 and 2.0.0 differ semantically for its five operations
`[UNKNOWN]`; the only difference it observed was cosmetic. So the *realised* cost here was one wasted
re-clone. The *potential* cost is a signature read from the wrong release with nothing to flag it.

## The generic fix

The clone instruction should check out the release matching the pinned package version, and say what
to do when no such tag exists. This is API-agnostic: it is true wherever a plugin pins a package
version and the repository has moved on, which is the normal state of a published SDK.

**Check where this actually lives before editing.** The clone block is rendered by `{{SDK_SOURCE}}`
(`Skills/SdkPackage.cs`, `DescribeSdkSource()`), not written per pack — so the fix is likely one
change in the token renderer rather than six prose edits, and the token already knows both the branch
and the package version. That makes it a **token-provider change**, in bounds, with the usual
conditions: wiring lands with the corpus edit, and a test proves the rendered text changed.

## Not yet verified by me

The agent's `[MEASURED]` covers its own two clones. I have not confirmed the `{{SDK_SOURCE}}` renderer
emits a bare `--branch` with no tag option, nor whether any pack overrides it. Do that first.
