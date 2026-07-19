---
name: ditsmod-project-setup
description: Guidance on using Ditsmod CLI (ditsmod, dm) for project bootstrapping and watch-running (start). Includes templates (rest, trpc), minimal single-file applications, and AI agent usage guidelines.
---

# Ditsmod Project Setup & CLI

This skill covers bootstrapping new Ditsmod applications, installing templates, configuring minimal single-file architectures, and using the `@ditsmod/cli` development helper.

---

## Installation & Bootstrapping

You can create a new Ditsmod application using `@ditsmod/cli` (via `npx` or local installation `ditsmod`/`dm`):

```bash
npx @ditsmod/cli new my-app [options]
# Or using the local alias:
dm new my-app [options]
```

> [!TIP]
> Run `npx @ditsmod/cli --help` or `dm --help` to discover all available commands, options, and templates.

### Options for `new`

- `-t, --template <name>`: Starter template to use:
  - `rest` (default)
  - `rest-monorepo`
  - `trpc-monorepo`
- `-m, --package-manager <name>`: Package manager to use (`npm`, `yarn`, `pnpm`). Default: `npm`.
- `--skip-install`: Do not automatically run package installation after bootstrapping.
- `--skip-git`: Do not initialize a new clean Git repository with an initial commit.

### Bootstrapping Examples

```bash
# Create a REST application using Yarn
npx @ditsmod/cli new my-rest-api -m yarn

# Create a REST monorepo application
npx @ditsmod/cli new my-rest-monorepo --template rest-monorepo

# Create a tRPC monorepo without installing packages
npx @ditsmod/cli new my-trpc-app -t trpc-monorepo --skip-install
```

---

## Ditsmod CLI (`@ditsmod/cli`)

Binary aliases `ditsmod` and `dm` are available when the package is installed locally:

```bash
npm i -D @ditsmod/cli
```

You can use the `dm` alias to run CLI commands:

```bash
dm start [entryFile] [options]
```

### The `start` Command

Runs the Ditsmod application in development mode with incremental TypeScript watch compilation (supporting TypeScript Project References for monorepos), automatic asset copying, and graceful process restarts.

```bash
npx @ditsmod/cli start [entryFile] [options]
# Or using the local alias:
dm start [entryFile] [options]
```

#### Options for `start`

- `-p, --project <path>`: Path to TypeScript config file or project directory (supporting TS Project References). Default: `tsconfig.build.json`.
- `-e, --exec <binary>`: Binary to execute the entry file (e.g., `node`). Default: `node`.
- `-d, --debug [hostport]`: Run Node.js in debug mode with the `--inspect` flag.
- `--env-file <paths...>`: Environment file(s) to load into `process.env` (Node.js >= v20).
- `--entry-file <file>`: Relative path to the compiled JavaScript entry file. Default: `dist/main.js`.
- `--watch-assets <globs...>`: Non-TypeScript asset globs to watch and copy to `dist/` (e.g. `"src/**/*.json"`).
- `--preserve-watch-output`: Do not clear the terminal screen between compilation cycles.

#### Examples

```bash
# Start application with custom entry file and inspect debugger enabled on port 9229
npx @ditsmod/cli start tmp.ts -d 9229

# Start with environment file and watch json assets
npx @ditsmod/cli start --env-file .env.local --watch-assets "src/**/*.json"
```

### CLI Programmatic API

`@ditsmod/cli` exports classes and command helper functions:

```ts
import { WatchCompiler, ProcessManager, AssetWatcher, startCommand, newCommand } from '@ditsmod/cli';
```

---

## AI Agent Usage Rules

> [!WARNING]
> `@ditsmod/cli start` is designed for **human developers** to run interactively in their terminal.
> **AI agents must NOT start long-running background compilation/watch processes (`ditsmod start`)** during normal coding or editing cycles unless explicitly requested by the user, as this can lead to memory leaks, orphaned background processes, and blocked ports.
>
> If an agent needs to verify compilation or run tests, it should use standard one-off build/test scripts (like `npm run build` or `npm run test`) instead of interactive watch mode.

---

## Minimal REST Application

Use this minimal setup instead of cloning a full starter repository when:

- The user requests a minimal, single-file, or lightweight demonstration/POC of Ditsmod.
- The project structure does not require a full starter template architecture.
- For quick debugging or unit testing of a simple Ditsmod REST application.

### Prerequisites

To run this minimal application, ensure you have:

1. Node.js >= v24.0.0.
2. The required Ditsmod and TypeScript packages installed:
   ```bash
   npm i @ditsmod/core @ditsmod/rest
   npm i -D typescript @types/node
   ```
3. The following compiler options enabled in `tsconfig.json`:
   ```json
   {
     "compilerOptions": {
       "experimentalDecorators": true,
       "emitDecoratorMetadata": true
     }
   }
   ```

### Code Example

You can save this code in a file (e.g., `index.ts`) and run it:

```ts
import { controller, route, restRootModule, RestApplication } from '@ditsmod/rest';

@controller()
class ExampleController {
  @route('GET', 'hello')
  tellHello() {
    return 'Hello, World!';
  }
}

@restRootModule({ controllers: [ExampleController] })
class AppModule {}

const app = await RestApplication.create(AppModule);
app.server.listen(3000, '0.0.0.0');
```
