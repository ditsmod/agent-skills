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
  resolvedCollisionsPerMod?: [any, ModRefId | ForwardRefFn<ModuleType>][];
  resolvedCollisionsPerRou?: [any, ModRefId | ForwardRefFn<ModuleType>][];
  resolvedCollisionsPerReq?: [any, ModRefId | ForwardRefFn<ModuleType>][];
}
```

`rootModule` adds one more field:

```ts
resolvedCollisionsPerApp?: [any, ModRefId | ForwardRefFn<ModuleType>][];
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
  initOpts?: InitOptsMap; // present when used with init decorators
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

Import order does **not** determine which provider wins in a collision. Ditsmod detects any ambiguity between two imported modules that export different providers for the same token and throws a collision error, regardless of import order. Always resolve collisions explicitly via `resolvedCollisionsPer*`.

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

- Identify the scope: `providersPerMod` → use `resolvedCollisionsPerMod`
- Identify the token: `LogService`
- Choose which module's provider to prefer: `[LogService, PreferredModule]`
- Add to the consuming module:

```ts
resolvedCollisionsPerMod: [[LogService, PreferredModule]],
```

## Init Decorators and InitHooks {#init-decorators}

### Custom Decorator Creation Syntax

Use `Reflector.makeClassDecorator(transformInitDecoratorOptions, name, parentInitDecorator)` to create an init decorator:

```ts
import {
  InitHooks,
  InitDecorator,
  Reflector,
  InitDecoratorOptions,
  DynamicModuleOptions,
  NormalizedInitMeta,
  NormalizedModuleMeta,
  RootDecoratorOptions,
} from '@ditsmod/core';
// ...

/**
 * An object with this type will be passed directly to the init decorator - @initSome({ one: 1, two: 2 })
 */
interface ExtInitDecorOpts extends InitDecoratorOptions<InitOpts> {
  one?: number;
  two?: number;
}

/**
 * The methods of this class will normalize and validate the module metadata.
 */
class SomeInitHooks extends InitHooks<ExtInitDecorOpts> {
  // ...
}

/**
 * An object with this type will be passed in the module metadata as a so-called "DynamicModule".
 */
interface InitOpts extends DynamicModuleOptions {
  path?: string;
  num?: number;
}

/**
 * Init hooks transform an object of ExtInitDecorOpts into an object of that type.
 */
interface InitMeta extends NormalizedInitMeta {
  normalizedModuleMeta: NormalizedModuleMeta;
  initDecoratorOptions: RootDecoratorOptions;
}

function transformInitDecoratorOptions(data?: ExtInitDecorOpts): InitHooks<ExtInitDecorOpts> {
  const metadata = Object.assign({}, data);
  const initHooks = new SomeInitHooks(metadata);
  initHooks.moduleRole = undefined;
  // OR initHooks.moduleRole = 'root';
  // OR initHooks.moduleRole = 'feature';
  return initHooks;
}

// Creating the init decorator
const initSome: InitDecorator<ExtInitDecorOpts, InitOpts, InitMeta> =
  Reflector.makeClassDecorator(transformInitDecoratorOptions);

// Using init decorator
@initSome({ one: 1, two: 2 })
export class SomeModule {}
```

### `InitHooks` Methods and Properties

- `moduleRole?: 'root' | 'feature'`: Set to `'root'` or `'feature'` to make the init decorator act as a complete module decorator (meaning standard decorators are not required).
- `hostModule?: ModuleType`: If specified, the module class representing the host module will be automatically imported into any module class decorated with this init decorator.
- `hostDecoratorOptions?: T`: Raw options to pass to the decorator for the host module. When `hostModule` is normalizer-scanned, this allows attaching metadata to the host module class without directly decorating it (avoiding circular imports).
- `normalize(normalizedModuleMeta)`: Normalizes and validates raw options, returning a normalized metadata object that is saved in `normalizedModuleMeta.initMeta`.
- `getModulesToScan(meta)`: Returns an array of `ModRefId` modules that should also be scanned (e.g., appended modules in REST).
- `exportAppProviders(config)`: Invoked at bootstrap to collect and export application-level providers.
- `importModulesShallow(config)`: Invoked during shallow imports step to collect routes, paths, controllers, and guards.
- `importModulesDeep(config)`: Invoked during deep imports step to resolve provider dependencies.
- `getProvidersToOverride(meta)`: Returns array of arrays of providers that are overridable.

### Grouping Init-Decorators via `decoratorId`

When creating a substitute decorator (with `'root'` or `'feature'` role) using `Reflector.makeClassDecorator()`, you **must** pass the base modifier decorator (e.g. `initRest` or `initSome`) as the third argument. This third argument serves as the `decoratorId`. It tells Ditsmod that these decorators belong to the same group, enabling the framework to correctly collect, normalize, and associate metadata with the proper group context during initialization.

### Separation of Module Logic and Substitute Decorators via hostModule

Separating the decorator's metadata definition from the host feature module is necessary to avoid circular dependencies (decorating the host module with itself would create an import loop):

1. Create a standard feature module (e.g. `MyCoreModule`) containing all necessary providers, middlewares, and extensions.
2. In your custom `InitHooks` subclass, specify `override hostModule = MyCoreModule`.
3. Create the base modifier decorator (e.g. `initMy`) which serves as the group parent decorator.
4. Create the transformer function that returns the hooks instance and sets `hooks.moduleRole = 'feature'` (or `'root'`).
5. Create the substitute custom decorator (e.g. `myFeatureModule`) using `Reflector.makeClassDecorator()`, passing the transformer as the first argument, its name as the second, and the base modifier decorator (`initMy`) as the third argument (the parent).
6. When developers apply this substitute decorator (e.g. `@myFeatureModule`), the framework recognizes it as a module decorator (requiring only one decorator on the class instead of two) and automatically imports `MyCoreModule`.

Example:

```ts
import { featureModule, InitHooks, Reflector } from '@ditsmod/core';

// 1. Standard module containing actual logic/providers
@featureModule({
  providersPerReq: [MyService],
  exports: [MyService],
})
export class MyCoreModule {}

// 2. Custom hooks setting hostModule
class MyInitHooks extends InitHooks {
  override hostModule = MyCoreModule;
}

// 3. Creating the base modifier decorator (serves as the group parent)
export const initMy = Reflector.makeClassDecorator((data) => new MyInitHooks(data), 'initMy');

// 4. Creating the transformer that sets moduleRole = 'feature'
function transformFeatureMeta(data?: any) {
  const hooks = new MyInitHooks(data);
  hooks.moduleRole = 'feature'; // Makes it a substitute module decorator
  return hooks;
}

// 5. Creating the substitute decorator, passing initMy as the 3rd argument
export const myFeatureModule = Reflector.makeClassDecorator(transformFeatureMeta, 'myFeatureModule', initMy);

// 6. Using only one decorator on the class (automatically imports MyCoreModule)
@myFeatureModule()
export class MyFeatureModule {}
```

### Parameter Merging and plain Modules with Parameters (MWP)

When importing a module with parameters in the context of an init decorator (e.g. `imports: [{ module: Module1, path: 'some-prefix' }]`):

1. The dynamic module's custom parameters (like `path` or `guards`) are merged into the `dynamicModule.initOpts` Map under the decorator's token.
2. If `Module1` itself is a plain `@featureModule` (not decorated with `@initRest` or `@restModule`), the framework automatically retrieves the default hook class for the decorator from the application's register, clones it, registers it in the module's `mInitHooks` list, and calls `normalize()`.
3. This ensures that custom parameters (such as REST routing prefixes and route guards) are correctly applied to plain feature modules during import.
