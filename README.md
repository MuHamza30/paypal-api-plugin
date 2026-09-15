# PayPal Server SDK SDK Plugin

A plugin whose skills teach a coding agent to install and use the APIMatic-generated **PayPal Server SDK SDK**, in csharp, java, php, Python, ruby, TypeScript. Every SDK fact the skills state is grounded in the SDK's own source and generated documentation, not in what a model remembers about this API.

## What's inside

One skill set per language. The entry point is that language's getting-started skill, which carries what is specific to this SDK; the rest are API-agnostic and describe how to use any SDK the same generator produces.

| Language | Skill prefix | Skills |
| --- | --- | --- |
| csharp | `csharp-` | `csharp-authentication`, `csharp-calling-endpoints`, `csharp-client-initialization`, `csharp-configuration-resilience`, `csharp-error-handling`, `csharp-getting-started`, `csharp-models`, `csharp-testing` |
| java | `java-` | `java-authentication`, `java-calling-endpoints`, `java-client-initialization`, `java-configuration-resilience`, `java-error-handling`, `java-getting-started`, `java-models`, `java-testing` |
| php | `php-` | `php-authentication`, `php-calling-endpoints`, `php-client-initialization`, `php-configuration-resilience`, `php-error-handling`, `php-getting-started`, `php-models`, `php-testing` |
| Python | `python-` | `python-authentication`, `python-calling-endpoints`, `python-client-initialization`, `python-configuration-resilience`, `python-error-handling`, `python-getting-started`, `python-models`, `python-testing` |
| ruby | `ruby-` | `ruby-authentication`, `ruby-calling-endpoints`, `ruby-client-initialization`, `ruby-configuration-resilience`, `ruby-error-handling`, `ruby-getting-started`, `ruby-models`, `ruby-testing` |
| TypeScript | `typescript-` | `typescript-authentication`, `typescript-calling-endpoints`, `typescript-client-initialization`, `typescript-configuration-resilience`, `typescript-error-handling`, `typescript-getting-started`, `typescript-models`, `typescript-testing` |

## Install

This plugin was **generated for you, not published to a shared plugin marketplace** — so you install it from a local path rather than by name. Point your coding agent at the directory holding this plugin, then install it:

```
/plugin marketplace add <path to the directory containing this plugin>
/plugin install paypal-api
```

In Codex the same two steps are CLI commands:

```
codex plugin marketplace add <path to the directory containing this plugin>
codex plugin add paypal-api@<marketplace>
```

`<marketplace>` is the name the directory you just added registers under — `codex plugin marketplace list` prints it.

Once your team publishes it to a plugin marketplace, add that marketplace in place of the local path and install `paypal-api` from it by name.

Then ask a usage question (e.g. *"how do I authenticate this SDK with an API key?"*) to trigger the relevant skill.

