---
name: ditsmod-modules
description: 'Ditsmod modules: rootModule, featureModule, restRootModule, restModule, trpcRootModule, trpcModule decorators; imports vs appends vs exports; provider visibility (providersPerApp/Mod/Rou/Req); getTokens(); ModuleWithParams; re-exporting; resolvedCollisionPer* for token collision resolution. Use when assembling modules, wiring imports/exports, setting route prefixes, configuring provider scope, or fixing provider collision errors.'
---

# Ditsmod Modules

## Choose The Module Type

Ditsmod applications have two module roles:

- **Root module** — exactly one per application; entry point for DI composition. Recommended class name: `AppModule`.
- **Feature module** — encapsulates a cohesive set of providers, controllers, or extensions. Reusable and importable.

`@ditsmod/core` provides `rootModule` and `featureModule` decorators. Both accept the same base metadata properties:

```ts
import { rootModule, featureModule } from '@ditsmod/core';

@featureModule({
  imports: [], // Modules whose exports this module consumes
  providersPerApp: [], // Providers registered at application level
  providersPerMod: [], // Providers registered at module level
  providersPerRou: [], // Providers registered at route level
  providersPerReq: [], // Providers registered per HTTP request
  exports: [], // Tokens or modules exposed to importers
  extensions: [], // Extensions to run
  extensionsMeta: {}, // Keyed data consumed by extensions
  resolvedCollisionPerMod: [], // Collision resolution at module level
  resolvedCollisionPerRou: [], // Collision resolution at route level
  resolvedCollisionPerReq: [], // Collision resolution at request level
})
export class SomeModule {}

@rootModule({
  // All featureModule properties, plus:
  resolvedCollisionPerApp: [], // Collision resolution at app level
})
export class AppModule {}
```

`@ditsmod/rest` and `@ditsmod/trpc` provide their own decorators with extended metadata:

```ts
import { restRootModule, restModule } from '@ditsmod/rest';

@restModule({
  // The same list of properties as the 'restRootModule' decorator, except 'resolvedCollisionPerApp'
})
export class SomeModule {}

@restRootModule({
  // All rootModule properties, plus:
  appends: [], // Attach a module's controllers without consuming its exports
  controllers: [], // Register controllers directly on the root module
})
export class AppModule {}
```

```ts
import { trpcRootModule, trpcModule } from '@ditsmod/trpc';

@trpcModule({
  // The same list of properties as the 'trpcRootModule' decorator, except 'resolvedCollisionPerApp'
})
export class SomeModule {}

@trpcRootModule({
  // All rootModule properties, plus:
  controllers: [], // Register tRPC controllers directly on the root module
})
export class AppModule {}
```

> **Do not mix** `@ditsmod/rest` and `@ditsmod/trpc` entities in the same application. Check a package's `peerDependencies` to confirm architectural style compatibility.

## Import, Append, Export

**`imports`** — use when the current module needs exported providers or extensions from another module:

```ts
@restModule({
  imports: [AuthModule],
})
export class UsersModule {}
```

**`imports` with `{ path, module }`** (`@ditsmod/rest`) — additionally mounts the imported module's controllers under a route prefix. The `path` property activates controller mounting even when set to an empty string. Use `absolutePath` instead of `path` when the prefix must be absolute (the two are mutually exclusive). Add `guards` to protect all routes in the imported module:

```ts
@restModule({
  imports: [
    { path: 'users', module: UsersModule },
    { path: 'admin', module: AdminModule, guards: [AuthGuard] },
    { absolutePath: 'health', module: HealthModule }, // absolute path, no prefix stacking
  ],
})
export class AppFeatureModule {}
```

**`appends`** — use when you only need to attach another module's controllers under the current module's route prefix without consuming its exported providers or extensions. Only append modules that have controllers:

```ts
@restRootModule({
  appends: [{ module: AdminModule, path: 'admin' }],
})
export class AppModule {}
```

## Export Providers By Token

Export provider tokens, not provider objects. `providersPerApp` providers are globally available and do not need exporting.

Use `getTokens()` when the provider array contains object-form providers (e.g., `{ token, useClass, ... }`), because their tokens cannot be statically extracted otherwise:

