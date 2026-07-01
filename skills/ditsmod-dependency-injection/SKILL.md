---
name: ditsmod-dependency-injection
description: Practical guidance for using dependency injection in Ditsmod applications. Use when building or fixing application services, controllers, guards, interceptors, modules, provider configuration, typed InjectionToken values, provider overrides, multi-providers, request-scoped data, Context, and parameter decorators in apps built with @ditsmod/rest or other Ditsmod application packages.
---

# Ditsmod Dependency Injection

Use this skill when writing application code that consumes Ditsmod DI. Keep the focus on app ergonomics: clear provider placement, typed tokens, predictable overrides, and request-safe data flow.

## Application Mental Model

Ditsmod app provider arrays map to injector levels. Place values by lifetime and override needs:

- Use `providersPerApp` for app-wide singletons and default configuration.
- Use `providersPerMod` for module-specific services or module-level configuration overrides.
- Use `providersPerRou` for route-level behavior.
- Use `providersPerReq` for request-scoped services and values that must not be shared across HTTP requests.

If a service depends on a config or helper, register that dependency in the same provider array or in an ancestor array. Do not put a dependency in a child scope.

## Tokens & Providers Configuration

### Token Constraints
- **Allowed Tokens:** Class references, string or numeric literals, symbols, or `InjectionToken<T>` instances.
- **Strict Forbidden:** TypeScript `interface`, `type`, `enum`, or `declare` constructs **cannot** be used as tokens because they do not exist in compiled JavaScript. Do not import tokens using the `type` keyword.

### Dependency Forms
- **Short Form:** `constructor(private service1: Service1)` (Allowed only when the token is exactly the class type).
- **Long Form:** Required when injecting interfaces, arrays, or custom tokens. Use the `@inject()` decorator:
  ```ts
  constructor(@inject(SOME_TOKEN) private items: InterfaceOfItem[]) {}
  ```

*Recommendation:* Always use `const SOME_TOKEN = new InjectionToken<T>('SOME_TOKEN')` for long-form type safety.

### Provider Types Syntax

```ts
// 1. ValueProvider
{ token: 'token1', useValue: 'some value' }

// 2. ClassProvider
{ token: Service2, useClass: ExtendedService2 }

// 3. FactoryProvider (Class-based - Recommended)
// Requires @factoryMethod() on the target method
{ token: 'token3', useFactory: [ClassWithFactory, ClassWithFactory.prototype.method1] }

// 4. FactoryProvider (Function-based)
// Parameters must be explicitly provided in 'deps'
{ token: 'token4', deps: [Dependency1, Dependency2], useFactory: myFunction }

// 5. TokenProvider (Alias creation)
{ token: ServiceAlias, useToken: RealService }
```

> **Duplicate Providers Rule:** If multiple regular providers share the same token, the **last** provider in the array overrides all previous ones.

## Injector Hierarchy & Encapsulation

Ditsmod operates on a hierarchical injector structure: `App` (Application) -> `Mod` (Module) -> `Rou` (Route) -> `Req` (Request).

### The Hierarchy Golden Rule

> **"If a provider depends on another provider, the dependency must not be placed at a lower level of the hierarchy (in child injectors)."**

- **Resolution Path:** Child injectors can look up the hierarchy chain to resolve dependencies in parent injectors, but parent injectors **never** query child injectors.
- **State & Caching:** Instances are cached within the specific injector level that registered the provider. If a child injector resolves a token from a parent, the instance lives and caches in the parent.

### Debugging & Naming Injectors

Always explicitly name injectors when creating them manually to make error `Resolution paths` scannable:

```ts
const child = parent.resolveAndCreateChild([Provider], 'childInjectorName');
```

### Advanced Hierarchy Methods

- `injector.pull(Token)`: Used in child injectors to force pulling a provider from the parent registry and instantiating it contextually within the child level (resolving child-level dependencies) *without* caching it globally.
- `constructor(private injector: Injector)`: Inject the current injector level directly into a service for patterns like lazy-loading dependencies.

## Multi-Providers

- Used to pass multiple values under a single token, returning an array of values.
- **Constraint:** You **cannot** mix regular providers and multi-providers for the same token within the same injector.
- **Override Pattern:** To substitute a multi-provider from an external module, use a `TokenProvider` pointing to a custom `ClassProvider`:

```ts
{ token: HTTP_INTERCEPTORS, useToken: DefaultInterceptor, multi: true },
DefaultInterceptor,
{ token: DefaultInterceptor, useClass: MyInterceptor }
```

## Request Data And Context

Use `Context` for values produced after injector creation, especially data produced by interceptors or guards and consumed by controllers or services later in the same request. Use as a mutable data intermediary across the immutable injector hierarchy.

Use `getSymbol<T>()` for typed context keys:

```ts
import { Context, getSymbol } from '@ditsmod/core';

export interface CurrentUser {
  id: string;
  roles: string[];
}

export const CURRENT_USER = getSymbol<CurrentUser>('CURRENT_USER');

export class CurrentUserWriter {
  constructor(private ctx: Context) {}

  setUser(user: CurrentUser) {
    this.ctx.set(CURRENT_USER, user);
  }
}
```

### Method Parameter Injection

To directly inject context values into class or controller methods, use the `@ctx()` decorator:

```ts
method1(@ctx('key1') param1: any) { ... }
```

## Parameter Decorators Reference

- `@inject(token)`: for non-class tokens and implementation aliases.
- `@input`: Used in constructor parameters to capture contextual data passed down from a parent `@inject(Dependency, 'contextual-data')` call.
- `@optional()`: Tells DI not to throw an error if no provider exists for this token (injects `undefined`).
- `@fromSelf()`: Restricts DI lookup strictly to the current injector level; skips parent lookup.
- `@skipSelf()`: Bypasses the current injector level immediately and begins the resolution chain from the parent injector.

## App Debugging Checklist

1. Confirm the failing token exists at runtime; replace interfaces/types with instance of `InjectionToken<T>`.
2. Confirm the consumer service/controller is registered at a scope that can see all its dependencies.
3. If an override is ignored, check whether the consuming service is created in a child injector.
4. If request data leaks across requests, move mutable data out of app/module singletons and into request scope or `Context`.
5. If a multi-provider result is incomplete, check whether a child scope replaced the parent multi-provider array.
6. If a function factory fails, verify every item in `deps` has a corresponding token.
