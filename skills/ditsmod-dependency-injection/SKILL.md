---
name: ditsmod-dependency-injection
description: Guidance for implementing, refactoring, debugging, and reviewing Ditsmod dependency injection code. Use when working with @ditsmod/core DI concepts such as injector, providers, tokens, InjectionToken, class factories, injector hierarchy, providersPerApp/providersPerMod/providersPerRou/providersPerReq, injector.get(), injector.pull(), multi-providers, Context, ctxProviders, getSymbol(), ParentParams, and parameter decorators like inject, input, optional, fromSelf, and skipSelf.
---

# Ditsmod Dependency Injection

Use this skill to make DI changes in the Ditsmod codebase without guessing framework behavior.

## Reference Sources

This skill is the primary source of truth for Ditsmod DI logic to minimize file-reading overhead. Refer to the original documentation file ONLY if you encounter an ambiguous edge case not covered here, or if you suspect the codebase implementation has drifted from this specification:

`website/i18n/en/docusaurus-plugin-content-docs/current/01-basic-components/01-dependency-injection.md`

## Working Rules

- Use TypeScript examples.
- Prefer existing Ditsmod APIs and local patterns over inventing new abstractions.
- Treat TypeScript-only constructs (`interface`, `type`, `declare`, `enum` used only as types, or type-only imports) as unavailable at JavaScript runtime; do not use them as DI tokens.
- **Array Token Constraint:** An array cannot be used simultaneously as a TypeScript type and as a DI token. Use `InjectionToken<T[]>` or a string/symbol token for arrays, matching it with the proper TypeScript interface in the constructor.
- Remember that provider values are cached in the injector where their provider is registered (unless contextual data is passed via `@inject`).
- When diagnosing hierarchy errors, use the error's `Resolution path` to identify the injector level where each token was found or missing.

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
  Logger,
  { token: LOGGER_CONFIG, useValue: { level: 'info' } },
  { token: Logger, useClass: ConsoleLogger },
  { token: 'alias-to-config', useToken: LOGGER_CONFIG },
];
```

Use the shorthand `SomeService` only when it is equivalent to `{ token: SomeService, useClass: SomeService }`.

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

Ditsmod application injectors are conceptually layered as:

```ts
providersPerApp -> providersPerMod -> providersPerRou -> providersPerReq

```

Apply this rule when placing providers:
If provider `A` depends on provider `B`, `B` must be registered in the same injector as `A` or in an ancestor injector. Do not put `B` only in a child injector.

Important consequences:

- Child injectors can ask parent injectors for missing tokens.
- Parent injectors never ask child injectors for tokens.
- If a child delegates `get(Service)` to a parent, the parent creates `Service` using dependencies visible from the parent level, not from the child level.
- Register a service in the child injector when it must use child-level overrides.
- Name injectors (e.g., `Injector.resolveAndCreate(providers, 'injectorName')`) in low-level tests or examples when hierarchy diagnostics matter.

## `get()` Versus `pull()`

Use `injector.get(Token)` for normal resolution and caching.

Use `child.pull(Token)` only for the specific case where the child lacks a provider that exists in an ancestor, but the pulled provider should be created in the child context so it can use child-level dependencies.

- **Crucial Caching Behavior:** When a provider is pulled from an ancestor, `pull()` returns a **new instance each time** instead of caching it. If the provider already exists in the current child injector, `pull()` acts identically to `get()` and utilizes the cache.

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

Use `Context` when data must be set after injector creation and read later at the same or lower injector level. Use `getSymbol<T>()` for typed context keys.

For class method parameters that read from `Context`, include `ctxProviders` unless the Ditsmod module already imports/re-exports the providers, such as through the `@ditsmod/rest`.

```ts
import { Context, Injector, ctx, ctxProviders, getSymbol } from '@ditsmod/core';

interface RequestState {
  userId: string;
}

const REQUEST_STATE = getSymbol<RequestState>('REQUEST_STATE');

class Handler {
  handle(@ctx(REQUEST_STATE) state: RequestState) {
    return state.userId;
  }
}

const injector = Injector.resolveAndCreate([
  ...ctxProviders,
  { token: 'user-id', useFactory: [Handler, Handler.prototype.handle] },
]);

injector.get(Context).set(REQUEST_STATE, { userId: '42' });
```

## Special Token: ParentParams (Inheritance)

When an `@injectable()` class extends a parent class, Ditsmod provides the `ParentParams` token to handle parent constructor dependencies cleanly without manual parameter mapping. DI automatically injects an array containing the parent's constructor arguments into this token.

To handle TypeScript type checking safely without suppression comments, use one of these two recommended alternatives in the child class constructor:

### Alternative 1 (Using `@inject` decorator)

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

### Alternative 2 (Using inline type assertion)

```ts
import { ParentParams, injectable } from '@ditsmod/core/di';

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
2. Find the provider level for that token.
3. For every constructor or factory dependency, verify that its provider is at the same level or an ancestor of the provider that needs it.
4. Check whether a child-level override is ineffective because the requested service is actually created in a parent injector.
5. Check for invalid runtime tokens caused by interfaces, type aliases, enums used only as types, or `import type`.
6. Ensure that array types are not being passed directly as runtime tokens; check for `InjectionToken` usage.
7. Verify that any `TokenProvider` (using `useToken`) eventually terminates in a non-token provider mapping.
8. Check for accidental mixing of regular and multi-providers.
