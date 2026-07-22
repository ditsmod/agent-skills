---
name: ditsmod-core-architecture
description: 'Core concepts of Ditsmod architecture: Modules, Dependency Injection, and the Metadata Reflector. Guidance on module composition, DI hierarchies, providers, scopes, and custom decorator design.'
---

# Ditsmod Core Architecture

This skill covers the three fundamental pillars of the Ditsmod framework: the Metadata Reflector, Dependency Injection (DI) and Modules.

## Part 1: Modules

### Choose The Module Type

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
  resolvedCollisionsPerMod: [], // Collision resolution at module level
  resolvedCollisionsPerRou: [], // Collision resolution at route level
  resolvedCollisionsPerReq: [], // Collision resolution at request level
})
export class SomeModule {}

@rootModule({
  // All featureModule properties, plus:
  resolvedCollisionsPerApp: [], // Collision resolution at app level
})
export class AppModule {}
```

`@ditsmod/rest` and `@ditsmod/trpc` provide their own decorators with extended metadata:

```ts
import { restRootModule, restModule } from '@ditsmod/rest';

@restModule({
  // The same list of properties as the 'restRootModule' decorator, except 'resolvedCollisionsPerApp'
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
  // The same list of properties as the 'trpcRootModule' decorator, except 'resolvedCollisionsPerApp'
})
export class SomeModule {}

@trpcRootModule({
  // All rootModule properties, plus:
  controllers: [], // Register tRPC controllers directly on the root module
})
export class AppModule {}
```

> **Do not mix** `@ditsmod/rest` and `@ditsmod/trpc` entities in the same application. Check a package's `peerDependencies` to confirm architectural style compatibility.

### Static vs Dynamic Modules

Ditsmod supports importing modules in two forms:

- **Static Module:** A raw class reference (e.g., `SomeModule`). Use this when importing a module with its default, statically declared metadata and providers.
  ```ts
  imports: [SomeModule];
  ```
- **Dynamic Module:** An object configuration implementing the `DynamicModule` interface (e.g., `{ module: SomeModule, path: 'prefix' }`). Use this when configuration details, route prefixes, or custom provider overrides need to be supplied at the import site. Dynamic modules are often returned by static helper methods (e.g., `SomeModule.withConfig(opts)`).
  ```ts
  imports: [{ module: SomeModule, providersPerReq: [MyOverrideService] }];
  ```

### Import, Append, Export

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

### Export Providers By Token

Export provider tokens, not provider objects. `providersPerApp` providers are globally available and **must not** be listed in `exports` (doing so will throw a `ForbiddenAppExport` error at bootstrap).

Use `getTokens()` when the provider array contains object-form providers (e.g., `{ token, useClass, ... }`), because their tokens cannot be statically extracted otherwise:

```ts
import { getTokens } from '@ditsmod/core';
import { restModule } from '@ditsmod/rest';
import { authProviders } from './auth.providers.js';

@restModule({
  providersPerReq: [...authProviders],
  exports: [...getTokens(authProviders)],
})
export class AuthModule {}
```

Export only what consumer modules inject directly. Internal implementation services do not need to be exported unless consumers inject them. Do not export controllers — exports apply only to providers and modules.

#### Exporting and Re-Exporting from the Root Module

Exporting providers or re-exporting modules from the root module (`AppModule`) is a special capability in Ditsmod:

- Any providers listed in `providersPerMod`, `providersPerRou`, or `providersPerReq` of the root module that are also added to its `exports` will be automatically imported into **all other application modules** (at their respective levels: module, route, or request).
- Similarly, if `AppModule` re-exports other modules (by listing them in both `imports` and `exports`), their exported providers/extensions will also be automatically imported into **all other application modules**.
- **Crucial nuance:** These exported/re-exported providers and modules will **not** be added to external modules (those imported from `node_modules`). This makes it a clean way to share providers and modules across all local feature modules without polluting third-party packages.

### Re-Export Modules

To pass through another module's exports to importers of the current module, both import and export it:

```ts
@restModule({
  imports: [AuthModule],
  exports: [AuthModule],
})
export class SecurityModule {}
```

When re-exporting a `DynamicModule` object, export the **same object reference** that was imported:

```ts
const usersWithPath = { module: UsersModule, path: 'users' };

