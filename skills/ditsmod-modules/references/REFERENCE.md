# Ditsmod Modules — Technical Reference

## Full Module Metadata Shapes

`@ditsmod/core` defines metadata via the `ModuleDecoratorOptions` class (passed to `rootModule` and `featureModule`):

```ts
// Source: @ditsmod/core — decorators/module-decorator-options.ts
class ModuleDecoratorOptions<T extends AnyObj = AnyObj> {
  // ModRefId = ModuleType | DynamicModule
  imports?: (ModRefId | ForwardRefFn<ModuleType>)[];
  exports?: any[];
  providersPerApp?: ProviderBuilder | (Provider | ForwardRefFn<Provider>)[];
  providersPerMod?: ProviderBuilder | (Provider | ForwardRefFn<Provider>)[];
  providersPerRou?: ProviderBuilder | (Provider | ForwardRefFn<Provider>)[];
  providersPerReq?: ProviderBuilder | (Provider | ForwardRefFn<Provider>)[];
  extensions?: (ExtensionConfig | ExtensionClass)[];
  extensionsMeta?: T; // generic — any object shape, not Record<string, unknown>
  resolvedCollisionPerMod?: [any, ModRefId | ForwardRefFn<ModuleType>][];
  resolvedCollisionPerRou?: [any, ModRefId | ForwardRefFn<ModuleType>][];
  resolvedCollisionPerReq?: [any, ModRefId | ForwardRefFn<ModuleType>][];
}
```

`rootModule` adds one more field:

```ts
resolvedCollisionPerApp?: [any, ModRefId | ForwardRefFn<ModuleType>][];
```

`@ditsmod/rest` and `@ditsmod/trpc` extend the metadata with their own fields (`appends`, `controllers`) — these are **not** part of `ModuleDecoratorOptions`.

## `DynamicModule` Shape

`DynamicModule` is composed of interfaces in `@ditsmod/core` (`DynamicModuleBase` and `DynamicModuleOptions`), and `DynamicModuleWithInit` extends `DynamicModule`:

```ts
// Source: @ditsmod/core — decorators/module-decorator-options.ts
interface DynamicModuleBase<M extends AnyObj = AnyObj> {
  id?: string; // optional module identity string for disambiguation
  module: ModuleType<M> | ForwardRefFn<ModuleType<M>>;
}

interface DynamicModuleOptions<E extends AnyObj = AnyObj> extends Partial<ProvidersByLevel> {
  // ProvidersByLevel provides: providersPerApp, providersPerMod, providersPerRou, providersPerReq
  exports?: any[];
  extensionsMeta?: E; // generic, not Record<string, unknown>
}

// The final exported type used in imports[]:
interface DynamicModule<M extends AnyObj = AnyObj> extends DynamicModuleBase<M>, DynamicModuleOptions {
  initParams?: InitParamsMap; // present when used with init decorators
}
```

Note: there is **no** index signature (`[key: string]: unknown`) in `DynamicModule`. Extra properties are not generically allowed.

`path` is **not** a field of `DynamicModule` from `@ditsmod/core`. When using `@ditsmod/rest`, the object passed to `imports[]` is typed as `RestModuleOptions`, which extends `DynamicModuleOptions` and adds route-specific fields:

```ts
// Source: @ditsmod/rest — init/rest-init-raw-meta.ts
// RestModuleOptions = PathRestModuleOptions | AbsolutePathRestModuleOptions
interface PathRestModuleOptions extends BaseRestModuleOptions {
  path?: string;
  absolutePath?: never; // mutually exclusive with absolutePath
}
interface AbsolutePathRestModuleOptions extends BaseRestModuleOptions {
  absolutePath?: string;
  path?: never; // mutually exclusive with path
}
interface BaseRestModuleOptions extends DynamicModuleOptions {
  exports?: any[];
  guards?: GuardItem[];
}
```

Use `path` for a relative route prefix, `absolutePath` for an absolute one. They cannot be set simultaneously.

When a module is imported with `DynamicModule`, the framework treats the object identity as the module's key. This is why re-exporting must use the **same object reference**.

## `getTokens()` Behavior

`getTokens(providers: Provider[]): Token[]` extracts the token from each provider in the array:

- `Class` → the class itself is the token
- `{ token, useValue | useClass | useFactory | useToken }` → extracts `token`
- Multi-providers (`{ token, useValue, multi: true }`) → token is included once

Use it whenever a provider array is constructed with object-form providers and you want to export all of them without manually listing tokens.

## Import Order vs. Collision Resolution

Import order does **not** determine which provider wins in a collision. Ditsmod detects any ambiguity between two imported modules that export different providers for the same token and throws a collision error, regardless of import order. Always resolve collisions explicitly via `resolvedCollisionPer*`.

## Route Prefix Composition

Path prefixes compose from outer to inner:

- Root module mounts a feature module at `api/v1` via `imports: [{ path: 'api/v1', module: ApiModule }]`
- `ApiModule` mounts `UsersModule` at `users` via `imports: [{ path: 'users', module: UsersModule }]`
- Controller route `GET :id` inside `UsersModule` → final URL: `GET /api/v1/users/:id`

The same composition applies for `appends`.

## Provider Scope Decision Guide

Use this decision tree when choosing provider scope:

1. Must be a single instance for the entire app lifetime? → `providersPerApp`
2. Must be a single instance per module (e.g., module-level config)? → `providersPerMod`
3. Must be a single instance per matched route (e.g., route-level state)? → `providersPerRou`
4. Must be fresh per HTTP request (e.g., request context, user session)? → `providersPerReq`

## `extensionsMeta` Usage Pattern

```ts
import { MY_EXTENSION } from './my.extension.js';

export class DataModule {
  static withConfig(config: DataConfig): DynamicModule<DataModule> {
    return {
      module: this,
      extensionsMeta: {
        [MY_EXTENSION]: config, // one key per extension
      },
    };
  }
}
```

Extensions receive this data when they process the module's metadata. Keep each extension's data isolated under its own symbol or string key to prevent conflicts between extensions.

## Common Collision Error Message Pattern

```
Error: Collision was found in DataModule (providersPerMod) for token LogService.
This collision can be resolved in one of the following modules: AppModule...
```

- Identify the scope: `providersPerMod` → use `resolvedCollisionPerMod`
- Identify the token: `LogService`
- Choose which module's provider to prefer: `[LogService, PreferredModule]`
- Add to the consuming module:

```ts
resolvedCollisionPerMod: [[LogService, PreferredModule]],
```
