# F006 — the OAuth token-provider sample poisons the client for the process lifetime on one transient failure

**Severity: HIGH (breaks production).** **Disposition: needs a decision — likely `reject-generator`
(issue) + a corpus warning.** Scope: typescript confirmed; **the other five are unverified**.
Source: round-01 TypeScript debrief §D2, `[MEASURED]` by reading the runtime adapter.

## The sample the skill gives

`typescript-authentication`, "OAuth 2.0 — persisting and restoring a token":

```ts
oAuthTokenProvider: async (lastOAuthToken, authManager) =>
  (await loadTokenFromDatabase()) ?? authManager.fetchToken()
```

## What the runtime does with it

`@apimatic/oauth-adapters` assigns the **promise** to a module-level `lastOAuthToken` and re-awaits it
on subsequent requests. If the provider's promise **rejects** — database unreachable, token endpoint
down — the rejected promise is cached and re-awaited at `case 0` of every later request, and nothing
ever reassigns it. **One transient failure disables the client for the lifetime of the process.**

The skill documents only *when* the provider is invoked ("whenever the cached token is missing or
expired"). It says nothing about rejection semantics, and the sample it gives is one `await` away from
the failure mode.

## Cost in this run: none, but only by luck

The agent read `oauthAuthenticationAdapter.js` and designed around it — recording the cause and
returning an already-expired sentinel instead of rejecting. It found this **during the debrief**, not
during the build. A reader who follows the sample as written gets the bug.

## Disposition is genuinely open

Two readings, and they lead to different places:

1. **Runtime defect** — caching a rejected promise with no reset is a bug in `@apimatic/oauth-adapters`.
   `CLAUDE.md` is explicit that the corpus must not paper over a generator or runtime defect: file an
   issue, and at most describe what actually happens.
2. **Corpus defect** — the skill ships a sample that triggers it. Even if the runtime is fixed, a
   sample whose failure mode is unstated is a corpus problem.

Both are probably true. The corpus half is the part in bounds: state that the provider's rejection is
cached, and that the provider should resolve to an expired token rather than reject.

## Before acting — two checks this finding has NOT had

1. **Is the same true in the other five languages?** Each has its own OAuth manager. `CLAUDE.md`
   already records two *deliberate* divergences that must not be "corrected":
   [#5445](https://github.com/apimatic/apimatic-codegen/issues/5445) (PHP OAuth manager references a
   class never generated) and [#5498](https://github.com/apimatic/apimatic-codegen/issues/5498) (Ruby
   OAuth token-update hook never fires). **Check both before writing a line of OAuth prose.**
2. **Single-session finding.** Per the corroboration rule this needs an independent check against the
   runtime source before it is accepted — the `[MEASURED]` tag covers the adapter read, not the
   conclusion that no code path reassigns.