@restModule({
  imports: [usersWithPath],
  exports: [usersWithPath], // must be the identical object, not a new literal
})
export class ApiModule {}
```

### Module Parameters

Use a static factory method when a module is commonly imported with configuration:

```ts
import { type DynamicModule } from '@ditsmod/core';

export class UsersModule {
  static withPrefix(path: string): DynamicModule<UsersModule> {
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

Pass extension-specific data via `extensionsMeta` inside `DynamicModule`. Keep each extension's data under a single dedicated key.

### Init Decorators

An **init decorator** is a custom class decorator created using `Reflector.makeClassDecorator()` whose options are processed and normalized by **init hooks** during the module initialization phase.

> [!WARNING]
> If you can easily pass metadata to a module using a dynamic module, creating an init decorator is **not recommended**. Consider using a dynamic module first.

For details on the roles of init decorators (root module, feature module, and modifier decorators), usage rules, and parent init hook propagation mechanics, see [references/REFERENCE.md](references/REFERENCE.md#part-3-init-decorators-and-inithooks).

### Provider Visibility And Lifetime

Provider registration level determines scope and sharing:

| Level   | Array             | Scope                  | Shared across modules?            |
| ------- | ----------------- | ---------------------- | --------------------------------- |
| App     | `providersPerApp` | Entire app             | Yes — singleton, no export needed |
| Module  | `providersPerMod` | Module + its importers | Only if exported                  |
| Route   | `providersPerRou` | Single route subtree   | Only if exported                  |
| Request | `providersPerReq` | Single HTTP request    | Only if exported                  |

Imported provider classes are registered — not their instances. If the same provider class is declared at mod/rou/req level in two separate modules, each module gets its own instance. Use `providersPerApp` when a true singleton must be shared across all modules.

### Collision Handling

A collision occurs when two imported modules export non-identical providers under the same token. Resolve it in the module that imports both:

```ts
@restRootModule({
  imports: [DefaultAuthModule, CustomAuthModule],
  resolvedCollisionsPerMod: [[AuthService, CustomAuthModule]],
})
export class AppModule {}
```

Match the `resolvedCollisionsPer*` array to the provider scope level named in the collision error:

- **Root module constraint:** `resolvedCollisionsPerApp` is **only** available and valid on the root module (`rootModule` / `restRootModule` / `trpcRootModule`). You cannot configure or resolve application-level provider collisions inside feature modules.
- **Feature module resolution:** Collisions at module, route, or request levels (`resolvedCollisionsPerMod`, `resolvedCollisionsPerRou`, `resolvedCollisionsPerReq`) must be resolved in whichever importing module encounters the conflict.
- If the collision originates from modules re-exported by a third-party package's root module, remove the conflicting re-exported module from the package root and import it explicitly where needed.

#### Default Providers and Collisions

Every `@ditsmod/*` package can define and export default provider arrays configured for different scopes (for example, `@ditsmod/core` exports `defaultProvidersPerApp` which includes `Logger`, `SystemLogMediator`, etc., and `defaultProvidersPerMod` which includes `ModuleInfo`).

These default providers are automatically added to their respective scopes. However, when exporting providers or modules, conflicts can arise:

- If multiple imported modules export different providers for the same default tokens (e.g., conflicting definitions of `Logger` or `ModuleInfo`), a token collision occurs.
- Avoid exporting default providers from your modules unless you are explicitly overriding them.
- If a collision occurs on default tokens, resolve it in the importing module using the corresponding `resolvedCollisionsPer*` metadata.

### Common Mistakes

- **Exporting controller classes** — controllers are not providers; only export injectable service tokens and modules.
- **Exporting `providersPerApp` providers** — they are registered at the application level and globally inherited; exporting them explicitly throws a `ForbiddenAppExport` validation error.
- **Exporting a new `DynamicModule` literal on re-export** — always reuse the same object reference that was passed to `imports`.
- **Using `appends` for provider consumption** — `appends` only mounts controllers; it does not make the appended module's providers available.
- **Expecting cross-module instance sharing at mod/rou/req level** — each module that declares a provider gets its own instance; use `providersPerApp` for true singletons.
- **Omitting `path` in `imports` when controllers must be mounted** — without `path`, controllers from the imported module are not mounted even if the module has them.

### Module Assembly Checklist

1. Keep each feature module narrowly specialized.
2. Use `imports` for provider/extension consumption; use `appends` for route attachment only.
3. Export tokens or modules — not provider objects.
4. Use `getTokens()` when exporting tokens from an array that contains object-form providers.
5. Use `DynamicModule` static methods for reusable parameterized imports.
6. When re-exporting `DynamicModule`, export the same object reference that was imported.
7. Resolve provider token collisions explicitly via `resolvedCollisionsPer*` instead of relying on import order.


## Part 2: Dependency Injection

> [!IMPORTANT]
> In Ditsmod applications, providers are registered inside modules (such as in `providersPer*` arrays) which shape the DI hierarchy. Before registering or wiring providers in application code, you **must** read the Modules section above to understand provider visibility, imports/exports, and collision resolution. Note that in unit tests, injectors can be instantiated directly (e.g., using `Injector.resolveAndCreate()`) without modules.

### Working Rules

- **`@injectable()` is mandatory** for every class that has constructor parameters used as DI dependencies. Without it, TypeScript does not emit parameter metadata and DI silently fails with a `No provider for [Token]!` error even when the provider is correctly registered:

  ```ts
  // ❌ WRONG — missing @injectable(), DI will fail silently
  class MyService {
    constructor(private dep: OtherService) {}
  }

  // ✅ CORRECT
  @injectable()
  class MyService {
    constructor(private dep: OtherService) {}
  }
  ```

- Treat TypeScript-only constructs (`interface`, `type`, `declare`, `enum` used only as types, or type-only imports) as unavailable at JavaScript runtime; do not use them as DI tokens.
- **Array Token Constraint:** An array cannot be used simultaneously as a TypeScript type and as a DI token. Use an instance of `InjectionToken<T[]>` or a string/symbol token for arrays, matching it with the proper TypeScript interface in the constructor.
- Remember that provider values are cached in the injector where their provider is registered (unless contextual data is passed via `@inject`).
- When diagnosing hierarchy errors, use the error's `Resolution path` to identify the injector level where each token was found or missing.

### Providers And Tokens

Recognize these provider forms:

```ts
import { InjectionToken, type Provider } from '@ditsmod/core';

class Logger {}
class ConsoleLogger extends Logger {}

interface LoggerConfig {
  level: 'info' | 'debug';
}

export const LOGGER_CONFIG = new InjectionToken<LoggerConfig>('LOGGER_CONFIG');

const providers: Provider[] = [
  Logger, // TypeProvider (shorthand)
  { token: LOGGER_CONFIG, useValue: { level: 'info' } },
  { token: Logger, useClass: ConsoleLogger },
  { token: 'alias-to-config', useToken: LOGGER_CONFIG },
];
```

The shorthand `SomeService` is exactly equivalent to `{ token: SomeService, useClass: SomeService }`.

Duplicate regular providers with the same token resolve to the last provider in the same injector. Multi-providers are the exception.

#### TokenProvider Resolution Rule

A `TokenProvider` creates an alias to another token. It is **not self-sufficient**. The dependency chain formed by `useToken` properties must ultimately resolve to a non-TokenProvider (`ValueProvider`, `ClassProvider`, `FactoryProvider`, or `TypeProvider`). If the target token lacks a concrete provider in the hierarchy, DI throws a `No provider for [TargetToken]!` error.

### Factory Providers

Prefer class factory providers when the factory needs decorated parameters or better encapsulation:

```ts
import { factoryMethod } from '@ditsmod/core';

class ConfigFactory {
  @factoryMethod()
  createConfig() {
    return { level: 'debug' };
  }
}

const provider = {
  token: LOGGER_CONFIG,
  useFactory: [ConfigFactory, ConfigFactory.prototype.createConfig],
};
```

Use function factory providers when plain token dependencies are enough:

```ts
const provider = {
  token: LOGGER_CONFIG,
  deps: ['log-level'],
  useFactory: (level: LoggerConfig['level']) => ({ level }),
};
```

For function factories, `deps` contains dependency tokens, not providers. Parameter decorators such as `optional`, `fromSelf`, or `skipSelf` are not passed in `deps`; use a class factory provider if those decorators are needed.

#### Parameter Validation & Transformation (Pipes) via Factory Providers

Ditsmod does not require a separate Pipe layer (like NestJS `PipeTransform`). Parameter validation and transformation are implemented inside custom `FactoryProvider` logic, using `@input` (for class factories) or the `input` token in `deps` (for function factories).

Passing an input argument in `@inject(Token, 'paramName')` causes DI to skip caching for that dependency and execute the factory afresh on each injection site. The developer's factory code then extracts raw request data, validates it, and returns the parsed result:

- **`ClassFactoryProvider` Pipe:** `@injectable()` class with `@factoryMethod()`, receiving request context via `@ctx(...)` and parameter name via `@input`.
- **`FunctionFactoryProvider` Pipe:** Function provider with `deps: [Context, input]`.

See [Parameter Validation & Transformation (Pipes)](references/REFERENCE.md#parameter-validation--transformation-pipes-via-factory-providers) in `references/REFERENCE.md` for full implementation details.

### Injector Hierarchy

Ditsmod application injectors are organized as a parent-child hierarchy (top = root/ancestor, bottom = deepest child):

```
providersPerApp   ← root (ancestor)
  └─ providersPerMod
       └─ providersPerRou
            └─ providersPerReq   ← deepest child
```

Child injectors look **up** to parents for missing tokens; parent injectors never look down to children.

Apply this rule when placing providers:
If provider `A` depends on provider `B`, `B` must be registered in the same injector as `A` or in an ancestor injector. Do not put `B` in a child injector.

Important consequences:

- Child injectors can ask parent injectors for missing tokens.
- Parent injectors never ask child injectors for tokens.
- If a child delegates `get(Service)` to a parent, the parent creates `Service` using dependencies visible from the parent level, not from the child level.
- Register a service in the child injector when it must use child-level overrides.
- **Provider Shadowing:** Registering a provider at a higher level (e.g., `providersPerApp` or `providersPerMod`) will **not** override or change the behavior of that token at a lower level (e.g., `providersPerReq`) if the lower level already has its own provider registered for that token. The child-level provider shadows (overrides) the parent-level provider, not vice-versa.
- Name injectors (e.g., `Injector.resolveAndCreate(providers, 'injectorName')`) in low-level tests or examples when hierarchy diagnostics matter.

### `get()` Versus `pull()`

Use `injector.get(Token)` for normal resolution and caching.

Use `child.pull(Token)` only for the specific case where the child lacks a provider that exists in an ancestor, but the pulled provider should be created in the child context so it can use child-level dependencies.

- **Pulled from ancestor:** `pull()` temporarily brings the ancestor provider into the child context, creates a **new instance each time** (no caching), using child-level dependencies.
- **Exists in child:** If the provider already exists in the current child injector, `pull()` acts identically to `get()` — creates and caches the instance on first call, returns cached value on subsequent calls.

### Multi-Providers

Use `multi: true` only with object providers and only when all providers for the same token in the same injector are multi-providers:

```ts
const LOCALES = new InjectionToken<string[]>('LOCALES');

const providers = [
  { token: LOCALES, useValue: 'uk', multi: true },
  { token: LOCALES, useValue: 'en', multi: true },
];
```

Do not mix regular and multi-providers for the same token in one injector. If a child injector declares multi-providers for a token, it returns its own multi-provider array for that token rather than merging parent values.

To make one multi-provider entry substitutable, point the multi-provider at a class with `useToken`, then override that class with a normal class provider.

> [!TIP]
> Authors of shared libraries or npm packages should prefer registering multi-providers via `useToken` pointing to a distinct class or token, rather than using `useClass` or `useValue` directly. This enables consumers of the package to easily substitute individual multi-provider entries by overriding the referenced class/token in their application injectors.

### Context

Use `Context` when data must be set after injector creation and read later at the same or lower injector level. Use `createInjectionSymbol<T>()` for typed context keys.

Always include `...contextProviders` in `Injector.resolveAndCreate()` when using `@ctx()` in class method parameters directly. In modular Ditsmod applications that import `ContextModule` (re-exported by `@ditsmod/rest`), these providers are already available — do not duplicate them.

```ts
import { Context, Injector, ctx, contextProviders, createInjectionSymbol } from '@ditsmod/core';

interface RequestState {
  userId: string;
}

const REQUEST_STATE = createInjectionSymbol<RequestState>('REQUEST_STATE');

class Handler {
  handle(@ctx(REQUEST_STATE) state: RequestState) {
    return state.userId;
  }
}

const injector = Injector.resolveAndCreate([
  ...contextProviders,
  { token: 'user-id', useFactory: [Handler, Handler.prototype.handle] },
]);

injector.get(Context).set(REQUEST_STATE, { userId: '42' });
const userId = injector.get('user-id'); // '42'
```

`ctx.get(key)` retrieves a value from the current context instance. `ctx.getInScope(key, injector)` traverses up the injector hierarchy to find the value, useful when Context instances exist at multiple levels.

### Special Token: ParentParams (Inheritance)

When an `@injectable()` class extends a parent class, DI automatically injects an array of the parent's constructor arguments into the `ParentParams` token.

Three approaches (choose one):

**Option A — `@ts-expect-error` (Recommended, simplest):**

```ts
import { ParentParams, injectable } from '@ditsmod/core/di';

@injectable()
class Child extends Parent {
  constructor(
    parentParams: ParentParams,
    public childParam1: ChildParam1,
  ) {
    // @ts-expect-error auto-injected
    super(...parentParams);
  }
}
```

**Option B — `@inject` decorator (type-safe, no suppression):**

```ts
import { ParentParams, injectable, inject } from '@ditsmod/core/di';

@injectable()
class Child extends Parent {
  constructor(
    @inject(ParentParams) parentParams: ConstructorParameters<typeof Parent>,
    public childParam1: ChildParam1,
  ) {
    super(...parentParams);
  }
}
```

**Option C — inline type assertion (type-safe, no suppression):**

```ts
@injectable()
class Child extends Parent {
  constructor(
    parentParams: ParentParams,
    public childParam1: ChildParam1,
  ) {
    super(...(parentParams as ConstructorParameters<typeof Parent>));
  }
}
```

### Current Injector & Lazy Loading

A service or controller can access the specific injector instance that instantiated it by requesting `Injector` directly in its constructor. This pattern is primarily used for **lazy loading** dependencies at runtime:

```ts
import { injectable, Injector } from '@ditsmod/core';
import { FirstService } from './first.service.js';

@injectable()
export class SecondService {
  constructor(private injector: Injector) {}

  someMethod() {
    // FirstService is requested dynamically only when someMethod is called
    const firstService = this.injector.get(FirstService);
  }
}
```

### Parameter Decorators

- Use `@inject(token)` when the runtime token differs from the TypeScript parameter type.
- **Contextual Input Decorator (`@inject` + `@input`):** You can pass contextual data during dependency instantiation by passing a second argument to `@inject(Dependency, 'input-data')`. The target dependency must accept it via the `@input` decorator (shorthand for `@inject(input)`).
- **Critical Nuance:** When a second argument is passed to `@inject()`, the injector **does not create a cache** for the specified dependency.
- Use `@optional()` when absence of a provider is valid. TypeScript optional syntax (`?`) alone is ignored by the JavaScript runtime DI mechanism.
- Use `@fromSelf()` to force lookup only in the injector creating the current value.
- Use `@skipSelf()` to start lookup from the parent injector.


## Part 3: Metadata Reflector

### Mental Model

Standard JavaScript `Reflect` API (supplemented by `reflect-metadata`) allows storing and retrieving arbitrary metadata keys on classes, properties, or parameters. However, it operates on low-level string keys (e.g., `design:paramtypes`) and does not natively support class inheritance merging or decorator structures.

Ditsmod's `Reflector` provides a unified, cached, and high-level abstraction:

- **Unified Decorators:** It generates type-safe decorators that automatically register metadata in internal registries when evaluated.
- **Inheritance Traversal:** It merges class, property, and parameter metadata from parent classes down to children.
- **Iterability:** `collectMeta(Cls)` returns a `MergedClassMeta` object which is iterable and provides property-by-property details.
- **Cache-Optimized:** Reflection metadata is resolved once and cached internally.

### Prerequisites & Compilation

For reflection metadata to work, ensure these requirements are met:

1. **tsconfig.json Flags:**
   ```json
   {
     "compilerOptions": {
       "experimentalDecorators": true,
       "emitDecoratorMetadata": true
     }
   }
   ```
2. **Runtime Polyfill:**
   `reflect-metadata/lite` must be imported before any decorators are evaluated. In typical Ditsmod applications, this is already imported automatically by `@ditsmod/core`, so you do **not** need to add it manually to the application entry points.
3. **Test Setup:**
   In test configurations (like Jest or Vitest), if `@ditsmod/core` is not imported early enough (or at all), ensure `reflect-metadata/lite` is loaded via `setupFilesAfterEnv` or an explicit top-level import.

For details on creating custom decorators, collecting metadata, inheritance chains, and programmatic metadata writing, see [references/REFERENCE.md](references/REFERENCE.md#part-4-metadata-reflector-references).


## Core Debugging & Troubleshooting Checklists

### Dependency Injection Debugging Checklist

1. Identify the requested token and the injector where the request starts.
2. Read the `Resolution path` in the error message (format: `[Token in injectorA >> injectorB] -> [Dep in injectorB]`). The `>>` chain shows which injectors were traversed to find the token; the `->` arrow shows the next dependency that failed. See the technical reference guide for detailed examples.
3. Find the provider level for that token.
4. For every constructor or factory dependency, verify that its provider is at the same level or an ancestor of the provider that needs it.
5. Check whether a child-level override is ineffective because the requested service is actually created in a parent injector.
6. Check for invalid runtime tokens caused by interfaces, type aliases, enums used only as types, or `import type`.
7. Check for missing `@injectable()` on a class that has constructor dependencies.
8. Ensure that array types are not being passed directly as runtime tokens; check for `InjectionToken` usage.
9. Verify that any `TokenProvider` (using `useToken`) eventually terminates in a non-token provider mapping.
10. Check for accidental mixing of regular and multi-providers.

### Metadata Reflector Troubleshooting Checklist

1. **`Reflect.defineMetadata is not a function` error:**
   - Ensure `@ditsmod/core` is imported (which automatically loads `reflect-metadata/lite` at startup), or explicitly import `reflect-metadata/lite` in entry points/test configurations that do not load `@ditsmod/core` early enough (e.g. `setupFilesAfterEnv` in Jest).
2. **Missing metadata for constructor arguments or properties:**
   - Verify that the class is decorated (e.g. with `@injectable()`, `@controller()`, etc.). If a class or property has no decorators at all, the TypeScript compiler will not emit any design metadata, even with `emitDecoratorMetadata` enabled.
   - Verify that `"emitDecoratorMetadata": true` and `"experimentalDecorators": true` are enabled in `tsconfig.json`.
3. **TypeScript compiler loses types after compiling imports:**
   - Check if parameter types in the constructor are imported using `import type` or interfaces. TypeScript does not emit runtime metadata for type-only constructs. Use classes, or use the `@inject(MY_TOKEN)` decorator on the parameter alongside the corresponding TypeScript interface/type.
4. **Decorator decoratorId type checks fail:**
   - When creating complex decorator factories, ensure you provide a valid `decoratorId` function/symbol and match it with a corresponding `TypeGuard`.
