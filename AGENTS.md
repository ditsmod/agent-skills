## Project context

This repository contains a Ditsmod application.

Ditsmod is TypeScript Node.js framework based on decorators, modules, dependency injection, extensions, metadata reflection, and explicit module composition.

## Introduction to Ditsmod Packages

The core of the framework resides in `@ditsmod/core`, and all other Ditsmod packages depend on it. This dependency is specified via `peerDependencies` in each package's `package.json`.

The `@ditsmod/core` contains only the bare-minimum foundational functionality (DI, modules, extensions, etc.), which is insufficient to run a full web application. This is enough to run only `StandaloneApplication`, it does not include route-creating extensions, nor does it define concepts like controllers, guards, or interceptors. All these high-level web entities are provided by `@ditsmod/rest` and `@ditsmod/trpc`. But do not mix entities from `@ditsmod/rest` and `@ditsmod/trpc` in the same application, as they have different architectural styles. If a user is using a third-party package and you don't know what architectural style it is compatible with, look at its `peerDependencies` in the `package.json` file.

## Code style

* Kebab-case for names of any files and directories.
* Camel-case for names of any classes or interfaces.
* The role of the class must be specified in the class name suffix and separated by a dot: `<file-name>.<role-name>.ts`. For example, `hello-world.module.ts`, `hello-world.controller.ts`, etc. The same rule applies to the class name — the class role must be present and go at the end: `HelloWorldModule`, `HelloWorldController`, etc.
* Decorator names start with lowercase.
* Prefer ESM syntax and do not introduce CommonJS unless explicitly required.
* Prefer native Node.js "Subpath Imports" (starting with `#`) over TypeScript `paths` (which are only defined in `tsconfig.json`). Subpath imports are runtime-resolved and more portable. Aliases starting with `#` must resolve to the compiled JavaScript directory (usually `dist`) in the `package.json` `imports` field. Additionally, these same aliases must be defined in TypeScript's `paths` inside `tsconfig.json`, but there they must resolve to the source TypeScript directory (`src`). This ensures that the runtime uses the compiled JavaScript, while the TypeScript compiler and IDE use the original source files for type-checking and autocompletion. For example:
  ```json
  // package.json
  {
    "imports": {
      "#services/*": "./dist/services/*"
    }
  }
  ```
  ```json
  // tsconfig.json
  "compilerOptions": {
    "paths": {
      "#services/*": ["./src/services/*"],
    },
    // ...
  },
  ```

* In TypeScript files, imports should be grouped by the following types and in the following order:

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
  Group imports only if there are more than two imports in the same file.

## Ditsmod rules

When working with Ditsmod code:

- Do not bypass dependency injection by creating service instances manually.

## Testing

Tests are part of the architecture.

When changing behavior:

* update existing tests;
* add regression tests.

## Communication

When explaining changes:

* mention affected modules;
* explain architectural decisions;
* highlight possible migration concerns.
