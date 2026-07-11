---
name: ditsmod-dependency-injection
description: "Ditsmod DI: InjectionToken, injector hierarchy (providersPerApp/providersPerMod/providersPerRou/providersPerReq), provider types (TypeProvider/ValueProvider/ClassProvider/FactoryProvider/TokenProvider), multi-providers, Context, createInjectionSymbol, ParentParams, parameter decorators (inject/input/optional/fromSelf/skipSelf), pull() vs get(), debugging 'No provider for X' errors, Resolution path interpretation, @injectable decorator."
---

# Ditsmod Dependency Injection

Use this skill to make DI changes in the Ditsmod codebase without guessing framework behavior.

## Working Rules

- **`@injectable()` is mandatory** for every class that has constructor parameters used as DI dependencies. Without it, TypeScript does not emit parameter metadata and DI silently fails with a `No provider for [Token]!` error even when the provider is correctly registered.
- Treat TypeScript-only constructs (`interface`, `type`, `declare`, `enum` used only as types, or type-only imports) as unavailable at JavaScript runtime; do not use them as DI tokens.
- **Array Token Constraint:** An array cannot be used simultaneously as a TypeScript type and as a DI token. Use an instance of `InjectionToken<T[]>` or a string/symbol token for arrays, matching it with the proper TypeScript interface in the constructor.
- Remember that provider values are cached in the injector where their provider is registered (unless contextual data is passed via `@inject`).
- When diagnosing hierarchy errors, use the error's `Resolution path` to identify the injector level where each token was found or missing. See [references/REFERENCE.md](references/REFERENCE.md) for Resolution path interpretation guide.

## Providers And Tokens

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

### TokenProvider Resolution Rule

A `TokenProvider` creates an alias to another token. It is **not self-sufficient**. The dependency chain formed by `useToken` properties must ultimately resolve to a non-TokenProvider (`ValueProvider`, `ClassProvider`, `FactoryProvider`, or `TypeProvider`). If the target token lacks a concrete provider in the hierarchy, DI throws a `No provider for [TargetToken]!` error.

---

## Factory Providers

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

## Injector Hierarchy

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
- Name injectors (e.g., `Injector.resolveAndCreate(providers, 'injectorName')`) in low-level tests or examples when hierarchy diagnostics matter.

## `get()` Versus `pull()`

Use `injector.get(Token)` for normal resolution and caching.

Use `child.pull(Token)` only for the specific case where the child lacks a provider that exists in an ancestor, but the pulled provider should be created in the child context so it can use child-level dependencies.

- **Pulled from ancestor:** `pull()` temporarily brings the ancestor provider into the child context, creates a **new instance each time** (no caching), using child-level dependencies.
- **Exists in child:** If the provider already exists in the current child injector, `pull()` acts identically to `get()` — creates and caches the instance on first call, returns cached value on subsequent calls.

## Multi-Providers

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

## Context

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

## Special Token: ParentParams (Inheritance)

When an `@injectable()` class extends a parent class, DI automatically injects an array of the parent's constructor arguments into the `ParentParams` token.

Three approaches (choose one):

**Option A — `@ts-expect-error` (simplest):**

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

---

## Current Injector & Lazy Loading

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

## Parameter Decorators

- Use `@inject(token)` when the runtime token differs from the TypeScript parameter type.
- **Contextual Input Decorator (`@inject` + `@input`):** You can pass contextual data during dependency instantiation by passing a second argument to `@inject(Dependency, 'input-data')`. The target dependency must accept it via the `@input` decorator (shorthand for `@inject(input)`).
- **Critical Nuance:** When a second argument is passed to `@inject()`, the injector **does not create a cache** for the specified dependency.

- Use `@optional()` when absence of a provider is valid. TypeScript optional syntax (`?`) alone is ignored by the JavaScript runtime DI mechanism.
- Use `@fromSelf()` to force lookup only in the injector creating the current value.
- Use `@skipSelf()` to start lookup from the parent injector.

## Debugging Checklist

1. Identify the requested token and the injector where the request starts.
2. Read the `Resolution path` in the error message (format: `[Token in injectorA >> injectorB] -> [Dep in injectorB]`). The `>>` chain shows which injectors were traversed to find the token; the `->` arrow shows the next dependency that failed. See [references/REFERENCE.md](references/REFERENCE.md) for detailed examples.
3. Find the provider level for that token.
4. For every constructor or factory dependency, verify that its provider is at the same level or an ancestor of the provider that needs it.
5. Check whether a child-level override is ineffective because the requested service is actually created in a parent injector.
6. Check for invalid runtime tokens caused by interfaces, type aliases, enums used only as types, or `import type`.
7. Check for missing `@injectable()` on a class that has constructor dependencies.
8. Ensure that array types are not being passed directly as runtime tokens; check for `InjectionToken` usage.
9. Verify that any `TokenProvider` (using `useToken`) eventually terminates in a non-token provider mapping.
10. Check for accidental mixing of regular and multi-providers.
