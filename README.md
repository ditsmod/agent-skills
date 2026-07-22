# Ditsmod Agent Skills

A collection of [Agent Skills](https://github.com/vercel-labs/skills) for AI coding assistants (such as Claude Code, Gemini CLI, Cursor, Copilot, Codex, and others) working with [Ditsmod](https://ditsmod.github.io/en/).

[Ditsmod](https://ditsmod.github.io/en/) is a TypeScript Node.js web framework built around decorators, modules, dependency injection, extensions, metadata reflection, and explicit module composition.

## 🚀 Installation

Install all available skills at once into your project using the `skills` CLI:

```bash
npx skills add ditsmod/agent-skills --skill '*' -y
```

Or install specific skills individually:

```bash
npx skills add ditsmod/agent-skills --skill ditsmod-core-architecture -y
```

## 📦 Available Skills

| Skill | Description |
| --- | --- |
| **[ditsmod-core-architecture](skills/ditsmod-core-architecture/SKILL.md)** | Core pillars of Ditsmod: Modules, Dependency Injection (DI) hierarchy, provider scopes (`providersPerApp`, `providersPerMod`, `providersPerReq`, etc.), and custom decorators. |
| **[ditsmod-project-setup](skills/ditsmod-project-setup/SKILL.md)** | Bootstrapping projects with `@ditsmod/cli` (`dm`), project templates (REST, tRPC), minimal single-file apps, and development workflows. |
| **[ditsmod-rest](skills/ditsmod-rest/SKILL.md)** | HTTP request lifecycle in `@ditsmod/rest`, sequence of execution (interceptors, guards, routing, controllers), and custom request dispatchers. |
| **[ditsmod-extensions](skills/ditsmod-extensions/SKILL.md)** | Building, registering, ordering, exporting, grouping, and overriding Ditsmod extensions (stage 1–3 initialization hooks). |
| **[ditsmod-schedule](skills/ditsmod-schedule/SKILL.md)** | Task scheduling via `@ditsmod/schedule` (`@cron`, `@interval`, `@timeout`), `SchedulerRegistry`, provider scopes, and graceful shutdown patterns. |

## 💡 How It Works

Agent skills provide authoritative guidance, architectural rules, and code patterns directly to AI assistants. When an AI agent encounters a Ditsmod codebase with these skills enabled, it leverages them to produce idiomatic, type-safe, and architecturally accurate code.

## 📜 License

[MIT](LICENSE)
