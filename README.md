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

The plugin is published as a **Claude Code plugin marketplace** at
[`MuHamza30/paypal-api-plugin`](https://github.com/MuHamza30/paypal-api-plugin). In Claude Code:

```
/plugin marketplace add MuHamza30/paypal-api-plugin
/plugin install paypal-api@paypal-api-plugin
```

Restart Claude Code (or run `/plugin`) and the 48 skills become available.

### Other agents

Codex, from the CLI:

```
codex plugin marketplace add https://github.com/MuHamza30/paypal-api-plugin
codex plugin add paypal-api@paypal-api-plugin
```

Cursor and anything else that reads a local plugin directory: clone the repo and point the
agent at the clone.

```
git clone https://github.com/MuHamza30/paypal-api-plugin.git
```

### Updating

```
/plugin marketplace update paypal-api-plugin
```

## Usage

Ask a usage question (e.g. *"how do I authenticate this SDK with an API key?"*) and the relevant
skill loads automatically, or invoke one by name — `/paypal-api:python-getting-started`.
