## Project context

This repository contains a Ditsmod application.

Ditsmod is TypeScript Node.js framework based on decorators, modules, dependency injection, extensions, metadata reflection, and explicit module composition.

The framework requires Node.js >=v24.0.0.

In Ditsmod module decorators, metadata configuration (especially `imports`, `exports`, and `providersPer*` arrays) directly shapes the DI injector hierarchy. The transfer and visibility of this metadata are completely governed by the framework's Dependency Injection rules and behavior. For details on modules, see the [ditsmod-modules](skills/ditsmod-modules/SKILL.md) skill, and for DI behavior, see the [ditsmod-dependency-injection](skills/ditsmod-dependency-injection/SKILL.md) skill.

## Introduction to Ditsmod Packages

The core of the framework resides in `@ditsmod/core`, and all other Ditsmod packages depend on it. This dependency is specified via `peerDependencies` in each package's `package.json`.

The `@ditsmod/core` contains only the bare-minimum foundational functionality (DI, modules, extensions, etc.), which is insufficient to run a full web application. This is enough to run only `StandaloneApplication`, it does not include route-creating extensions, nor does it define concepts like controllers, guards, or interceptors. All these high-level web entities are provided by `@ditsmod/rest` and `@ditsmod/trpc`. But do not mix entities from `@ditsmod/rest` and `@ditsmod/trpc` in the same application, as they have different architectural styles. If a user is using a third-party package and you don't know what architectural style it is compatible with, look at its `peerDependencies` in the `package.json` file.

Ditsmod packages and `@ts-stack/*` packages are always published with their source files in the `src` folder, so agents can utilize them if needed.

## Code style

- Kebab-case for names of any files and directories.
- Camel-case for names of any classes or interfaces.
- The role of the class must be specified in the class name suffix and separated by a dot: `<file-name>.<role-name>.ts`. For example, `hello-world.module.ts`, `hello-world.controller.ts`, etc. The same rule applies to the class name — the class role must be present and go at the end: `HelloWorldModule`, `HelloWorldController`, etc.
- Decorator names start with lowercase.
- Prefer ESM syntax and do not introduce CommonJS unless explicitly required.
- In TypeScript files, imports should be grouped by the following types and in the following order:

  ```ts
  // 1. Built-in Node.js modules or non-Ditsmod external modules
  import path from 'node:path';
  import axios from 'axios';

  // 2. Ditsmod-external modules
  import { inject } from '@ditsmod/core';

  // 3. Subpath Imports / Alias imports / Local / Relative imports
  import { SomeService } from '#services/some-service';
  import { OtherService } from './other-service';
  ```

  Group imports only if there are more than 4 imports in the same file.

- Prefer use native Node.js subpath imports (`#di/*`, `#init/*`, etc.) as internal aliases. In `tsconfig.json` they resolve to `src/`; in `package.json` `imports` they resolve to `dist/`. Always maintain both in sync when adding new internal modules.

## User Interaction

- If the user asks a question, do not modify any code immediately. It is sufficient to answer the question first. Only modify the code when the user explicitly instructs to do so.

