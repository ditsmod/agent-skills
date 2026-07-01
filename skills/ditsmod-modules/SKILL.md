---
name: ditsmod-modules
description: Practical guidance for creating, importing, appending, exporting, re-exporting, parameterizing, and troubleshooting Ditsmod modules in applications. Use when a Ditsmod app developer works with root modules, feature modules, restRootModule, restModule, imports, appends, exports, ModuleWithParams, route prefixes, provider visibility, getTokens, and provider collision resolution.
---

# Ditsmod Modules

Use this skill for application code that assembles Ditsmod modules. Keep the focus on clean module boundaries, explicit imports/exports, and provider visibility.

## Choose The Module Type

Ditsmod applications can have two types of modules:

- Root module. Other modules are imports to the root module; it is the only one for the entire application, and its class is recommended to be named `AppModule`.
- Feature module. Use feature modules for cohesive application features or reusable packages.

`@ditsmod/core` provides `rootModule` and `featureModule` decorators, both of which accept a basic set of properties in their metadata configuration:

```ts
import { rootModule, featureModule } from '@ditsmod/core';

@featureModule({
  // The same list of properties as the root module, except "resolvedCollisionPerApp"
})
export class SomeModule {}

@rootModule({
  imports: [], // Imported modules
  providersPerApp: [], // Providers at the application level
  providersPerMod: [], //         ...at the module level
  providersPerRou: [], //         ...at the route level
  providersPerReq: [], //         ...at the HTTP request level
  exports: [], // Exported modules and providers from the current module
  extensions: [], // Extensions
  extensionsMeta: {}, // Data for extensions
  resolvedCollisionPerApp: [], // Resolution of imported class collisions at the application level
  resolvedCollisionPerMod: [], //                                   ...at the module level
  resolvedCollisionPerRou: [], //                                   ...at the route level
  resolvedCollisionPerReq: [], //                                   ...at the HTTP request level
})
export class AppModule {}
```

`@ditsmod/rest` and `@ditsmod/trpc` also have root and feature modules, but their metadata accepts an extended type:

```ts
import { restRootModule, restModule } from '@ditsmod/rest';

@restModule({
  // The same list of properties as the "trpcRootModule" decorator, except "resolvedCollisionPerApp"
})
export class SomeModule {}

@restRootModule({
  // Same list of properties from "rootModule", plus next:
  appends: [],
  controllers: []
})
export class SomeModule {}
```

```ts
import { trpcRootModule, trpcModule } from '@ditsmod/trpc';

@trpcModule({
  // The same list of properties as the "trpcRootModule" decorator, except "resolvedCollisionPerApp"
})
export class SomeModule {}

@trpcRootModule({
  // Same list of properties from "rootModule", plus next:
  controllers: [] // new property
})
export class SomeModule {}
```

## Import, Append, Export

Use `imports` when the current module needs exported providers or extensions from another module:

```ts
@restModule({
  imports: [AuthModule],
})
export class UsersModule {}
```

Use `imports` with `{ module, path }` when controllers from the imported module must also be mounted. The presence of `path` means controllers are considered, even when `path` is an empty string:

```ts
@restModule({
  imports: [{ path: 'users', module: UsersModule }],
})
export class AppFeatureModule {}
```

Use `appends` when you only need to attach another module's controllers under the current module's path prefix and do not need its exported providers or extensions:

```ts
@restModule({
  appends: [{ module: AdminModule, path: 'admin' }],
})
export class ApiModule {}
```

Only append modules that have controllers.

## Export Providers By Token

Export provider tokens, not provider objects. Providers from `providersPerApp` are app-level and do not need exporting for app-wide visibility.

```ts
import { restModule, getTokens } from '@ditsmod/rest';
import { authProviders } from './auth.providers.js';

@restModule({
  providersPerReq: [...authProviders],
  exports: [...getTokens(authProviders)],
})
export class AuthModule {}
```

Export only what consumer modules use directly. If an exported provider depends on an internal service, the internal service does not need to be exported unless consumers inject it directly.

Do not export controllers; exports apply to providers and modules.

## Re-Export Modules

Import and export a module when the current module should pass through another module's exports:

```ts
@restModule({
  imports: [AuthModule],
  exports: [AuthModule],
})
export class SecurityModule {}
```

If importing a `ModuleWithParams` object, export the same object instance:

```ts
const usersWithPath = { module: UsersModule, path: 'users' };

@restModule({
  imports: [usersWithPath],
  exports: [usersWithPath],
})
export class ApiModule {}
```

## Module Parameters

Use a static method when a module is commonly imported with parameters:

```ts
import { type ModuleWithParams } from '@ditsmod/core';

export class UsersModule {
  static withPrefix(path: string): ModuleWithParams<UsersModule> {
    return {
      module: this,
      path,
    };
  }
}
```

Then import it clearly:

```ts
@restModule({
  imports: [UsersModule.withPrefix('users')],
})
export class ApiModule {}
```

Use `extensionsMeta` in `ModuleWithParams` for extension-specific data. Keep one extension's data under one key.

## Provider Lifetime Across Modules

Imported provider classes are imported, not their instances. If a provider is declared at module/route/request level in two modules, instances are not shared between those modules. Use `providersPerApp` when a singleton must be shared application-wide.

## Collision Handling

When two imported modules export non-identical providers with the same token, resolve the collision in the consuming application module:

```ts
@restRootModule({
  imports: [DefaultAuthModule, CustomAuthModule],
  resolvedCollisionPerMod: [[AuthService, CustomAuthModule]],
})
export class AppModule {}
```

Use the `resolvedCollisionPer*` array that matches the provider level in the error message. If the collision comes from modules exported by the root module into an external package module, remove one exported module from the root and import it explicitly where needed.

## App Checklist

1. Keep each feature module narrowly specialized.
2. Use `imports` for provider/extension consumption and `appends` for route attachment only.
3. Export tokens or modules, not provider objects.
4. Use `getTokens()` when exporting tokens from a provider array containing object providers.
5. Use `ModuleWithParams` static methods for reusable parameterized imports.
6. Re-export the same `ModuleWithParams` object instance that was imported.
7. Resolve provider collisions explicitly instead of relying on import order.