```ts
import { restModule, getTokens } from '@ditsmod/rest';
import { authProviders } from './auth.providers.js';

@restModule({
  providersPerReq: [...authProviders],
  exports: [...getTokens(authProviders)],
})
export class AuthModule {}
```

Export only what consumer modules inject directly. Internal implementation services do not need to be exported unless consumers inject them. Do not export controllers — exports apply only to providers and modules.

## Re-Export Modules

To pass through another module's exports to importers of the current module, both import and export it:

```ts
@restModule({
  imports: [AuthModule],
  exports: [AuthModule],
})
export class SecurityModule {}
```

When re-exporting a `ModuleWithParams` object, export the **same object reference** that was imported:

```ts
const usersWithPath = { module: UsersModule, path: 'users' };

@restModule({
  imports: [usersWithPath],
  exports: [usersWithPath], // must be the identical object, not a new literal
})
export class ApiModule {}
```

## Module Parameters

Use a static factory method when a module is commonly imported with configuration:

```ts
import { type ModuleWithParams } from '@ditsmod/core';

export class UsersModule {
  static withPrefix(path: string): ModuleWithParams<UsersModule> {
    return { module: this, path };
  }
}
```

```ts
@restModule({
  imports: [UsersModule.withPrefix('users')],
})
export class ApiModule {}
```

Pass extension-specific data via `extensionsMeta` inside `ModuleWithParams`. Keep each extension's data under a single dedicated key.

## Provider Visibility And Lifetime

Provider registration level determines scope and sharing:

| Level   | Array             | Scope                  | Shared across modules?            |
| ------- | ----------------- | ---------------------- | --------------------------------- |
| App     | `providersPerApp` | Entire app             | Yes — singleton, no export needed |
| Module  | `providersPerMod` | Module + its importers | Only if exported                  |
| Route   | `providersPerRou` | Single route subtree   | Only if exported                  |
| Request | `providersPerReq` | Single HTTP request    | Only if exported                  |

Imported provider classes are registered — not their instances. If the same provider class is declared at mod/rou/req level in two separate modules, each module gets its own instance. Use `providersPerApp` when a true singleton must be shared across all modules.

## Collision Handling

A collision occurs when two imported modules export non-identical providers under the same token. Resolve it in the module that imports both:

```ts
@restRootModule({
  imports: [DefaultAuthModule, CustomAuthModule],
  resolvedCollisionPerMod: [[AuthService, CustomAuthModule]],
})
export class AppModule {}
```

Match the `resolvedCollisionPer*` array to the provider scope level named in the collision error. If the collision originates from modules re-exported by a third-party package's root module, remove the conflicting re-exported module from the package root and import it explicitly where needed.

## Common Mistakes

- **Exporting controller classes** — controllers are not providers; only export injectable service tokens and modules.
- **Exporting a new `ModuleWithParams` literal on re-export** — always reuse the same object reference that was passed to `imports`.
- **Using `appends` for provider consumption** — `appends` only mounts controllers; it does not make the appended module's providers available.
- **Expecting cross-module instance sharing at mod/rou/req level** — each module that declares a provider gets its own instance; use `providersPerApp` for true singletons.
- **Omitting `path` in `imports` when controllers must be mounted** — without `path`, controllers from the imported module are not mounted even if the module has them.

## Checklist

1. Keep each feature module narrowly specialized.
2. Use `imports` for provider/extension consumption; use `appends` for route attachment only.
3. Export tokens or modules — not provider objects.
4. Use `getTokens()` when exporting tokens from an array that contains object-form providers.
5. Use `ModuleWithParams` static methods for reusable parameterized imports.
6. When re-exporting `ModuleWithParams`, export the same object reference that was imported.
7. Resolve provider token collisions explicitly via `resolvedCollisionPer*` instead of relying on import order.

## Further Reading

For full metadata type shapes, `getTokens()` internals, route prefix composition rules, provider scope decision guide, `extensionsMeta` usage patterns, and collision error message interpretation, see [references/REFERENCE.md](references/REFERENCE.md).
